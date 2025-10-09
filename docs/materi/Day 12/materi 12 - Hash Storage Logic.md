# Day 12 — Hash Storage Logic
**Topic:** Hash Storage Logic  
**Objective:** Implement hash + timestamp + origin logic  
**Task:** Implement storage + callable extrinsic  

---

## Ringkasan
Hari ini kamu akan **meng-implement** pallet `audit-log` dari desain Day 11: menyimpan **hash kontribusi model** berikut **origin (who)**, **block number**, dan **timestamp** (opsional) melalui **extrinsic** yang bisa dipanggil klien. Kita juga siapkan **event**, **error**, **WeightInfo placeholder**, dan **instruksi uji** (Polkadot.js + Subxt).

Referensi:
- Transaksi & submission: <https://docs.substrate.io/build/tx-logic/tx-submission/>
- Video pengantar: <https://www.youtube.com/watch?v=YUb5X3UtrY4>

---

## 1) Struktur Pallet (FRAME V2)
```
pallets/pallet-audit/
├─ Cargo.toml
└─ src
   ├─ lib.rs
   ├─ weights.rs         # optional (placeholder OK)
   ├─ mock.rs            # untuk unit-test
   └─ tests.rs
```

---

## 2) `Cargo.toml` (contoh)
> Samakan versi dengan workspace-mu.

```toml
[package]
name = "pallet-audit-log"
version = "0.1.0"
edition = "2021"
license = "MIT-0"

[dependencies]
scale-info = { version = "2", default-features = false, features = ["derive"] }
codec = { package = "parity-scale-codec", version = "3", default-features = false, features=["derive"] }
frame-support = { version = "*", default-features = false }
frame-system  = { version = "*", default-features = false }
sp-std        = { version = "*", default-features = false }

[dev-dependencies]
sp-core    = "*"
sp-io      = "*"
sp-runtime = "*"
frame-benchmarking = { version = "*", default-features = false, optional = true }

[features]
default = ["std"]
std = [
  "scale-info/std",
  "codec/std",
  "frame-support/std",
  "frame-system/std",
  "sp-std/std",
]
runtime-benchmarks = ["frame-benchmarking"]
```

---

## 3) Implementasi `lib.rs`
> Implementasi **Opsi A (first-seen immutable)** + indeks `ByAccount` dan `Recent` opsional.

