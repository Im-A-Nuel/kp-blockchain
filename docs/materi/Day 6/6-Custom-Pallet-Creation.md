
# Day 6 — Custom Pallet Creation
**Topic:** Custom Pallet Creation  
**Objective:** Learn how to create a custom pallet in Substrate  
**Task:** Fork Substrate repo, add new pallet crate

---

## 1) Overview
Pada hari ini, tujuan praktik adalah **membuat pallet kustom** di dalam monorepo Substrate, hingga berhasil di-*build* tanpa error dan siap dipush ke GitHub.

**Hasil akhir yang diharapkan:**
- Fork `paritytech/substrate` sudah dibuat di akun GitHub.
- Branch kerja `feat/pallet-audit` berisi crate baru **`frame/pallet-audit`**.
- Build crate sukses: `cargo build -p pallet-audit --release`.

---

## 2) Prasyarat (ringkas)
- WSL + Ubuntu sudah siap (Day 3).
- Rust toolchain mengikuti `rust-toolchain.toml` dari Substrate.
- Target wasm telah ditambahkan:
  ```bash
  rustup target add wasm32-unknown-unknown
  ```

---

## 3) Langkah A — Fork & Clone Substrate
1. **Fork** repo `paritytech/substrate` ke akun GitHub pribadi.
2. **Clone** fork ke lokal dan tambahkan remote upstream:
   ```bash
   git clone https://github.com/Im-A-Nuel/substrate.git
   cd substrate
   git remote add upstream https://github.com/paritytech/substrate.git
   git fetch upstream
   git checkout -b feat/pallet-audit
   ```

> Tips: Tetap di branch feature (`feat/pallet-audit`) agar mudah dilacak dan di-review.

---

## 4) Langkah B — Tambah Pallet Baru (frame/pallet-audit)
1. Buat folder:
   ```bash
   mkdir -p frame/pallet-audit/src
   ```
2. Buat file **Cargo.toml** di `frame/pallet-audit/`:
   ```toml
   [package]
   name = "pallet-audit"
   version = "0.1.0"
   edition = "2021"
   license = "Apache-2.0"

   [dependencies]
   frame-support = { default-features = false, path = "../support" }
   frame-system  = { default-features = false, path = "../system" }

   parity-scale-codec = { version = "3", default-features = false, features = ["derive"] }
   scale-info         = { version = "2", default-features = false, features = ["derive"] }

   [features]
   default = ["std"]
   std = [
       "frame-support/std",
       "frame-system/std",
       "parity-scale-codec/std",
       "scale-info/std",
   ]
   try-runtime = []
   runtime-benchmarks = []
   ```

3. Buat file **src/lib.rs** (FRAME v2 minimal):
   ```rust
   #![cfg_attr(not(feature = "std"), no_std)]

   pub use pallet::*;

   #[frame_support::pallet]
   pub mod pallet {
       use frame_support::{ pallet_prelude::*, weights::Weight };
       use frame_system::pallet_prelude::*;

       #[pallet::pallet]
       pub struct Pallet<T>(_);

       #[pallet::config]
       pub trait Config: frame_system::Config {
           type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
           type WeightInfo: WeightInfo;
       }

       #[pallet::storage]
       #[pallet::getter(fn last_commit)]
       pub type LastCommit<T: Config> = StorageValue<_, BoundedVec<u8, ConstU32<256>>, OptionQuery>;

       #[pallet::event]
       #[pallet::generate_deposit(pub(super) fn deposit_event)]
       pub enum Event<T: Config> {
           Committed { who: T::AccountId, len: u32 },
       }

       #[pallet::error]
       pub enum Error<T> {
           Empty,
           TooLong,
       }

       #[pallet::call]
       impl<T: Config> Pallet<T> {
           /// Simpan komitmen kecil (mis. hash) ke storage
           #[pallet::call_index(0)]
           #[pallet::weight(T::WeightInfo::commit(0))]
           pub fn commit(origin: OriginFor<T>, data: Vec<u8>) -> DispatchResult {
               let who = ensure_signed(origin)?;
               ensure!(!data.is_empty(), Error::<T>::Empty);
               let bounded: BoundedVec<u8, ConstU32<256>> = data.try_into().map_err(|_| Error::<T>::TooLong)?;
               LastCommit::<T>::put(&bounded);
               Self::deposit_event(Event::Committed { who, len: bounded.len() as u32 });
               Ok(())
           }
       }

       pub trait WeightInfo {
           fn commit(_len: u32) -> Weight;
       }

       impl WeightInfo for () {
           fn commit(_len: u32) -> Weight {
               Weight::from_parts(10_000, 0)
           }
       }
   }
   ```

4. Daftarkan crate ke workspace:
   - Di `frame/Cargo.toml`, tambahkan `pallet-audit` pada `[workspace].members`.
   - Jika root workspace juga mengatur members eksplisit, tambahkan `"frame/pallet-audit"`.



---

## 5) Build Pallet
Jalankan dari root repo Substrate:
```bash
cargo build -p pallet-audit --release
```

**Sukses bila** build selesai tanpa error (warning dari dependency upstream boleh diabaikan untuk materi ini).

---

## 6) Verifikasi & Bukti
Lampirkan minimal 2 bukti berikut di repo (folder contoh: `evidence/day6/`):
- Screenshot hasil sukses `cargo build -p pallet-audit --release`.
- Cuplikan struktur folder `frame/pallet-audit/` (Cargo.toml & src/lib.rs terlihat).

File contoh:
```
evidence/day6/build_success.png
evidence/day6/folder_structure.png
```

---

## 7) Troubleshooting Singkat
- **E0207 unconstrained type parameter** pada `impl<T: Config> WeightInfo for ()`
  → Ganti jadi `impl WeightInfo for ()` (tanpa `T`).
- **`unexpected cfg` untuk `try-runtime`/`runtime-benchmarks`**
  → Tambahkan fitur kosong itu di `[features]` Cargo.toml pallet.
- **Toolchain mismatch**
  → Ikuti versi di `rust-toolchain.toml` repo Substrate.

---