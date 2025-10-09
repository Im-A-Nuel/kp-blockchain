# Topic

Review Hash Traceability

# Objective

Menguji keterlacakan (traceability) dan konsistensi logika data yang disimpan oleh `pallet-audit`.

---

# A) Success Criteria

1. **Unik per-hash**: satu `model_hash (H256)` hanya memetakan ke **satu** `id`.
2. **Jejak lengkap**: `records(id)` menyimpan `who`, `at_block`, `institution (snapshot)`, `ipfs_cid?`, `note?`.
3. **Index konsisten**: setelah `submit`, ketiga index sinkron:

   * `ByHash[hash] = Some(id)`
   * `ByAccount[who]` berisi `id`
   * `ByInstitution[inst]` berisi `id`
4. **Akses terkontrol**: akun non-whitelist **ditolak** (`NotAuthorized`).
5. **Idempotensi**: `submit` ulang dengan hash yang sama **gagal** (`AlreadyExists`).

---

# B) Invariant (wajib selalu benar)

* `ByHash.contains_key(h) == Records.contains_key(ByHash[h])`.
* Setiap `id ∈ ByAccount[who]` → `Records[id].who == who`.
* Setiap `id ∈ ByInstitution[inst]` → `Records[id].institution == inst`.

---

# C) Test Plan – Manual (Polkadot.js)

1. **Authorize** (ALICE via sudo):
   `sudo.sudo(audit.forceAuthorize(BOB, "DINSOS-KAB"))` → event `AuthorizedSet`.
2. **Submit** (BOB):

   * `audit.submit(modelHash=0x<hashA>, ipfsCid=None, note=None)` → event `Submitted{id=0,...}`.
3. **Verifikasi state**:

   * `audit.byHash(0x<hashA>)` → `0`
   * `audit.records(0)` → nilai fields sesuai (who=BOB, institution="DINSOS-KAB")
   * `audit.byAccount(BOB)` → `[0]`
   * `audit.byInstitution("DINSOS-KAB")` → `[0]`
4. **Non-authorized** (CHARLIE):
   `audit.submit(0x<hashB>, None, None)` → **gagal** `NotAuthorized`.
5. **Duplicate** (BOB):
   `audit.submit(0x<hashA>, None, None)` → **gagal** `AlreadyExists`.

*(Tips membuat hash: `sha256sum model.pt | awk '{print "0x"$1}'`)*

---

# D) Test Otomatis – Unit tests (siap tempel)

Di `pallets/pallet-audit/src/tests.rs` (atau `lib.rs` di bawah `#[cfg(test)]`), pakai mock runtime sederhana.

