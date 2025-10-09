# Day 11 — Audit Log Pallet Design
**Topic:** Audit Log Pallet Design  
**Objective:** Design a pallet to store model contribution hashes  
**Task:** Design schema for audit hash log (hash, time, origin)  

---

## Ringkasan
Merancang **pallet audit log** yang menyimpan _model contribution hashes_ secara **immutable** dengan jejak waktu dan asal transaksi. 

Fokus :
- **Skema storage** yang simpel, hemat biaya, dan mudah di-query.
- **Event-first logging** (log lengkap via event) + **storage indeks** untuk verifikasi cepat.
- Antisipasi duplikasi, bloat, dan kebutuhan _query_ harian (per subjek, per blok, terbaru).

Referensi utama:
- Substrate Storage guide: <https://docs.substrate.io/reference/how-to-guides/pallets/define-storage/>
- Video pengantar (konsep): <https://www.youtube.com/watch?v=-ZL0_6Rzvz8>

---

## 1) Persyaratan Fungsional
- Dapat menyimpan **hash kontribusi model** (`[u8;32]` atau `T::Hash`).
- Mencatat **waktu/urutan** (minimal **block number**; opsional **timestamp** jika pallet `timestamp` dipasang).
- Mencatat **origin** (`who: AccountId`) pengirim.
- Cegah **duplikasi** (opsi kebijakan: _first-wins_ atau _multi-submit_).
- Mudah diverifikasi dari klien: query by `hash`, by `who`, atau tarik **latest** untuk audit.

---

## 2) Desain Skema — Opsi & Trade-offs

### **First-seen immutable** (paling sederhana, rekomendasi default)
- **Kebijakan:** Satu `hash` hanya boleh tercatat sekali (pertama yang masuk menang).  
- **Storage Utama:**
  - `HashLog: map T::Hash => LogEntry { who, block_number, timestamp? }`
- **Indeks bantu (opsional):**
  - `ByAccount: map AccountId => BoundedVec<T::Hash, MaxPerAccount>` (riwayat hash yang pernah ia submit).
  - `Recent: BoundedVec<T::Hash, MaxRecent>` (ring buffer hash terbaru untuk UI).
- **Kelebihan:** storage minimal, query by hash cepat, anti-duplicated.  
- **Kekurangan:** kalau butuh _multi-submit_ hash yang sama (mis. verifikasi ulang) tidak muat.

---

## 3) Tipe Data & Time Source

### Tipe Log
```rust
#[derive(Encode, Decode, TypeInfo, MaxEncodedLen, Clone, PartialEq, Eq, RuntimeDebug)]
pub struct LogEntry<AccountId, BlockNumber, Moment> {
    pub who: AccountId,
    pub block: BlockNumber,
    pub time: Option<Moment>, 
}
```

### Time Source
- **Minimal**: `block = frame_system::Pallet<T>::block_number()` → selalu ada.
- **Opsional**: `time = pallet_timestamp::Pallet::<T>::get()` → butuh integrasi pallet `timestamp` di runtime.
- Simpan `time` sebagai `Option<Moment>` untuk tetap _no_std_ dan tetap kompatibel bila `timestamp` tidak dipasang.

---

## 4) Konfigurasi Pallet (FRAME V2)
```rust
#[pallet::config]
pub trait Config: frame_system::Config {
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;

    /// Alias waktu; gunakan `pallet_timestamp::Moment` di runtime jika diaktifkan.
    type Moment: Parameter + Default + MaxEncodedLen + MaybeSerializeDeserialize;

    /// Batas ukuran indeks
    #[pallet::constant] type MaxPerAccount: Get<u32>;
    #[pallet::constant] type MaxRecent: Get<u32>;

    /// WeightInfo
    type WeightInfo: WeightInfo;
}
```

> **Catatan:** Kalau kamu **pasti** pakai pallet_timestamp, kamu bisa ganti `type Moment = pallet_timestamp::Moment;` di runtime dan isi `time: Some(now)` pada extrinsic.

---

## 5) Storage Layout (Opsi A — First-seen Immutable)
```rust
#[pallet::storage]
pub type HashLog<T: Config> = StorageMap<
    _,
    Blake2_128Concat,
    T::Hash, // 32-byte hash
    LogEntry<T::AccountId, T::BlockNumber, T::Moment>,
    OptionQuery
>;

#[pallet::storage]
#[pallet::getter(fn by_account)]
pub type ByAccount<T: Config> = StorageMap<
    _,
    Blake2_128Concat,
    T::AccountId,
    BoundedVec<T::Hash, T::MaxPerAccount>,
    ValueQuery
>;

#[pallet::storage]
pub type Recent<T: Config> = StorageValue<
    _,
    BoundedVec<T::Hash, T::MaxRecent>,
    ValueQuery
>;
```

> **Alasan:**  
> - Kunci utama `hash` → lookup cepat saat verifikasi.  
> - Dua indeks bantu *opsional* untuk keperluan UI / audit harian.  
> - Semua nilai bertipe bounded → proteksi bloat dan pastikan deterministik.

---

## 6) Events & Errors
```rust
#[pallet::event]
#[pallet::generate_deposit(pub(super) fn deposit_event)]
pub enum Event<T: Config> {
    /// Hash baru dicatat pertama kali
    HashSubmitted { who: T::AccountId, hash: T::Hash, block: T::BlockNumber, time: Option<T::Moment> },
}

#[pallet::error]
pub enum Error<T> {
    AlreadyExists,     // hash sudah tercatat
    AccountIndexFull,  // ByAccount penuh
    RecentFull,        // Recent penuh
}
```