```rust
#![cfg_attr(not(feature = "std"), no_std)]

pub use pallet::*;

#[frame_support::pallet]
pub mod pallet {
    use frame_support::{
        pallet_prelude::*,
        traits::Get,
        BoundedVec,
    };
    use frame_system::pallet_prelude::*;

    #[derive(
        Encode, Decode, scale_info::TypeInfo, MaxEncodedLen, Clone, PartialEq, Eq, RuntimeDebug
    )]
    pub struct LogEntry<AccountId, BlockNumber, Moment> {
        pub who: AccountId,
        pub block: BlockNumber,
        pub time: Option<Moment>,
    }

    #[pallet::pallet]
    pub struct Pallet<T>(_);

    #[pallet::config]
    pub trait Config: frame_system::Config {
        type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;

        type Moment: Parameter + Default + MaxEncodedLen;

        #[pallet::constant] type MaxPerAccount: Get<u32>;
        #[pallet::constant] type MaxRecent: Get<u32>;

        type WeightInfo: WeightInfo;
    }

    #[pallet::storage]
    pub type HashLog<T: Config> = StorageMap<
        _, Blake2_128Concat, T::Hash,
        LogEntry<T::AccountId, T::BlockNumber, T::Moment>,
        OptionQuery
    >;

    #[pallet::storage]
    #[pallet::getter(fn by_account)]
    pub type ByAccount<T: Config> = StorageMap<
        _, Blake2_128Concat, T::AccountId,
        BoundedVec<T::Hash, T::MaxPerAccount>,
        ValueQuery
    >;

    #[pallet::storage]
    pub type Recent<T: Config> = StorageValue<
        _, BoundedVec<T::Hash, T::MaxRecent>,
        ValueQuery
    >;

    #[pallet::event]
    #[pallet::generate_deposit(pub(super) fn deposit_event)]
    pub enum Event<T: Config> {
        HashSubmitted {
            who: T::AccountId,
            hash: T::Hash,
            block: T::BlockNumber,
            time: Option<T::Moment>,
        },
    }

    #[pallet::error]
    pub enum Error<T> {
        AlreadyExists,
        AccountIndexFull,
        RecentFull,
    }

    #[pallet::call]
    impl<T: Config> Pallet<T> {
        #[pallet::call_index(0)]
        #[pallet::weight(T::WeightInfo::submit_hash())]
        pub fn submit_hash(origin: OriginFor<T>, hash: T::Hash) -> DispatchResult {
            let who = ensure_signed(origin)?;
            ensure!(!HashLog::<T>::contains_key(&hash), Error::<T>::AlreadyExists);

            let block = <frame_system::Pallet<T>>::block_number();
            let now_opt: Option<T::Moment> = None;

            let entry = LogEntry::<T::AccountId, T::BlockNumber, T::Moment> {
                who: who.clone(),
                block,
                time: now_opt,
            };

            HashLog::<T>::insert(&hash, &entry);

            ByAccount::<T>::try_mutate(&who, |vec| -> DispatchResult {
                if vec.len() as u32 >= T::MaxPerAccount::get() {
                    return Err(Error::<T>::AccountIndexFull.into());
                }
                vec.try_push(hash).map_err(|_| Error::<T>::AccountIndexFull)?;
                Ok(())
            })?;

            Recent::<T>::mutate(|vec| {
                if vec.len() as u32 == T::MaxRecent::get() {
                    let _ = vec.remove(0);
                }
                let _ = vec.try_push(hash);
            });

            Self::deposit_event(Event::HashSubmitted { who, hash, block, time: now_opt });
            Ok(())
        }
    }

    pub trait WeightInfo {
        fn submit_hash() -> Weight;
    }

    impl WeightInfo for () {
        fn submit_hash() -> Weight { Weight::from_parts(10_000, 0) }
    }
}
```

> **Aktifkan timestamp**: di runtime pakai `pallet_timestamp` dan di call ganti `let now_opt = Some(pallet_timestamp::Pallet::<T>::get());`

---

## 4) Integrasi Runtime
```rust
parameter_types! {
    pub const MaxPerAccount: u32 = 64;
    pub const MaxRecent: u32 = 256;
}

impl pallet_audit_log::Config for Runtime {
    type RuntimeEvent = RuntimeEvent;
    type Moment = u64;
    type MaxPerAccount = MaxPerAccount;
    type MaxRecent = MaxRecent;
    type WeightInfo = ();
}
```

---

## 5) Uji Cepat (Polkadot.js)
- **Extrinsics** → `auditLog.submitHash(hash)` → pakai `0x` + 64 hex.
- **Events** → lihat `auditLog.HashSubmitted`.
- **Chain state** → `HashLog(hash)` → `Some(LogEntry)`, `ByAccount(sender)` mengandung hash, `Recent()` berisi hash.

---

## 6) Subxt (opsional)
Script contoh submit + query ada di Day 11. Tinggal ganti modul jadi `audit_log()` dan storage `hash_log()`.

---

## 7) Unit Test
Tulis minimal:
- `submit_hash` sukses mengisi 3 storage & event.
- Duplikasi ditolak (`AlreadyExists`).
- Penuh indeks akun (`AccountIndexFull`) jika `MaxPerAccount` kecil pada mock.

---

## 8) README Ringkas
```markdown
# pallet-audit-log (Day 12)
Implementasi extrinsic `submit_hash(hash)` yang menyimpan `who`, `block`, dan timestamp (opsional),
dengan indeks `ByAccount` & `Recent` (opsional). Event: `HashSubmitted`.


```

---

## 9) Checklist Day 12
- [ ] `lib.rs`: storage, event/error, `submit_hash`, WeightInfo placeholder.
- [ ] Integrasi runtime & build OK.
- [ ] Smoke test via UI.
- [ ] Unit test lulus.
- [ ] (Opsional) Subxt script.