```rust
// tests.rs
#![cfg(test)]
use super::*;
use frame_support::{
    assert_ok, assert_noop, construct_runtime, parameter_types, traits::Everything,
    BoundedVec,
};
use frame_system as system;
use sp_core::H256;
use sp_runtime::{testing::Header, traits::{BlakeTwo256, IdentityLookup}};

// --- Mock Runtime ---
type UncheckedExtrinsic = frame_system::mocking::MockUncheckedExtrinsic<Test>;
type Block = frame_system::mocking::MockBlock<Test>;

construct_runtime!(
    pub enum Test where
        Block = Block,
        NodeBlock = Block,
        UncheckedExtrinsic = UncheckedExtrinsic,
    {
        System: frame_system,
        Audit: pallet_audit,
    }
);

parameter_types! {
    pub const BlockHashCount: u64 = 2400;
    pub const AuditMaxInstNameLen: u32 = 64;
    pub const AuditMaxNoteLen: u32 = 256;
    pub const AuditMaxCidLen: u32 = 64;
    pub const AuditMaxPerAccount: u32 = 100;
    pub const AuditMaxPerInstitution: u32 = 100;
}

impl system::Config for Test {
    type BaseCallFilter = Everything;
    type BlockWeights = ();
    type BlockLength = ();
    type DbWeight = ();
    type RuntimeOrigin = RuntimeOrigin;
    type RuntimeCall = RuntimeCall;
    type RuntimeEvent = RuntimeEvent;
    type AccountId = u64;
    type Lookup = IdentityLookup<u64>;
    type Index = u64;
    type BlockNumber = u64;
    type Hash = H256;
    type Hashing = BlakeTwo256;
    type Header = Header;
    type RuntimeTask = ();
    type RuntimeVersion = ();
    type PalletInfo = PalletInfo;
    type AccountData = ();
    type OnNewAccount = ();
    type OnKilledAccount = ();
    type SystemWeightInfo = ();
    type SS58Prefix = ();
    type OnSetCode = ();
    type MaxConsumers = frame_support::traits::ConstU32<16>;
}

impl Config for Test {
    type RuntimeEvent = RuntimeEvent;
    type ModelHash = H256;
    type MaxInstNameLen = AuditMaxInstNameLen;
    type MaxNoteLen = AuditMaxNoteLen;
    type MaxCidLen = AuditMaxCidLen;
    type MaxPerAccount = AuditMaxPerAccount;
    type MaxPerInstitution = AuditMaxPerInstitution;
    // Admin = Root di test; pakai EnsureRoot<u64>
    type AuthorityOrigin = frame_system::EnsureRoot<u64>;
    type WeightInfo = ();
}

fn new_test_ext() -> sp_io::TestExternalities {
    let mut t = frame_system::GenesisConfig::<Test>::default().build_storage().unwrap();
    t.into()
}

// Helpers
fn inst(name: &str) -> BoundedVec<u8, AuditMaxInstNameLen> {
    BoundedVec::try_from(name.as_bytes().to_vec()).unwrap()
}

#[test]
fn submit_tracks_hash_and_indexes() {
    new_test_ext().execute_with(|| {
        // Root authorize account 1 (BOB) -> "DINSOS-KAB"
        assert_ok!(Audit::force_authorize(
            RuntimeOrigin::root(), 1u64, inst("DINSOS-KAB")
        ));

        let h = H256::repeat_byte(1);
        // BOB submit
        assert_ok!(Audit::submit(
            RuntimeOrigin::signed(1u64), h, None, None
        ));

        // ByHash -> Some(id)
        let id = Audit::by_hash(h).expect("id must exist");
        // Record lengkap
        let rec = Audit::records(id).expect("record exist");
        assert_eq!(rec.who, 1u64);
        assert_eq!(&rec.institution[..], b"DINSOS-KAB");

        // ByAccount & ByInstitution konsisten
        assert!(Audit::by_account(1u64).contains(&id));
        assert!(Audit::by_institution(inst("DINSOS-KAB")).contains(&id));
    });
}

#[test]
fn duplicate_hash_is_rejected() {
    new_test_ext().execute_with(|| {
        assert_ok!(Audit::force_authorize(RuntimeOrigin::root(), 1u64, inst("X")));
        let h = H256::repeat_byte(7);
        assert_ok!(Audit::submit(RuntimeOrigin::signed(1u64), h, None, None));
        assert_noop!(
            Audit::submit(RuntimeOrigin::signed(1u64), h, None, None),
            Error::<Test>::AlreadyExists
        );
    });
}

#[test]
fn non_authorized_cannot_submit() {
    new_test_ext().execute_with(|| {
        let h = H256::repeat_byte(9);
        assert_noop!(
            Audit::submit(RuntimeOrigin::signed(2u64), h, None, None),
            Error::<Test>::NotAuthorized
        );
    });
}
```

> Jalankan: `cargo test -p pallet-audit`
> Hasil tiga test di atas harus **PASS**.

---

# E) Off-chain Verification (opsional tapi kuat)

* Hitung SHA-256 lokal pada file artefak; bandingkan dengan `model_hash` di chain (`records(id).model_hash`).
* Jika kamu pakai IPFS: ambil file via `ipfs cat <CID> > file.bin` lalu `sha256sum file.bin` dan cocokkan.

---

# F) Traceability Matrix (singkat)

| Requirement      | Test                             | Hasil yang diharapkan                                              |
| ---------------- | -------------------------------- | ------------------------------------------------------------------ |
| Unik per-hash    | `duplicate_hash_is_rejected`     | Error `AlreadyExists`                                              |
| Akses terkontrol | `non_authorized_cannot_submit`   | Error `NotAuthorized`                                              |
| Index konsisten  | `submit_tracks_hash_and_indexes` | `ByHash/ByAccount/ByInstitution` berisi `id`; `records(id)` akurat |

---

