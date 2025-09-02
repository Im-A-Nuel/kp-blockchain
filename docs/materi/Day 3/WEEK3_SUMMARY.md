# Week 3 — Local Node Setup

## Tujuan
- Menyiapkan environment Substrate di WSL (Ubuntu).
- Install Rust dan dependensi.
- Build dan run parachain template node.

## Langkah Utama
1. Install WSL: `wsl --install`
2. Update Ubuntu: `sudo apt update`
3. Install dependensi:  
   `sudo apt install --assume-yes git clang curl libssl-dev llvm libudev-dev make protobuf-compiler`
4. Install Rust:  
   `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`  
   lalu `source ~/.cargo/env`
5. Tambahkan target & komponen:  
```
rustup default stable
rustup update
rustup target add wasm32-unknown-unknown
rustup component add rust-src

```

6. Clone template parachain:  
`git clone -b v0.0.4 https://github.com/paritytech/polkadot-sdk-parachain-template.git parachain-template`
7. Build node:  
```
cd parachain-template
cargo build -p parachain-template-node --release
```

8. Run node dev:  
`./target/release/parachain-template-node --dev --tmp`

## Verifikasi
- Akses Polkadot.js UI di:  
[http://localhost:9944](https://polkadot.js.org/apps/?rpc=ws%3A%2F%2F127.0.0.1%3A9944#/accounts)  
- Pastikan terhubung ke `ws://127.0.0.1:9944`.

## Bukti
- Screenshot terminal node jalan (`evidence/week3/node_running.png`).
- Screenshot Polkadot.js UI terhubung (`evidence/week3/polkadotjs_connected.png`).
