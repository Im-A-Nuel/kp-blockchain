# Day 10 — Review Pallet Module
**Topik:** Review Pallet Module  
**Objective:** Validate & clean custom pallet module  
**Task:** Clean and finalize pallet structure  

---

## 0) Ringkasan
Hari ini kamu akan *audit* ulang pallet kustom (mis. `pallet-audit`) agar rapi, valid untuk produksi/dev, dan enak dipelihara. Target: struktur file bersih, makro FRAME V2 benar, weight berbasis benchmarking (tidak hard‑code), build/test mulus, dan dokumentasi siap tempel.

---

## 1) Definition
Pallet dianggap **bersih & siap** jika:

- Struktur file rapi: `lib.rs`, `weights.rs`, `benchmarking.rs` (opsional: hanya saat fitur `runtime-benchmarks`), `mock.rs`, `tests.rs`, `Cargo.toml`, `README.md`.
- Semua makro FRAME V2 digunakan dengan benar:  
  `#[frame_support::pallet]`, `#[pallet::pallet]`, `#[pallet::config]`, `#[pallet::storage]`, `#[pallet::event]`, `#[pallet::error]`, `#[pallet::call]`.
- **Tidak ada deprecated constant weight.** Semua extrinsic memakai `T::WeightInfo::*` yang berasal dari hasil benchmarking (`weights.rs`).
- **Lint & build lulus:**
  ```bash
  cargo fmt --all
  cargo clippy --all-targets --all-features -- -D warnings
  cargo build --release
  ```
- **Unit test minimal ada & lulus:**
  ```bash
  cargo test -p pallet-audit
  ```

---

## 2) Struktur Folder yang Direkomendasikan
```
pallets/pallet-audit/
├─ Cargo.toml
├─ README.md
└─ src
   ├─ lib.rs
   ├─ weights.rs
   ├─ benchmarking.rs         # only if --features runtime-benchmarks
   ├─ mock.rs                 # test runtime
   └─ tests.rs                # unit tests
```

---

## 3) Template Minimal `lib.rs` (FRAME V2 + WeightInfo siap)

```rust
#![cfg_attr(not(feature = "std"), no_std)]

pub use pallet::*;

#[frame_support::pallet]
pub mod pallet {
    use frame_support::{
        dispatch::DispatchResult,
        pallet_prelude::*,
        traits::Get,
        BoundedVec,
    };
    use frame_system::pallet_prelude::*;
    use scale_info::TypeInfo;

    #[pallet::pallet]
    pub struct Pallet<T>(_);

    /// Konfigurasi pallet
    #[pallet::config]
    pub trait Config: frame_system::Config {
        /// Event runtime
        type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;

        /// Batas panjang data note/metadata
        #[pallet::constant]
        type MaxNoteLen: Get<u32>;

        /// WeightInfo hasil benchmarking
        type WeightInfo: WeightInfo;
    }

    /// Storage: menyimpan catatan audit berbasis hash
    #[pallet::storage]
    pub type Records<T: Config> = StorageMap<
        _,
        Blake2_128Concat,
        T::Hash,                                      // key: hash data
        (T::AccountId, BoundedVec<u8, T::MaxNoteLen>),// value: (owner, note)
        OptionQuery
    >;

    #[pallet::event]
    #[pallet::generate_deposit(pub(super) fn deposit_event)]
    pub enum Event<T: Config> {
        RecordCreated { who: T::AccountId, hash: T::Hash },
        NoteUpdated   { who: T::AccountId, hash: T::Hash },
    }

    #[pallet::error]
    pub enum Error<T> {
        AlreadyExists,
        NotFound,
        NotOwner,
        NoteTooLong,
    }

    /// Extrinsics
    #[pallet::call]
    impl<T: Config> Pallet<T> {
        /// Submit hash bukti (audit trail), opsional ada catatan singkat
        #[pallet::call_index(0)]
        #[pallet::weight(T::WeightInfo::create_record(note.len() as u32))]
        pub fn create_record(
            origin: OriginFor<T>,
            hash: T::Hash,
            note: Vec<u8>,
        ) -> DispatchResult {
            let who = ensure_signed(origin)?;
            let note_bv: BoundedVec<_, T::MaxNoteLen> =
                note.clone().try_into().map_err(|_| Error::<T>::NoteTooLong)?;
            ensure!(!Records::<T>::contains_key(&hash), Error::<T>::AlreadyExists);

            Records::<T>::insert(&hash, (who.clone(), note_bv));
            Self::deposit_event(Event::RecordCreated { who, hash });
            Ok(())
        }

        /// Update note (hanya pemilik)
        #[pallet::call_index(1)]
        #[pallet::weight(T::WeightInfo::update_note(new_note.len() as u32))]
        pub fn update_note(
            origin: OriginFor<T>,
            hash: T::Hash,
            new_note: Vec<u8>,
        ) -> DispatchResult {
            let who = ensure_signed(origin)?;
            Records::<T>::try_mutate(&hash, |rec| -> DispatchResult {
                let (owner, note_ref) = rec.as_mut().ok_or(Error::<T>::NotFound)?;
                ensure!(&who == owner, Error::<T>::NotOwner);
                let note_bv: BoundedVec<_, T::MaxNoteLen> =
                    new_note.clone().try_into().map_err(|_| Error::<T>::NoteTooLong)?;
                *note_ref = note_bv;
                Ok(())
            })?;

            Self::deposit_event(Event::NoteUpdated { who, hash });
            Ok(())
        }
    }

    // ------- WEIGHTS TRAIT -------
    pub trait WeightInfo {
        fn create_record(n: u32) -> Weight;
        fn update_note(n: u32) -> Weight;
    }

    // Fallback untuk tests (jangan dipakai di produksi)
    impl WeightInfo for () {
        fn create_record(_n: u32) -> Weight { Weight::from_parts(10_000, 0) }
        fn update_note(_n: u32) -> Weight { Weight::from_parts(10_000, 0) }
    }
}
```

