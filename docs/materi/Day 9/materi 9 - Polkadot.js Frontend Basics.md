# Day 9 — Polkadot.js Frontend Basics (Extension + Query State)

> Target: *Query on‑chain state* dari browser memakai **Polkadot{.js} Extension** dan **Polkadot.js Apps / @polkadot/api**; tampilkan hasil dalam **JSON view**.

---

## Ringkasan Materi

* **Polkadot{.js} Extension**: pengelola akun & signer di browser (Chrome/Firefox). Dapps meminta signature lewat extension.
* **Polkadot.js Apps**: UI web untuk melihat state, mengirim extrinsic, events, dsb.
* **@polkadot/api**: library JS/TS untuk connect WS, query storage, submit tx dari frontend.

---

## 1) Prasyarat

1. **Node lokal** aktif di `ws://127.0.0.1:9944` (contoh: node-template `--dev`).
2. Browser **Chrome**/**Firefox**.

---

## 2) Install & Siapkan Polkadot{.js} Extension

1. Pasang extension (Chrome/Firefox) dari store resmi.
2. Buka extension → **Add account** → buat akun dev (atau import dev key). Simpan mnemonic dengan aman.
3. Buka halaman **Polkadot.js Apps**.
4. Di pojok kanan atas, pindah network ke **Local Node** (ws\://127.0.0.1:9944).
5. Saat Apps meminta akses akun, **Approve** di extension.

> Catatan: Extension bukan wallet penuh, namun account manager + signer. Dapp tidak pernah melihat mnemonic; hanya signature yang diberikan.

---

## 3) Query State via Polkadot.js Apps (JSON View)

1. **Developer → Chain state**.
2. Pilih modul dan item storage, misalnya:

   * `system → account(AccountId32)`
   * `balances → freeBalance(AccountId32)` atau `balances → locks(AccountId32)` tergantung runtime
   * `audit → records(Hash)` (pallet `audit` dari Day 8)
3. Isi parameter (contoh SS58 `//Alice`).
4. Klik **+** untuk mengeksekusi. Hasil akan tampil di panel kanan.
5. Klik **ikon JSON** (atau toggle raw) untuk melihat **JSON view** state.

Tips:

* Aktifkan **include options** untuk item bertipe `Option<...>`.
* Kamu bisa menambahkan beberapa query sekaligus dan menyimpan preset.

---

## 4) Query State di Browser dengan @polkadot/api (Tanpa Build Tools)

>

Buat file `index.html`:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Day 9 — Polkadot.js Query Demo</title>
  <style>body{font:14px system-ui, sans-serif; margin:24px} pre{background:#111; color:#0f0; padding:12px; overflow:auto}</style>
</head>
<body>
  <h1>Polkadot.js Query Demo</h1>
  <label>WS Endpoint: <input id="ws" value="ws://127.0.0.1:9944" size="35"></label>
  <button id="connect">Connect & Query</button>
  <pre id="out">(output JSON)</pre>

  <script type="module">
    import { ApiPromise, WsProvider } from "https://cdn.skypack.dev/@polkadot/api";

    const $ = (id) => document.getElementById(id);
    const out = $("out");

    function print(obj){ out.textContent = JSON.stringify(obj, null, 2); }

    document.getElementById("connect").onclick = async () => {
      try {
        const endpoint = $("ws").value.trim();
        const api = await ApiPromise.create({ provider: new WsProvider(endpoint) });

        // 1) Versi runtime & chain
        const [chain, node, version] = await Promise.all([
          api.rpc.system.chain(),
          api.rpc.system.name(),
          api.rpc.system.version()
        ]);

        // 2) Ambil block terbaru & hash
        const header = await api.rpc.chain.getHeader();
        const blockHash = await api.rpc.chain.getBlockHash(header.number);

        // 3) Query storage: System.Account untuk Alice dev
        const alice = "5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY";
        const account = await api.query.system.account(alice);

        // 4) (Opsional) Query pallet custom, contoh: audit.records(hash)
        // const hash = api.registry.createType('Hash', '0x...');
        // const rec = await api.query.audit.records(hash);

        print({
          chain: chain.toString(),
          node: node.toString(),
          version: version.toString(),
          bestBlock: header.number.toNumber(),
          blockHash: blockHash.toHex(),
          aliceAccount: account.toJSON()
          //, auditRecord: rec?.toJSON()
        });

        api.disconnect();
      } catch (e) {
        print({ error: String(e) });
      }
    };
  </script>
</body>
</html>
```

> Buka file ini di browser (double‑click atau via live server). Klik **Connect & Query** → hasil JSON tampil di `<pre>`.

---

## 5) Query Historical State (Block tertentu)

Contoh query storage pada block hash tertentu (pruning default 256 block):

```js
const header = await api.rpc.chain.getHeader();
const at = await api.at(header.hash);
const aliceAt = await at.query.system.account(alice);
console.log(aliceAt.toJSON());
```

---

## 6) Bonus: Hubungkan Extension ↔ Dapp

Saat dapp kamu butuh signature (misalnya submit extrinsic dari browser), gunakan **@polkadot/extension-dapp** untuk meminta akses akun & signature:

```html
<script type="module">
  import { web3Enable, web3Accounts } from 'https://cdn.skypack.dev/@polkadot/extension-dapp';

  const extensions = await web3Enable('My Dapp');
  if (!extensions.length) throw new Error('No extension approved');

  const accounts = await web3Accounts();
  console.log('Injected accounts:', accounts.map(a => a.address));
</script>
```

> Nanti di Day berikutnya kita pakai ini untuk **submit extrinsic** langsung dari frontend.

---

## 7) Checklist Day 9

* [ ] Extension terpasang & akun dev dibuat/import.
* [ ] Apps terhubung ke node lokal & akses extension di-approve.
* [ ] **Chain state** berhasil di-query (System.Account / Balances / pallet audit).
* [ ] **JSON view** tampil dari Apps atau dari dapp demo.

---

## 8) Troubleshooting

* **Apps tidak melihat akun**: pastikan extension **Allow access** untuk situs Apps; cek jaringan yang dipilih benar (Local Node).
* **Gagal connect WS**: pastikan node `--dev` aktif dan port 9944 tidak diblok firewall.
* **Query item tidak ada**: runtime custom bisa beda; cek nama modul & storage dari *metadata* (Developer → Metadata) dan sesuaikan.
* **CORS saat pakai file HTML**: sebagian browser membatasi `file://`; gunakan Live Server atau `python -m http.server` sederhana.

---

## 9) Tugas Hari Ini

1. Tampilkan `system.account(Alice)` via **Apps** dan **JSON view**.
2. Query 1 item lain (mis. `balances.locks(Alice)` atau storage pallet kamu) dan **screenshot** JSON‑nya.
3. Jalankan `index.html` demo dan pastikan JSON berisi `chain`, `bestBlock`, `aliceAccount`.

---

## 10) Catatan Lanjutan

* Pelajari **state queries** dan resep di *cookbook* untuk pattern umum (`Option<>`, iterasi map, paginasi keys).
* Untuk chain kustom, kamu bisa memuat **custom types** agar decode JSON rapi di Apps & API.

---

**Selesai — Day 9** ✅
