# Day 8 — CLI Submit Extrinsic dengan Subxt

## Objective
Membuat **CLI sederhana** (Rust) untuk submit hash/data ke blockchain Substrate melalui WebSocket, menggunakan library **[Subxt](https://github.com/paritytech/subxt)**.

---

## Latar Belakang
- **Extrinsic** = input dari luar runtime (signed, unsigned, inherent).  
- Supaya bisa mengirim extrinsic dari luar chain, kita butuh **client library**.  
- **Subxt** adalah client resmi berbasis Rust yang memudahkan query storage dan submit extrinsic secara **programmatic**.  
- CLI ini akan mengirim extrinsic ke node lokal (`node-template`), misalnya ke call:
  - `audit.create_record(Vec<u8>)` → data disimpan on-chain.  
  - `audit.submit_hash(H256)` → hash disimpan on-chain.  
  - atau call umum: `system.remark(Vec<u8>)`.

---

## Persiapan

### 1. Jalankan node lokal
Gunakan **polkadot-sdk-solochain-template** atau **node-template**:

```bash
git clone https://github.com/paritytech/polkadot-sdk-solochain-template.git
cd polkadot-sdk-solochain-template
cargo build --release
./target/release/node-template --dev -l runtime=info
Node akan terbuka di ws://127.0.0.1:9944.
```

### 2. Buat proyek CLI
```bash
cargo new audit-cli
cd audit-cli
```

```bash
[package]
name = "audit-cli"
version = "0.1.0"
edition = "2021"

[dependencies]
anyhow = "1"
clap = { version = "4", features = ["derive"] }
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
hex = "0.4"

# Subxt (pilih versi stabil sesuai rilis terbaru)
subxt = { version = "0.43", default-features = false, features = ["jsonrpsee", "native", "metadata"] }

```

### Implementasi CLI
src/main.rs
```bash
use anyhow::{Context, Result};
use clap::{Parser, ValueEnum};
use subxt::{
    config::polkadot::PolkadotConfig,
    dynamic::Value,
    tx::PairSigner,
    OnlineClient,
};
use subxt::ext::sp_core::sr25519;

#[derive(Copy, Clone, Debug, ValueEnum)]
enum InputKind { Hex, Utf8, File }

#[derive(Parser, Debug)]
#[command(name="audit-cli", about="Submit extrinsic via Subxt")]
struct Opts {
    #[arg(long, default_value="ws://127.0.0.1:9944")]
    url: String,

    #[arg(long, default_value="//Alice")]
    seed: String,

    #[arg(long, value_enum, default_value_t=InputKind::Utf8)]
    input: InputKind,

    #[arg(long)]
    data: String,

    #[arg(long, default_value="Audit")]
    pallet: String,

    #[arg(long, default_value="create_record")]
    call: String,

    #[arg(long, default_value="finalized")]
    wait: String,

    #[arg(long, default_value_t=false)]
    as_hash: bool,
}

#[tokio::main]
async fn main() -> Result<()> {
    let opts = Opts::parse();

    let api = OnlineClient::<PolkadotConfig>::from_url(&opts.url)
        .await
        .with_context(|| format!("Connect failed: {}", opts.url))?;

    let pair = sr25519::Pair::from_string(&opts.seed, None)
        .map_err(|e| anyhow::anyhow!("Bad seed: {e}"))?;
    let signer = PairSigner::<PolkadotConfig, _>::new(pair);

    let mut raw: Vec<u8> = match opts.input {
        InputKind::Hex => hex::decode(opts.data.trim_start_matches("0x"))?,
        InputKind::Utf8 => opts.data.as_bytes().to_vec(),
        InputKind::File => std::fs::read(&opts.data)?,
    };

    let args: Vec<Value> = if opts.as_hash {
        use subxt::utils::H256;
        use subxt::ext::sp_core::hashing::blake2_256;
        let digest = blake2_256(&raw);
        vec![Value::from(H256::from_slice(&digest))]
    } else {
        vec![Value::from_bytes(std::mem::take(&mut raw))]
    };

    let call = subxt::dynamic::tx(&opts.pallet, &opts.call, args);

    let mut progress = api.tx()
        .sign_and_submit_then_watch_default(&call, &signer)
        .await?;

    let events = match opts.wait.as_str() {
        "inblock" | "in_block" => progress.wait_for_in_block().await?,
        _ => progress.wait_for_finalized_success().await?,
    };

    println!("Included at block: 0x{}", hex::encode(events.block_hash().as_ref()));
    for ev in events.as_events().iter() {
        if let Ok(details) = ev {
            println!("Event: {}.{}", details.pallet_name(), details.variant_name());
        }
    }
    Ok(())
}
```

### Cara Menjalankan
1. Build
- Pallet Audit → create_record(Vec<u8>):
```
cargo build --release
```
2. Jalankan node (di terminal lain)
```
./target/release/node-template --dev
```
3. Coba kirim extrinsic
- Submit hash (Blake2_256 di client) → submit_hash(H256):
```
./target/release/audit-cli --input utf8 --data "hello day8"
```
- Pallet System → remark(Vec<u8>):
```
./target/release/audit-cli --input utf8 --data "hello day8"
```
- Pallet System → remark(Vec<u8>):
```
./target/release/audit-cli --pallet System --call remark --input hex --data 0x48656c6c6f
```
- Submit hash (Blake2_256 di client) → submit_hash(H256):
```
./target/release/audit-cli \
  --pallet Audit --call submit_hash \
  --input utf8 --data "hello day8" \
  --as-hash
```

## Verifikasi
- Buka Polkadot.js Apps
 → Developer → Explorer → Events.
- Lihat event dari pallet (Audit.RecordCreated, Audit.HashSubmitted, atau system.Remarked).
- CLI juga mencetak hash block & nama event.