---

## 7) Extrinsic Minimal
```rust
#[pallet::call]
impl<T: Config> Pallet<T> {
    /// Catat hash kontribusi model (first seen wins)
    #[pallet::call_index(0)]
    #[pallet::weight(T::WeightInfo::submit_hash())]
    pub fn submit_hash(origin: OriginFor<T>, hash: T::Hash) -> DispatchResult {
        let who = ensure_signed(origin)?;
        ensure!(!HashLog::<T>::contains_key(&hash), Error::<T>::AlreadyExists);

        // waktu
        let now_opt = {
            // jika runtime memasang pallet_timestamp, panggil di sini; kalau tidak, biarkan None
            None::<T::Moment>
        };

        let entry = LogEntry::<T::AccountId, T::BlockNumber, T::Moment> {
            who: who.clone(),
            block: <frame_system::Pallet<T>>::block_number(),
            time: now_opt,
        };

        HashLog::<T>::insert(&hash, &entry);

        // update ByAccount (opsional)
        ByAccount::<T>::try_mutate(&who, |vec| -> DispatchResult {
            if vec.len() as u32 >= T::MaxPerAccount::get() {
                return Err(Error::<T>::AccountIndexFull.into());
            }
            vec.try_push(hash).map_err(|_| Error::<T>::AccountIndexFull)?;
            Ok(())
        })?;

        // update Recent (opsional) — ring buffer sederhana
        Recent::<T>::mutate(|vec| {
            if vec.len() as u32 == T::MaxRecent::get() {
                let _ = vec.remove(0);
            }
            let _ = vec.try_push(hash);
        });

        Self::deposit_event(Event::HashSubmitted { who, hash, block: <frame_system::Pallet<T>>::block_number(), time: now_opt });
        Ok(())
    }
}
```

---

## 8) WeightInfo & Benchmarking (singkat)
```rust
pub trait WeightInfo {
    fn submit_hash() -> Weight;
}

impl WeightInfo for () {
    fn submit_hash() -> Weight { Weight::from_parts(10_000, 0) } // placeholder untuk dev/test
}
```
- Setelah stabil, **generate** `weights.rs` via benchmarking dan map di runtime:
  ```rust
  type WeightInfo = pallet_audit_log::weights::SubstrateWeight<Runtime>;
  ```

---

## 9) Integrasi Runtime (contoh)
**`runtime/src/lib.rs`**
```rust
parameter_types! {
    pub const MaxPerAccount: u32 = 64;
    pub const MaxRecent: u32 = 256;
}

impl pallet_audit_log::Config for Runtime {
    type RuntimeEvent = RuntimeEvent;
    type Moment = u64; // atau pallet_timestamp::Moment jika kamu pasang pallet_timestamp
    type MaxPerAccount = MaxPerAccount;
    type MaxRecent = MaxRecent;
    type WeightInfo = ();
}
```

**`construct_runtime!`**
```rust
PalletAuditLog: pallet_audit_log,
```

---

## 10) Query Pattern (untuk UI & verifikasi)
- **Verifikasi hash** (utama):
  - `HashLog(hash)` → `Option<LogEntry>`
- **Riwayat per akun** (opsional):
  - `ByAccount(who)` → `Vec<Hash>`; untuk tiap `hash` lakukan `HashLog(hash)` bila perlu detail.
- **Terbaru** (opsional & ringan):
  - `Recent()` → `Vec<Hash>`; tampilkan di dashboard dan lanjutkan lookup detail per hash.

---

## 11) Kebijakan Duplikasi & Keamanan
- **First-seen wins** mencegah spam duplikasi di kunci `hash`.  
- _Front‑running_ tidak kritikal karena kita hanya log hash; yang penting kamu punya file asli untuk membuktikan kepemilikan.  
- Bila kamu ingin “pemilik pertama saja yang sah”, cukup **ikat** semantik: hanya _first-seen_ yang dianggap “canonical”; submit selanjutnya (walau berbeda akun) ditolak.

---

## 12) Uji Minimal (unit test)
- `submit_hash` sukses → `HashLog` terisi, `ByAccount`/`Recent` ter-update, event keluar.
- `submit_hash` duplikat → `AlreadyExists`.
- `ByAccount` penuh → `AccountIndexFull`.
- (Jika timestamp diaktifkan) `time.is_some()`.

Contoh pendek:
```rust
#[test]
fn submit_hash_works() {
    new_test_ext().execute_with(|| {
        let h = H256::repeat_byte(0x11);
        assert_ok!(AuditLog::submit_hash(Origin::signed(ALICE), h));
        let e = HashLog::<Test>::get(h).expect("exists");
        assert_eq!(e.who, ALICE);
        System::assert_has_event(Event::HashSubmitted{ who: ALICE, hash: h, block: 1u64.into(), time: None }.into());
    });
}
```

---

## Integrasi
```rust
impl pallet_audit_log::Config for Runtime {
  type RuntimeEvent = RuntimeEvent;
  type Moment = u64;
  type MaxPerAccount = MaxPerAccount;
  type MaxRecent = MaxRecent;
  type WeightInfo = ();
}
```

---

