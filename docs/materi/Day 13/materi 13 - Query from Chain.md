# Day 13 — Query from Chain
**Topic:** Query from Chain  
**Objective:** Query stored hashes and filter by institution  
**Task:** Write query logic from chain to retrieve history  

---

## Ringkasan
Hari ini fokus **membaca** data dari chain: mengambil riwayat hash kontribusi model dan **memfilter berdasarkan institusi**. Kita sediakan dua jalur:

1. **Tanpa indeks khusus institusi (pakai desain Day 12):**  
   - Ambil daftar **member akun** suatu institusi (via `pallet_membership` / registry akun → institusi).  
   - Untuk tiap akun, baca `ByAccount(account)` → gabungkan & *deduplicate* hash → tarik detail `HashLog(hash)`.

2. **Dengan indeks khusus institusi (opsional, jika kamu tambahkan di pallet):**  
   - Baca langsung `ByInstitution(institutionId)` → daftar hash → detail `HashLog(hash)`.

**Referensi:**  
- Substrate tutorial (struktur runtime & storage): <https://docs.substrate.io/tutorials/build-a-blockchain/>  
- Video pengantar (baca data dari chain): <https://www.youtube.com/watch?v=cJ0EF2Jv7_s>

---

## 1) Storage (Day 12)
Dari pallet `audit-log` (Opsi A):

- `HashLog(hash) -> LogEntry { who, block, time? }`  
- `ByAccount(account) -> BoundedVec<Hash, MaxPerAccount>`  
- (Opsional) `Recent() -> BoundedVec<Hash, MaxRecent>`

**Sumber data “institusi”:**
- `pallet_membership` per institusi (Instance1: DUKCAPIL, Instance2: DINSOS, Instance3: KEMENKES) → `Members: Vec<AccountId>`
- **atau** registry sederhana `InstitutionOf(AccountId) -> InstitutionId` (pallet lain).
- **atau** file *off-chain* (JSON) peta akun → institusi (untuk demo cepat).

---

## 2) Polkadot.js — Node script (siap jalan)
> Simpan sebagai `query-audit-history.js`, jalankan `node query-audit-history.js` (Node 18+).

```js
// query-audit-history.js
import { ApiPromise, WsProvider } from '@polkadot/api';
import { u8aToHex } from '@polkadot/util';

// ======== CONFIG ========
const WS = process.env.WS || 'ws://127.0.0.1:9944';
const INSTITUTION = process.env.INST || 'DUKCAPIL';

// === VARIAN-A: membership pallet (multi-instance) ===
const MEMBERSHIP_PATH = {
  DUKCAPIL:  { pallet: 'membershipDuk',  storage: 'Members' },
  DINSOS:    { pallet: 'membershipDis',  storage: 'Members' },
  KEMENKES:  { pallet: 'membershipKem',  storage: 'Members' },
};

// === VARIAN-B: registry AccountId -> InstitutionId ===
const REGISTRY = { pallet: 'institution', storage: 'InstitutionOf' };

// Pallet audit log (Day 12)
const AUDIT_PALLET = {
  byAccount: { pallet: 'auditLog', storage: 'ByAccount' },
  hashLog:   { pallet: 'auditLog', storage: 'HashLog'   },
};

// Mode sumber institusi: 'membership' atau 'registry'
const SOURCE_MODE = process.env.SOURCE_MODE || 'membership';
// ======== END CONFIG ========

const uniq = (arr) => [...new Set(arr)];

async function getMembersFromMembership(api, institutionKey) {
  const path = MEMBERSHIP_PATH[institutionKey];
  if (!path) throw new Error(`Membership path for ${institutionKey} not configured`);
  const entries = await api.query[path.pallet][path.storage]();
  return entries.map((acc) => acc.toString());
}

async function getMembersFromRegistry(api, institutionKey) {
  const keys = await api.query[AUDIT_PALLET.byAccount.pallet][AUDIT_PALLET.byAccount.storage].keys();
  const accounts = uniq(keys.map((k) => k.args[0].toString()));

  const result = [];
  for (const a of accounts) {
    const inst = await api.query[REGISTRY.pallet][REGISTRY.storage](a);
    const s = inst.toHuman ? inst.toHuman() : inst.toString();
    if ((s || '') === institutionKey) result.push(a);
  }
  return result;
}

async function main() {
  const api = await ApiPromise.create({ provider: new WsProvider(WS) });
  console.log(`Connected: ${WS}`);

  let members = [];
  if (SOURCE_MODE === 'membership') members = await getMembersFromMembership(api, INSTITUTION);
  else members = await getMembersFromRegistry(api, INSTITUTION);
  console.log(`Members of ${INSTITUTION}:`, members);

  const allHashes = [];
  for (const acc of members) {
    const hashes = await api.query[AUDIT_PALLET.byAccount.pallet][AUDIT_PALLET.byAccount.storage](acc);
    allHashes.push(...hashes.map((h) => (h.toHex ? h.toHex() : u8aToHex(h))));
  }
  const hashesUniq = uniq(allHashes);
  console.log(`Total uniq hashes: ${hashesUniq.length}`);

  const details = [];
  for (const h of hashesUniq) {
    const rec = await api.query[AUDIT_PALLET.hashLog.pallet][AUDIT_PALLET.hashLog.storage](h);
    if (rec.isSome) {
      const ent = rec.unwrap();
      details.push({
        hash: h,
        who: ent.who.toString(),
        block: ent.block.toString(),
        time: ent.time.isSome ? ent.time.unwrap().toString() : null,
      });
    }
  }
  details.sort((a, b) => BigInt(a.block) - BigInt(b.block));
  console.table(details);

  await api.disconnect();
}

main().catch((e) => { console.error(e); process.exit(1); });
````

**Cara pakai:**

```bash
# mode membership (default)
node query-audit-history.js