**Struktur**
- Makro FRAME V2 memastikan pallet kompatibel dengan `construct_runtime!` dan pattern terbaru.
- *hard‑coded constant weight* → `T::WeightInfo::*` dan hasilkan `weights.rs` via benchmarking.
- `BoundedVec` dengan `MaxNoteLen` membatasi ukuran data agar aman & deterministik untuk SCALE.

---

## 4) `weights.rs` (Hasil *Benchmarking*)
File ini **di‑generate** dari proses benchmarking. Setelah `weights.rs` ada, gunakan implementasi `SubstrateWeight<T>` alih‑alih `()`:

```rust
// src/weights.rs (generated)
use frame_support::weights::Weight;

pub struct SubstrateWeight<T>(core::marker::PhantomData<T>);
impl<T: frame_system::Config> pallet::WeightInfo for SubstrateWeight<T> {
    fn create_record(_n: u32) -> Weight { /* auto-generated */ }
    fn update_note(_n: u32) -> Weight { /* auto-generated */ }
}
```

Di `lib.rs` tambahkan modul:
```rust
#[cfg(feature = "runtime-benchmarks")]
mod benchmarking;
pub mod weights;
```

Di runtime:
```rust
type WeightInfo = pallet_audit::weights::SubstrateWeight<Runtime>;
```

**Cara generate `weights.rs`:**
```bash
# 1) Build node dengan fitur benchmarking
cargo build --release --features runtime-benchmarks

# 2) Jalankan benchmark (sesuaikan <node-bin> dan path pallet)
./target/release/<node-bin> benchmark pallet   --chain=dev   --pallet=pallet-audit   --extrinsic='*'   --steps=50 --repeat=20   --output=pallets/pallet-audit/src/weights.rs
```

> Catatan: nama binary (`<node-bin>`) mengikuti project kamu (mis. `parachain-template-node` / `solochain-template-node` / lainnya).

---

## 5) `benchmarking.rs` (Opsional)
Skeleton minimal (agar perintah benchmark di atas bisa jalan):

```rust
#![cfg(feature = "runtime-benchmarks")]

use super::*;
use frame_benchmarking::{benchmarks, whitelisted_caller, impl_benchmark_test_suite};
use frame_system::RawOrigin;

benchmarks! {
    create_record {
        let caller: T::AccountId = whitelisted_caller();
        let hash = T::Hashing::hash(b"hello");
        let note = vec![0u8; 16];
    }: _(RawOrigin::Signed(caller), hash, note)

    update_note {
        let caller: T::AccountId = whitelisted_caller();
        let hash = T::Hashing::hash(b"hello");
        let note = vec![0u8; 4];
        Pallet::<T>::create_record(RawOrigin::Signed(caller.clone()).into(), hash, note).unwrap();
        let new_note = vec![1u8; 32];
    }: _(RawOrigin::Signed(caller), hash, new_note)
}

impl_benchmark_test_suite!(
    Pallet,
    crate::mock::new_test_ext(),
    crate::mock::Test
);
```

---

## 6) `mock.rs` & `tests.rs`
- **`mock.rs`**: buat runtime uji sederhana (define `Test` runtime, `new_test_ext()` untuk inisialisasi storage).  
- **`tests.rs`**: tulis *unit test* untuk **happy-path** dan **error-path** minimal:
  - `create_record` sukses.
  - `update_note` oleh non‑pemilik → gagal `NotOwner`.

