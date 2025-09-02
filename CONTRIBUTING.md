# Contributing

## Cara kerja harian (KP)
- Buat branch per tugas: `feat/...`, `fix/...`, `docs/...`.
- Commit kecil dan sering, gunakan pesan jelas:
  - `feat(pallet): add ContributionSubmitted event`
  - `chore(ci): add Rust build workflow`
  - `docs(materi-4): add screenshots extrinsic demo`

## Pull Request
- Deskripsikan tujuan singkat, langkah uji, dan bukti (screenshot/log).
- Jika menyentuh pallet/runtime, pastikan build `--release` sukses.

## Standar
- Rust: `cargo fmt` dan `cargo clippy` bersih.
- Dok: gunakan Markdown, simpan di `docs/`.