# mode registry
SOURCE_MODE=registry node query-audit-history.js

# ganti endpoint / institusi
WS=ws://127.0.0.1:9946 INST=DINSOS node query-audit-history.js
```

---

## 3) Polkadot.js — Console (cepat untuk demo)

> Buka **Developer → JavaScript** di polkadot.js Apps. Paste salah satu varian di bawah.

### Varian-A (membership)

```js
const INSTITUTION = 'DUKCAPIL';
const M = { pallet: 'membershipDuk', storage: 'Members' };
const AUD = { byAccount: ['auditLog', 'ByAccount'], hashLog: ['auditLog', 'HashLog'] };

(async () => {
  const members = await api.query[M.pallet][M.storage]();
  const accs = members.map(x => x.toString());
  console.log('Members:', accs);

  const uniq = (a) => [...new Set(a)];
  const allHashes = [];
  for (const a of accs) {
    const hs = await api.query[AUD.byAccount[0]][AUD.byAccount[1]](a);
    allHashes.push(...hs.map(h => h.toHex()));
  }
  const hashes = uniq(allHashes);

  const details = [];
  for (const h of hashes) {
    const rec = await api.query[AUD.hashLog[0]][AUD.hashLog[1]](h);
    if (rec.isSome) {
      const r = rec.unwrap();
      details.push({ hash: h, who: r.who.toString(), block: r.block.toString(), time: r.time.isSome ? r.time.unwrap().toString() : null });
    }
  }
  details.sort((a,b)=> BigInt(a.block) - BigInt(b.block));
  console.table(details);
})();
```

### Varian-B (registry)

```js
const INSTITUTION = 'DUKCAPIL';
const REG = ['institution','InstitutionOf'];
const AUD = { byAccount: ['auditLog', 'ByAccount'], hashLog: ['auditLog', 'HashLog'] };

(async () => {
  const keys = await api.query[AUD.byAccount[0]][AUD.byAccount[1]].keys();
  const accounts = [...new Set(keys.map(k => k.args[0].toString()))];

  const accs = [];
  for (const a of accounts) {
    const inst = await api.query[REG[0]][REG[1]](a);
    const s = inst.toHuman ? inst.toHuman() : inst.toString();
    if ((s||'') === INSTITUTION) accs.push(a);
  }
  console.log('Members by registry:', accs);

  const uniq = (a) => [...new Set(a)];
  const allHashes = [];
  for (const a of accs) {
    const hs = await api.query[AUD.byAccount[0]][AUD.byAccount[1]](a);
    allHashes.push(...hs.map(h => h.toHex()));
  }
  const hashes = uniq(allHashes);

  const details = [];
  for (const h of hashes) {
    const rec = await api.query[AUD.hashLog[0]][AUD.hashLog[1]](h);
    if (rec.isSome) {
      const r = rec.unwrap();
      details.push({ hash: h, who: r.who.toString(), block: r.block.toString(), time: r.time.isSome ? r.time.unwrap().toString() : null });
    }
  }
  details.sort((a,b)=> BigInt(a.block) - BigInt(b.block));
  console.table(details);
})();
```

---

## 4) Rust — Subxt (query + filter singkat)

> Buat crate `audit_query`, dump metadata: `subxt metadata -f bytes > metadata.scale`.

**`Cargo.toml`**

```toml
[package]
name = "audit_query"
version = "0.1.0"
edition = "2021"