Contoh *assert error* umum:
```rust
assert_noop!(
    Pallet::<T>::update_note(origin, hash, note),
    Error::<T>::NotOwner
);
```

---

## 7) `Cargo.toml` Pallet (Contoh)
> Samakan versi dengan workspace Polkadot SDK agar tidak konflik.

```toml
[package]
name = "pallet-audit"
version = "0.1.0"
edition = "2021"
license = "MIT-0"

[dependencies]
scale-info = { version = "2", default-features = false, features = ["derive"] }
codec = { package = "parity-scale-codec", version = "3", default-features = false, features=["derive"] }
frame-support = { version = "*", default-features = false }
frame-system  = { version = "*", default-features = false }
sp-std        = { version = "*", default-features = false }

# Benchmarking (opsional)
frame-benchmarking = { version="*", optional = true, default-features = false }
frame-system-benchmarking = { version="*", optional = true, default-features = false }

[dev-dependencies]
sp-core = "*"
sp-io   = "*"
sp-runtime = "*"

[features]
default = ["std"]
std = [
  "scale-info/std",
  "codec/std",
  "frame-support/std",
  "frame-system/std",
  "sp-std/std",
]
runtime-benchmarks = [
  "frame-benchmarking",
  "frame-system-benchmarking",
]
try-runtime = [] 
```

---

## 8) Integrasi di Runtime
Di `runtime/Cargo.toml`, tambahkan dependensi `pallet-audit`.  
Di `runtime/src/lib.rs`:

```rust
impl pallet_audit::Config for Runtime {
    type RuntimeEvent = RuntimeEvent;
    type MaxNoteLen = ConstU32<128>;
    type WeightInfo = pallet_audit::weights::SubstrateWeight<Runtime>;
}

// construct_runtime! {
//   // ...
//   PalletAudit: pallet_audit,
//   // ...
// }
```

---

## 9) Quick‑Fix Error Umum
1) **Deprecated constant weight**
   - Hindari `#[pallet::weight(10_000)]`. Ganti ke `T::WeightInfo::function(args…)` dan hasilkan `weights.rs` via benchmarking.

2) **`the trait bound T: scale_info::TypeInfo is not satisfied`**
   - Biasanya karena menyimpan tipe generik `T` langsung. Solusi:
     - Simpan tipe konkret runtime (`T::AccountId`, `T::Hash`, dst.) yang sudah mendukung `TypeInfo`.
     - Untuk struct kustom yang masuk storage/event, gunakan derivasi:
       ```rust
       #[derive(Encode, Decode, TypeInfo, MaxEncodedLen)]
       #[scale_info(skip_type_params(T))]
       pub struct MyThing<T: Config> {
           pub owner: T::AccountId,
       }
       ```
   - Pastikan semua parameter event mengimplementasikan `TypeInfo` (umumnya aman jika pakai tipe standar runtime).

---

## 10) Migrasi & Upgrade (Jika Storage Berubah)
- Naikkan `spec_version` pada runtime.
- Implement `OnRuntimeUpgrade` untuk migrasi storage.
- Uji dengan `try-runtime` (build dengan fitur `try-runtime`).

---

## 11) Test
```markdown
# pallet-audit

## Build & Test
```bash
cargo fmt --all
cargo clippy -- -D warnings
cargo test -p pallet-audit
```

## Benchmark & Generate Weights
```bash
cargo build --release --features runtime-benchmarks
./target/release/<node> benchmark pallet   --chain=dev   --pallet=pallet-audit   --extrinsic='*'   --steps=50 --repeat=20   --output=pallets/pallet-audit/src/weights.rs
```

## Integrasi Runtime
```rust
impl pallet_audit::Config for Runtime {
    type RuntimeEvent = RuntimeEvent;
    type MaxNoteLen = ConstU32<128>;
    type WeightInfo = pallet_audit::weights::SubstrateWeight<Runtime>;
}
```


---

## 12) Referensi
- FRAME runtime macros (pallet): <https://docs.substrate.io/build/runtime-macros/#pallet>  
- Video pendukung (overview konsep pallet): <https://www.youtube.com/watch?v=TApJY1bT2kA>

---

## 13) Checklist Eksekusi Cepat
- [ ] Rapikan `lib.rs` sesuai template & ganti weight ke `T::WeightInfo`.
- [ ] Tambah `weights.rs` (generate via benchmarking).
- [ ] Pastikan di runtime: `type WeightInfo = ...SubstrateWeight<Runtime>`.
- [ ] Tambah/rapikan `mock.rs` & `tests.rs` (min. 2 test: create OK, update non‑owner → error).
- [ ] Jalankan `fmt`, `clippy`, `build`.
- [ ] (Jika pernah ubah storage) tulis migrasi + uji `try-runtime`.
- [ ] Update `README.md`.
