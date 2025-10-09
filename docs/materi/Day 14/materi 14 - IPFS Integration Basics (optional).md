# Day 14 — IPFS Integration Basics (Optional)
**Topic:** IPFS Integration Basics (optional)  
**Objective:** Optional: link hash to IPFS for metadata  
**Task:** Add IPFS hash uploader (optional bonus)  

---

## Ringkasan
Tujuan opsional ini: **menghubungkan hash on-chain** (dari pallet audit-log) dengan **metadata di IPFS**. On-chain tetap ringan (hash 32-byte), sedangkan metadata lengkap (round, institution/client tag, checksum, dsb.) disimpan **off-chain** sebagai **CID** (Content ID).

Manfaat:
- Hemat biaya on-chain, tapi tetap punya *rich metadata*.
- Verifikasi: **file ⇒ SHA-256** cocok dengan **hash on-chain**, dan metadata IPFS menyebut hash yang sama.
- Portabel: CID bisa dipin di node lokal/cluster atau layanan pinning.

**Referensi:**
- Instalasi IPFS: <https://docs.ipfs.tech/install/>  
- Video overview: <https://www.youtube.com/watch?v=6Yk9gPVzELU>

---

## 1) Arsitektur & Alur
1. **Klien** menyiapkan artefak kontribusi (mis. `client01_round5_delta.bin`) + bukti (`evidence_round5.txt`).
2. Hitung **SHA-256** → `model_sha256` dan `evidence_sha256` (32-byte hex).
3. Buat **metadata JSON** yang mencantumkan kedua checksum + konteks (round, institution, dsb.).
4. **Upload metadata JSON** ke IPFS → dapat **CID (v1 base32)**.
5. (Opsional) Panggil extrinsic **`link_ipfs(hash, cid)`** untuk mengikat hash on-chain ke CID.

> On-chain tetap menyimpan hash 32-byte (kunci verifikasi). CID adalah pointer metadata off-chain.

---

## 2) Skema Metadata JSON (disarankan)
Simpan sebagai `audit_meta.json`:
```json
{
  "round_id": 5,
  "institution": "DUKCAPIL",
  "client_id": "client-ukdw-01",
  "model_sha256": "0x4fe2...c85a",
  "evidence_sha256": "0x9b7c...11aa",
  "tag": "pilot-A",
  "ts": 1726886400,
  "notes": "optional short note"
}
````

* **wajib**: `model_sha256` harus **persis** hash yang kamu submit ke chain.
* Hindari PII; metadata hanya ringkas & verifikasi.

---

## 3) Ekstensi Pallet (opsional): Simpan mapping `hash → CID`

Tambahkan storage & extrinsic ringan ke `pallet-audit-log`.

### 3.1 Storage & Config

```rust
// di #[pallet::config]
#[pallet::constant] type MaxCidLen: Get<u32>;

// di storage
#[pallet::storage]
pub type IpfsOfHash<T: Config> = StorageMap<
    _,
    Blake2_128Concat,
    T::Hash,
    BoundedVec<u8, T::MaxCidLen>, // CID v1 base32 sebagai bytes
    OptionQuery
>;
```

### 3.2 Event & Error

```rust
#[pallet::event]
#[pallet::generate_deposit(pub(super) fn deposit_event)]
pub enum Event<T: Config> {
    // ...
    IpfsLinked { who: T::AccountId, hash: T::Hash, cid_len: u32 },
}

#[pallet::error]
pub enum Error<T> {
    AlreadyExists,
    AccountIndexFull,
    RecentFull,
    NotOwner,       // hanya pemilik log yang boleh link
    AlreadyLinked,  // hash sudah punya CID
    CidTooLong,     // melebihi MaxCidLen
    NotFound,       // hash belum ada di HashLog
}
```

### 3.3 Extrinsic `link_ipfs`

```rust
#[pallet::call]
impl<T: Config> Pallet<T> {
    /// Link-kan CID IPFS ke hash yang sudah ada di HashLog.
    #[pallet::call_index(1)]
    #[pallet::weight(T::WeightInfo::link_ipfs(cid.len() as u32))]
    pub fn link_ipfs(origin: OriginFor<T>, hash: T::Hash, cid: Vec<u8>) -> DispatchResult {
        let who = ensure_signed(origin)?;

        // pastikan hash ada
        let log = HashLog::<T>::get(&hash).ok_or(Error::<T>::NotFound)?;

        // opsional: hanya pemilik (pencatat pertama) boleh link
        ensure!(log.who == who, Error::<T>::NotOwner);

        // belum pernah dilink
        ensure!(!IpfsOfHash::<T>::contains_key(&hash), Error::<T>::AlreadyLinked);

        // validasi panjang CID
        let bv: BoundedVec<_, T::MaxCidLen> =
            cid.clone().try_into().map_err(|_| Error::<T>::CidTooLong)?;

        IpfsOfHash::<T>::insert(&hash, bv);
        Self::deposit_event(Event::IpfsLinked { who, hash, cid_len: cid.len() as u32 });
        Ok(())
    }
}
```

### 3.4 WeightInfo

```rust
pub trait WeightInfo {
    fn submit_hash() -> Weight;
    fn link_ipfs(n: u32) -> Weight;
}