[dependencies]
anyhow = "1"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
subxt = { version = "0.37", default-features = false, features = ["substrate-compat"] }
```

**`src/main.rs`**

```rust
use anyhow::Result;
use std::collections::BTreeSet;
use subxt::{OnlineClient, PolkadotConfig, utils::AccountId32};

#[subxt::subxt(runtime_metadata_path = "metadata.scale")]
pub mod runtime {}

#[tokio::main]
async fn main() -> Result<()> {
    let api = OnlineClient::<PolkadotConfig>::new().await?;

    // Contoh: ambil members dari membership instance "DUK"
    let members: Vec<AccountId32> = api
        .storage()
        .at_latest()
        .await?
        .fetch(&runtime::storage().membership_duk().members())
        .await?
        .unwrap_or_default();

    let mut hashes = BTreeSet::new();
    for m in &members {
        let list = api
            .storage()
            .at_latest()
            .await?
            .fetch(&runtime::storage().audit_log().by_account(m.clone()))
            .await?
            .unwrap_or_default();
        for h in list {
            hashes.insert(format!("{:?}", h)); // convert to hex if needed
        }
    }

    println!("Total uniq hashes: {}", hashes.len());

    // Alternatif: iterasi langsung kunci HashLog & filter by who ∈ members
    let keys = api
        .storage()
        .at_latest()
        .await?
        .keys(&runtime::storage().audit_log().hash_log_root())
        .await?;

    let member_set: BTreeSet<_> = api
        .storage()
        .at_latest()
        .await?
        .fetch(&runtime::storage().membership_duk().members())
        .await?
        .unwrap_or_default()
        .into_iter()
        .collect();

    for k in keys {
        if let Some(rec) = api
            .storage()
            .at_latest()
            .await?
            .fetch(&runtime::storage().audit_log().hash_log(k.clone()))
            .await? {
            if member_set.contains(&rec.who) {
                println!("block={} who={} hash_key={:?} time={:?}", rec.block, rec.who, k, rec.time);
            }
        }
    }

    Ok(())
}
```

> Nama pallet & storage yang dipanggil Subxt mengikuti **metadata chain** kamu. Cek modul yang dihasilkan oleh `#[subxt::subxt]`.

---

## 5) Desain Indeks **ByInstitution** (opsional, untuk query super cepat)

Jika query institusional sering, tambah indeks langsung di pallet:

```rust
#[pallet::storage]
pub type ByInstitution<T: Config> = StorageMap<
    _, Blake2_128Concat,
    InstitutionId,                      // enum/BoundedVec<u8>, dsb
    BoundedVec<T::Hash, T::MaxPerInst>, // daftar hash
    ValueQuery
>;
```

* Update `submit_hash`: push `hash` ke `ByInstitution(inst_of(&who))`.
* Sediakan fungsi `inst_of(AccountId) -> InstitutionId` (ambil dari pallet registry/membership).
* Query per institusi jadi **O(1) key**.

---

## 6) Checklist Day 13

* [ ] Tentukan **sumber data institusi** (membership vs registry).
* [ ] Jalankan **polkadot.js Node script** / **Console** untuk tarik riwayat hash institusi.
* [ ] (Opsional) Buat **Subxt** CLI untuk laporan otomatis.
* [ ] (Opsional) Tambah **ByInstitution** index jika query institusional sering dan perlu cepat.
* [ ] Dokumentasikan contoh output (tabel block/who/time/hash) sebagai bukti verifikasi.