impl WeightInfo for () {
    fn submit_hash() -> Weight { Weight::from_parts(10_000, 0) }
    fn link_ipfs(_n: u32) -> Weight { Weight::from_parts(10_000, 0) }
}
```

---

## 4) Integrasi Runtime (contoh)

```rust
parameter_types! {
    pub const MaxPerAccount: u32 = 64;
    pub const MaxRecent: u32 = 256;
    pub const MaxCidLen: u32 = 90; // CIDv1 base32 tipikal < 90 chars utk JSON kecil
}

impl pallet_audit_log::Config for Runtime {
    type RuntimeEvent = RuntimeEvent;
    type Moment = u64;
    type MaxPerAccount = MaxPerAccount;
    type MaxRecent = MaxRecent;
    type MaxCidLen = MaxCidLen;
    type WeightInfo = ();
}
```

---

## 5) Instal & Menjalankan IPFS (lokal cepat)

**Linux/macOS** (go-ipfs):

```bash
# 1) ikuti panduan resmi
# https://docs.ipfs.tech/install/
# 2) inisialisasi & jalankan daemon
ipfs init
ipfs daemon
```

**Windows**: gunakan installer resmi / `choco install ipfs`.
Cek versi:

```bash
ipfs version
```

Tambah file ke IPFS (uji cepat):

```bash
echo '{"hello":"ipfs"}' > audit_meta.json
ipfs add audit_meta.json
# output: added <CID> audit_meta.json
```

Ambil konten kembali:

```bash
ipfs cat <CID>
```

> Untuk deploy non-lokal, pertimbangkan **pinning service** (Pinata, Web3.Storage, dll.).

---

## 6) Uploader+Linker (Node.js): upload JSON → dapat CID → panggil extrinsic

Script ini: (1) hitung SHA-256 file artefak & bukti, (2) buat `audit_meta.json`, (3) upload ke IPFS lokal, (4) panggil extrinsic `link_ipfs`.

### 6.1 Persiapan

```bash
npm init -y
npm i ipfs-http-client @polkadot/api
# tambahkan "type": "module" di package.json agar bisa pakai 'import'
```

### 6.2 `upload_and_link.js`

```js
import fs from 'fs';
import crypto from 'crypto';
import { create } from 'ipfs-http-client';
import { ApiPromise, WsProvider, Keyring } from '@polkadot/api';

const WS = process.env.WS || 'ws://127.0.0.1:9944';
const SUBJECT_HASH = process.env.HASH; // 0x... (hash yang sudah submit_hash ke chain)
const MODEL = process.env.MODEL || './client01_round5_delta.bin';
const EVID  = process.env.EVID  || './evidence_round5.txt';
const ROUND = Number(process.env.ROUND || '5');
const INST  = process.env.INST  || 'DUKCAPIL';
const CLIENT= process.env.CLIENT|| 'client-ukdw-01';

if (!SUBJECT_HASH) {
  console.error('Set environment: HASH=0x... (on-chain hash to link)');
  process.exit(1);
}

function sha256Hex(path) {
  const h = crypto.createHash('sha256');
  h.update(fs.readFileSync(path));
  return '0x' + h.digest('hex');
}

const modelHex = sha256Hex(MODEL);
const evidHex  = sha256Hex(EVID);

const meta = {
  round_id: ROUND,
  institution: INST,
  client_id: CLIENT,
  model_sha256: modelHex,
  evidence_sha256: evidHex,
  ts: Math.floor(Date.now() / 1000),
};
fs.writeFileSync('audit_meta.json', JSON.stringify(meta));

// 1) connect IPFS local (default API 127.0.0.1:5001)
const ipfs = create({ url: 'http://127.0.0.1:5001' });
const { cid } = await ipfs.add(fs.readFileSync('audit_meta.json'));
const CID = cid.toString(); // v1 base32
console.log('IPFS CID:', CID);

// 2) link ke chain
const api = await ApiPromise.create({ provider: new WsProvider(WS) });
const keyring = new Keyring({ type: 'sr25519' });
const alice = keyring.addFromUri('//Alice');

const cidBytes = Buffer.from(CID, 'utf8');
const tx = api.tx.auditLog.linkIpfs(SUBJECT_HASH, cidBytes);

const unsub = await tx.signAndSend(alice, ({ status, dispatchError, events }) => {
  if (dispatchError) {
    if (dispatchError.isModule) {
      const decoded = api.registry.findMetaError(dispatchError.asModule);
      console.error('Module error:', decoded.section, decoded.name);
    } else {
      console.error('Dispatch error:', dispatchError.toString());
    }
    process.exit(1);
  }
  if (status.isInBlock) console.log('In block', status.asInBlock.toHex());
  if (status.isFinalized) {
    console.log('Finalized at', status.asFinalized.toHex());
    events.forEach(({ event }) => {
      if (event.section === 'auditLog' && event.method === 'IpfsLinked') {
        console.log('IpfsLinked event:', event.toHuman());
      }
    });
    unsub();
    process.exit(0);
  }
});
```

**Cara pakai (Linux/macOS bash):**

```bash
# di terminal 1
ipfs daemon

# di terminal 2 (pastikan hash on-chain sudah ada via submit_hash)
HASH=0x4fe2a1...c85a \
MODEL=./client01_round5_delta.bin \
EVID=./evidence_round5.txt \
ROUND=5 INST=DUKCAPIL CLIENT=client-ukdw-01 \
node upload_and_link.js
```

**Cara pakai (Windows PowerShell):**

```powershell
# terminal 1
ipfs daemon

# terminal 2
$env:HASH="0x4fe2a1...c85a"
$env:MODEL=".\client01_round5_delta.bin"
$env:EVID=".\evidence_round5.txt"
$env:ROUND="5"
$env:INST="DUKCAPIL"
$env:CLIENT="client-ukdw-01"
node .\upload_and_link.js
```

---

## 7) Verifikasi End-to-End

1. **On-chain:**

   * `auditLog.HashLog(0xHASH)` → `Some(LogEntry { who, block, time })`
   * `auditLog.IpfsOfHash(0xHASH)` → `Some(<CID-bytes>)`
2. **IPFS:**

   * `ipfs cat <CID>` → JSON berisi `model_sha256` = `0xHASH` yang sama.
3. **Event:**

   * Lihat `auditLog.IpfsLinked` di Explorer → param `hash`, `cid_len` wajar (< MaxCidLen).
4. (Opsional) **Query by Institution** (Day 13):

   * Ambil hash hasil filter institusi → ambil `IpfsOfHash` untuk tiap hash → tampilkan CID dan metadata.

---

## 8) Catatan Keamanan & Operasional

* **Pinning**: pastikan CID kamu dipin (lokal &/atau layanan pinning) agar tidak GC.
* **Privasi**: metadata **jangan** mengandung PII. Simpan yang minimal untuk verifikasi audit.
* **Format CID**: gunakan **CID v1 base32** (default ipfs-http-client). Batasi panjang via `MaxCidLen`.
* **Kesesuaian Hash**: selalu cocokkan `model_sha256` di metadata dengan hash on-chain.
* **Gateway**: hindari bergantung pada gateway publik untuk operasi kritikal; gunakan IPFS API lokal/cluster.

---

## 9) Checklist Day 14
* [ ] Install & jalankan IPFS lokal (daemon).
* [ ] Bentuk metadata JSON (schema di atas).
* [ ] Upload ke IPFS → dapat CID.
* [ ] Tambah storage & extrinsic `link_ipfs` (opsional) ke pallet `audit-log`.
* [ ] Panggil `link_ipfs(hash, cid)` dari klien.
* [ ] Verifikasi: `IpfsOfHash(hash)` mengembalikan CID; `ipfs cat CID` → metadata benar.
* [ ] Dokumentasikan alur + bukti verifikasi (screenshot/CLI log).