# Complete History Safe Rebuild Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Preserve every recoverable Codex/Claude session, current managed Vault note, and locally or remotely reachable managed Git version while rebuilding a privacy-safe AI history on one clean Git root, with authenticated backups, conflict-safe activation, registered-client receipts, and a proven Codex → Claude → Codex loop.

**Architecture:** Keep the existing runtime collector, eight-stage receipt chain, and single Obsidian publisher. Add versioned migration components around them: immutable source snapshots and a two-layer coverage ledger, Keychain-backed private routes, exhaustive privacy evidence, authenticated audit/backup roots, deterministic history artifacts, clean-root Git/LFS auditing, lease-bound remote switching, and a fixed activation transaction. `ai_session_sync_migrate.py` remains the v1 compatibility coordinator and becomes the v2 orchestrator; no production mutation occurs during implementation or readiness auditing.

**Tech Stack:** Python 3.9 standard library, frozen dataclasses, canonical JSON, `unittest`, Git plumbing, Git LFS, macOS Security and CryptoKit through a compiled Swift helper, Bash, launchd, Obsidian Markdown, and disposable local Git remotes in tests.

## Global Constraints

- Implement `docs/superpowers/specs/2026-07-22-complete-history-safe-rebuild-design.md` exactly. The specification wins if wording differs.
- Work only in the linked worktree on branch `codex/global-ai-handoff-loop`. Never modify the live Vault, platform sessions, launchd jobs, Keychain, real remotes, or production state during development/tests.
- Tests use only synthetic sessions, identities, credentials, financial values, paths, Vaults, Git graphs, LFS bodies, providers, and clients. Test diagnostics remain content-free.
- Keep `/usr/bin/python3` 3.9 compatibility and add no third-party Python runtime package. Compile `ai_session_sync_private_crypto.swift` for Keychain/AES-GCM operations; tests inject in-memory providers.
- Preserve the one-way privacy boundary: frozen raw source → adapter → S0–S3 sanitizer → independent audit → deterministic classification → safe archive/reference. No downstream API accepts a raw-message type or raw source bytes.
- Raw bytes may exist only in an open source descriptor, a streaming Git object pipe, or an authenticated `0700` audit/backup root. They never enter the staging Vault, clean repository, system temp, ordinary state, logs, receipts, or Hermes.
- Ordinary state and Vault contain safe IDs/HMAC references only. Exact paths, source mappings, real remote URLs, complete ref names, and client routes exist only in AES-256-GCM private routes; credentials remain in the system credential store.
- `com.harryd317.screener-vault-sync` remains the only continuous Git publisher. Migration may perform one lease-bound update while holding the same publication lock; it never installs a second publisher.
- `--since` is preview-only. Only a non-dry `stage --all-history` whose full hard gates pass can produce an activation manifest and one-time confirmation token.
- V1 manifest authentication/restore remains supported by a frozen compatibility path and fixed fixtures. Unknown versions fail closed; v1 activation does not inherit v2 claims.
- Do not touch `screener.py`, `rules_mid.py`, `params.py`, `storage.py`, or `service.py`; do not alter trading databases, services, schedules, or parameters.
- Every write has no-follow path/owner/type validation, private modes, atomic replacement, file and parent `fsync`, and fixed error categories. Append-only ledgers retain locking/flush/`fsync` semantics.
- Production activation, real force-with-lease, credential rotation, client reset, and real-data disaster recovery are not authorized by this implementation plan. The final readiness task stops before those actions.

## Per-Task TDD and Review Gate

Every task follows this exact sequence:

1. A fresh implementer records the task base SHA and writes the specified failing tests.
2. The implementer runs the focused command and records the expected RED cause; a test that passes before implementation is corrected before coding.
3. The implementer adds the minimum complete implementation, obtains focused GREEN, runs Shared Verification, reviews the owned diff, and makes the task-scoped Chinese commit named below.
4. A fresh reviewer reads the specification, task brief, implementation report, and `BASE..HEAD`. The reviewer separately reports specification compliance and code quality.
5. Any Critical or Important finding returns to a named remediation step/commit in the owning task. The same task is re-reviewed until Critical=0 and Important=0. Only then may progress advance.

Integration and whole-branch reviews are read-only. They may open named remediation tasks but may not mix code fixes into a later documentation commit.

## Shared Verification

After each code/shell task, run the focused tests plus:

```bash
/usr/bin/python3 -m unittest -q test_ai_session_sync.py test_ai_session_sync_sources.py test_ai_session_sync_privacy.py test_ai_session_sync_projects.py test_ai_session_sync_classify.py test_ai_session_sync_archive.py test_ai_session_sync_handoff.py test_ai_session_sync_ledger.py test_ai_session_sync_migrate.py test_ai_session_sync_e2e.py test_ai_session_sync_publish_barrier.py test_ai_session_sync_migration_shadow.py test_obsidian_sync.py
.venv/bin/python test_rules.py
.venv/bin/python test_rules_mid.py
.venv/bin/python test_web.py
.venv/bin/python test_regime.py
/usr/bin/python3 -m py_compile ai_session_sync*.py obsidian_sync.py obsidian_sync_core.py obsidian_sync_digest.py
/bin/bash -n install_ai_session_sync.sh install_obsidian_sync.sh
git diff --check
git diff --exit-code b0e702d -- screener.py rules_mid.py params.py storage.py service.py
```

Add every newly created focused module to the first command from the task where it appears. The final exact aggregate command is in Task 19.

## Frozen Cross-Task Contracts

Task 1 owns strict parsers and canonical serialization for:

- `FrozenIdentity`, `FileLedgerRow`, `RecordLedgerRow`, `CoverageReport`;
- `PrivacyEvidenceManifest` with per-surface S0/S1 scan evidence and exhaustive structured S2/S3 proof digests;
- frozen `BackupVerificationEvidence` containing version, primary/recovery backup UUIDs, both authenticated manifest digests, both restore-drill digests, and the HMAC source ref of the covered legacy key; `BackupVerificationPort.verify_legacy_key(source_ref)` either returns that exact evidence or fails closed;
- fixed `AuditArtifactState`: `prepared`, `populated`, `registered_for_backup`, `backed_up`, `cleanup_pending`, `removed`, `cleanup_failed`;
- deterministic `RebuildArtifactsManifest` containing only normalized input, policy, coverage, safe managed-output, ordinary-state, and provenance hashes;
- deterministic final `RebuildManifest`, assembled only after Git/LFS work, adding clean tree/root, candidate-object audit, LFS evidence, remote-source inventory, and audited safe reuse;
- random/authenticated `ActivationManifest`, with strict fields `private_route_ciphertext_sha256`, `private_route_schema_version`, `private_route_keychain_label`, and `private_route_allowed_fields_digest`, plus final rebuild hash, safe route ID, URL/ref HMACs, old/new OIDs, remote-ref digest, backup/audit roots, unsupported digest/acceptance, recovery-key label, token hash, and transaction journal;
- fixed `TransactionPhase`: `validated`, `locked`, `snapshotted`, `restore_tested`, `local_switched`, `remote_switched`, `installed_disabled`, `smoked`, `committed`, `services_started`;
- fixed terminal outcomes from specification §12.3.

Canonical JSON is UTF-8, sorted keys, no insignificant whitespace, and one trailing newline for files. `RebuildArtifactsManifest` and `RebuildManifest` reject time, paths, Keychain labels, ciphertext, nonces, token/transaction IDs, and remote plaintext. AES-GCM ciphertext is intentionally randomized and belongs only to `ActivationManifest`; nonce reuse is always rejected.

---

### Task 1: Freeze v2 schemas, deterministic/transactional manifests, and version dispatch

**Files:**
- Create: `ai_session_sync_migration_models.py`
- Create: `test_ai_session_sync_migration_models.py`
- Modify: `ai_session_sync_migrate.py`
- Modify: `test_ai_session_sync_migrate.py`

**Interfaces:** `canonical_json_bytes`; strict `parse_frozen_identity`, `parse_file_ledger_row`, `parse_record_ledger_row`, `parse_coverage_report`, `parse_privacy_evidence_manifest`, `parse_rebuild_artifacts_manifest`, `parse_rebuild_manifest`, `parse_activation_manifest`, `parse_backup_verification_evidence`, and `parse_transaction_journal`; `CoverageReport.validate`; `PrivacyEvidenceManifest.validate`; `RebuildArtifactsManifest.to_bytes`; `RebuildManifest.to_bytes`; `ActivationManifest.unsigned_bytes`; `BackupVerificationPort.verify_legacy_key`; `manifest_version`; and `dispatch_backup_manifest`.

- [ ] Write failing tests for exact fields/types, including the four named private-route activation fields and 64-hex `private_route_allowed_fields_digest`; closed source/status/reason/audit-artifact enums; nonnegative integer counts; source-kind-specific identities; file/record/byte equations; canonical stability; manifest separation; backup-verification evidence/port failure semantics; fixed phases/outcomes; unknown fields/versions; and v1/v2 dispatch.
- [ ] Run `/usr/bin/python3 -m unittest -q test_ai_session_sync_migration_models.py test_ai_session_sync_migrate.py`; expected RED is the missing model/dispatcher module.
- [ ] Implement frozen models and strict persisted-payload parsers. Reject booleans as integers, free-text reasons, raw paths/URLs/credentials, and misplaced random/private fields.
- [ ] Prove deterministic manifests are byte-identical for identical values while activation manifests may differ through independent nonce/time/token/private ciphertext and still authenticate.
- [ ] Run focused GREEN and Shared Verification; commit `迁移: 定义完整历史v2清单与版本边界`.
- [ ] Pass the Per-Task TDD and Review Gate.

---

### Task 2: Freeze all local sessions, current Vault files, and local reachable managed Git blobs

**Files:**
- Create: `ai_session_sync_migration_inventory.py`
- Create: `test_ai_session_sync_migration_inventory.py`
- Modify: `ai_session_sync_sources.py`
- Modify: `test_ai_session_sync_sources.py`
- Modify: `ai_session_sync_migration_models.py`

**Interfaces:** `discover_all_session_files`, `open_jsonl_snapshot`, `JsonlSnapshot.iter_complete_records`, `JsonlSnapshot.validate_append_only`, `snapshot_vault_file`, `iter_managed_git_blobs`, `build_local_inventory`, and `write_coverage_ledger`.

- [ ] Write failing fixtures for recent/old/archived/active/malformed/subagent/unknown-schema/incomplete-tail/invalid-UTF8/compressed/symlink/special-file cases; discovery must ignore mtime and inventory every `*.jsonl`.
- [ ] Test `O_NOFOLLOW` + `fstat` identity, last-complete-line high-water, `os.pread`, normal append/deferred bytes, partial tail, prefix rewrite, truncation, and inode replacement.
- [ ] Test current managed Vault Markdown/YAML/JSON/manual/binary cases and unique local managed blobs streamed with `git cat-file --batch`; repeated refs count without duplicate bodies.
- [ ] Test every §6.5 file/record/byte/source/platform/month/project equality; one missing or double-counted terminal record fails.
- [ ] Run `/usr/bin/python3 -m unittest -q test_ai_session_sync_sources.py test_ai_session_sync_migration_inventory.py test_ai_session_sync_migration_models.py`; expected RED is missing all-history snapshot/ledger behavior.
- [ ] Implement domain-separated HMAC source/record refs, closed reasons, aggregate-only Vault coverage, canonical `0600` local detail, and no raw path/session ID in ordinary state.
- [ ] Run focused GREEN and Shared Verification; commit `迁移: 建立本机全历史冻结盘点与覆盖账本`.
- [ ] Pass the Per-Task TDD and Review Gate.

---

### Task 3: Add Keychain/CryptoKit private routes and migrate runtime pseudonym keys

**Files:**
- Create: `ai_session_sync_migration_private.py`
- Create: `ai_session_sync_private_crypto.swift`
- Create: `test_ai_session_sync_migration_private.py`
- Modify: `ai_session_sync_core.py`
- Modify: `ai_session_sync_models.py`
- Modify: `ai_session_sync_projects.py`
- Modify: `test_ai_session_sync.py`
- Modify: `test_ai_session_sync_projects.py`
- Modify: `install_ai_session_sync.sh`

**Interfaces:** `SecretProvider.get_or_create`, `SecretProvider.put_if_absent`, `MacOSSecurityProvider`, `AeadProvider.seal/open`, `PrivateRouteStore.put/get`, `PrivateRouteStore.allowed_field_digest`, `keyed_route_fingerprint`, `PseudonymKeyProvider`, and `migrate_legacy_pseudonym_key(legacy_key: bytes, backup_evidence: BackupVerificationPort)`.

- [ ] Write failing in-memory-provider tests for AES-256-GCM authentication/AAD, randomized nonces, nonce reuse, wrong key/tag/AAD, strict route schemas/field selection, `0600` atomic store, symlink boundaries, and absence of plaintext/credentials.
- [ ] Compile the Swift helper in a disposable directory when `swiftc` exists. Its bounded JSON protocol uses stdin/stdout and Security/CryptoKit; captured argv/env/stderr contain no key/plaintext.
- [ ] Test public project registry projection: only safe project IDs, display names, root aliases, and pseudonymized remote identities persist; exact roots/workdirs/remotes/refs/client routes round-trip only through encrypted routes.
- [ ] Test the pure migration operation accepts only injected authenticated `BackupVerificationPort` evidence, imports the legacy `pseudonym.key` into Keychain label `ai-session-sync/pseudonym/v1`, returns only label/fingerprint, and prepares a new state without the file. Task 14 invokes it only after Task 8/9 backup evidence. V1 runtime/restore remains compatible; first v2 collection refuses plaintext-file fallback and cannot recreate the file.
- [ ] Run `/usr/bin/python3 -m unittest -q test_ai_session_sync_migration_private.py test_ai_session_sync.py test_ai_session_sync_projects.py`; expected RED is missing provider/store/runtime migration behavior.
- [ ] Implement helper/provider/store/public-registry/runtime-key contracts. `--dry-run` and tests never touch real Keychain.
- [ ] Run focused GREEN and Shared Verification; commit `安全: 建立私密路由并迁移运行期密钥`.
- [ ] Pass the Per-Task TDD and Review Gate.

---

### Task 4: Produce exhaustive S0–S3 evidence, safe session IDs, and a central Hermes outlet

**Files:**
- Create: `test_ai_session_sync_hermes.py`
- Modify: `ai_session_sync_privacy.py`
- Modify: `ai_session_sync_audit.py`
- Modify: `ai_session_sync_hermes.py`
- Modify: `ai_session_sync_models.py`
- Modify: `ai_session_sync_archive.py`
- Modify: `ai_session_sync_classify.py`
- Modify: `ai_session_sync_digest.py`
- Modify: `ai_session_sync_handoff.py`
- Modify: `obsidian_sync_digest.py`
- Modify: `test_ai_session_sync_privacy.py`
- Modify: `test_ai_session_sync_archive.py`
- Modify: `test_ai_session_sync_classify.py`
- Modify: `test_ai_session_sync_handoff.py`
- Modify: `test_obsidian_sync.py`

**Interfaces:** `safe_identity_alias`, `sanitize_migration_record`, `PrivacyProofBuilder`, `write_quarantine_reference`, immutable `AuditedHermesPrompt`, `audit_hermes_prompt`, and `send_audited_hermes_prompt`.

- [ ] Write failing tests that enumerate every structured S2/S3 field. S2 keeps type/result/before-after keyed HMAC only and rejects exact or denominator-recoverable values. S3 normalizes number type/Unicode/newlines and requires equality. Unstructured samples use a seed derived from canonical input and explicitly avoid semantic-equality claims.
- [ ] Test independent S0/S1 scans and metadata-only quarantine: safe source/record refs, category counts, policy version; no `RawMessage`, path, platform session ID, body, or finding fragment.
- [ ] Test `SafeMessage.session_id`, archive filenames/frontmatter, read receipts, ledgers, handoffs, and status use stable purpose-separated aliases after the privacy boundary.
- [ ] Test the one Hermes outlet: callers must pass `AuditedHermesPrompt`, the outlet independently scans immediately before request construction, and evidence contains only request hash/policy/result. Raw strings and tampered prompts fail; prompt/finding text never enters logs/receipts. Refactor every Hermes entry point, including active `obsidian_sync_digest.request_digest`, through it.
- [ ] Add an import/AST inventory plus runtime interception test proving every shipped `urllib`/HTTP Hermes request originates inside `send_audited_hermes_prompt`; publisher failure logging must not expose request or finding text.
- [ ] Run `/usr/bin/python3 -m unittest -q test_ai_session_sync_privacy.py test_ai_session_sync_hermes.py test_ai_session_sync_archive.py test_ai_session_sync_classify.py test_ai_session_sync_handoff.py test_obsidian_sync.py`; expected RED is missing proofs/aliases/outlet and the Obsidian digest bypass.
- [ ] Implement proof manifest rows, aliasing, v2 quarantine, and the mandatory outlet without weakening the independent audit implementation. Migration stage continues to make zero Hermes calls.
- [ ] Run focused GREEN and Shared Verification; commit `安全: 封闭迁移证明与Hermes最终出口`.
- [ ] Pass the Per-Task TDD and Review Gate.

---

### Task 5: Create recovery-root credentials and authenticated audit-root primitives

**Files:**
- Create: `ai_session_sync_migration_backup.py`
- Create: `test_ai_session_sync_migration_backup.py`
- Modify: `ai_session_sync_migration_private.py`
- Modify: `ai_session_sync_migration_models.py`

**Interfaces:** `encode_recovery_code`, `decode_recovery_code`, `derive_backup_manifest_key`, `seal_recovery_secret`, `open_recovery_secret`, `RecoveryCredentialProvider`, `AuthenticatedRoot.create/open/verify`, and `AuditArtifactRegistry.prepare/populate/register_for_backup/mark_backed_up/remove/resume_cleanup`.

- [ ] Write failing fixed-key tests for exact `AIMR2-Base32-10-character-checksum`, version/checksum errors, HKDF-SHA-256 domain separation by backup UUID/purpose, wrong key, empty-Keychain offline recovery, and one-time recovery-code display evidence.
- [ ] Test recovery-only escrow for the independent route and pseudonym/HMAC keys: each value is AES-GCM sealed once under a recovery-root-derived purpose with AAD containing escrow schema, migration transaction ID, source-snapshot digest, and key label. It is never plaintext or accepted as the recovery root key and cannot be swapped across labels or migrations. The identical authenticated escrow bytes may be copied from primary to recovery backup; each backup-specific manifest separately binds their hash. Git authentication credentials are not exported.
- [ ] Test independent route/recovery keys, `0700` roots, `0600` manifests/files, HMAC + SHA-256 verification, no system-temp roots, no symlink/special-file boundary, and safe cleanup/residual scan.
- [ ] Test the fixed audit-artifact states and authenticated transitions. Any raw artifact that is not registered for both backups or completely removed with a residual scan blocks readiness; interrupted cleanup becomes `cleanup_failed` and is resumable only with the bound recovery credential.
- [ ] Run `/usr/bin/python3 -m unittest -q test_ai_session_sync_migration_backup.py test_ai_session_sync_migration_private.py`; expected RED is missing recovery/audit-root primitives.
- [ ] Implement the shared authenticated-root foundation used by remote-object, LFS, backup, conflict, and drill roots. Recovery code is never accepted in argv/env or ordinary logs.
- [ ] Run focused GREEN and Shared Verification; commit `恢复: 建立恢复密钥与认证审计根`.
- [ ] Pass the Per-Task TDD and Review Gate.

---

### Task 6: Inventory all remote refs securely and merge remote-only managed history

**Files:**
- Create: `ai_session_sync_git_transport.py`
- Create: `test_ai_session_sync_git_transport.py`
- Modify: `ai_session_sync_migration_inventory.py`
- Modify: `ai_session_sync_migration_private.py`
- Modify: `install_ai_session_sync.sh`

**Interfaces:** `GitRouteTransport.run`, `inventory_remote_refs`, `fetch_remote_object_inventory`, `RemoteObjectInventory.iter_managed_blobs`, and `merge_local_remote_inventory`.

- [ ] Write failing transport tests: checked-in Git config contains `ai-vault::safe-route-id`; the wrapper decrypts URL from private routes, obtains auth from the credential store, supplies an anonymous inherited config FD, and never places either in argv/env/disk config/log/stdout/stderr.
- [ ] Use a local bare remote with heads/tags/notes/custom namespaces and remote-only commits. Enumerate every `ls-remote` ref, store exact ref mapping only encrypted, fetch objects into an authenticated audit root, and stream unique managed blobs into the coverage ledger.
- [ ] Make the remote-object root owner explicit: plan/dry-run roots must reach `removed`; a successful non-dry stage reaches `registered_for_backup` and is bound in its activation evidence. Stage failure attempts authenticated removal; interrupted removal remains `cleanup_failed`, emits only safe recovery metadata, and cannot produce an activation token.
- [ ] Add a remote-only safe managed blob fixture and require it to be imported later; add duplicate local/remote refs and count references without duplicate bodies. Unsafe extra-ref blocking is tested separately in Task 12.
- [ ] Run `/usr/bin/python3 -m unittest -q test_ai_session_sync_git_transport.py test_ai_session_sync_migration_inventory.py`; expected RED is missing secure transport/remote inventory merge.
- [ ] Implement remote-source freezing and extend coverage equations so “all reachable managed Git history” means merged local and enumerated remote refs, never just local refs.
- [ ] Run focused GREEN and Shared Verification; commit `迁移: 纳入远端全部引用与独有历史`.
- [ ] Pass the Per-Task TDD and Review Gate.

---

### Task 7: Deterministically rebuild safe history artifacts

**Files:**
- Create: `ai_session_sync_migration_rebuild.py`
- Create: `test_ai_session_sync_migration_rebuild.py`
- Modify: `ai_session_sync_archive.py`
- Modify: `ai_session_sync_digest.py`
- Modify: `ai_session_sync_handoff.py`
- Modify: `test_ai_session_sync_archive.py`
- Modify: `test_ai_session_sync_handoff.py`

**Interfaces:** `rebuild_history`, `render_historical_reference`, `diff_managed_note`, `rewrite_safe_links`, `merge_historical_sources`, `RebuildArtifactsManifest`, and `verify_rebuild_artifact_idempotence`.

- [ ] Write failing fixtures for duplicate session/current-Vault/local-Git/remote-only-Git facts, manual additions, generated/manual ambiguity, safe/unsafe/missing internal links, all six categories, and unsafe/unreadable gaps.
- [ ] Require one sanitized body per normalized fingerprint with all sources, stable safe IDs, exact Vault layout, `historical_only: true`, safe times/policy, `migration_temporary` fallback, and explicit unavailable-link markers.
- [ ] Test authority: newer explicit user decision → newer visible main session → current Vault → Git reference. Unresolved conflicts show both anchors and `待人工拍板`; old summaries never become current constraints automatically.
- [ ] Run identical frozen merged inputs twice. Managed files, ordinary state, aggregate coverage, provenance, privacy-evidence digest, and `RebuildArtifactsManifest` are byte-identical; second pass has zero Hermes calls/writes/new receipts.
- [ ] Run `/usr/bin/python3 -m unittest -q test_ai_session_sync_migration_rebuild.py test_ai_session_sync_archive.py test_ai_session_sync_handoff.py`; expected RED is missing adapters/legacy rendering/manual diff/link rewriting/artifact manifest.
- [ ] Implement only safe history outputs. The artifact manifest excludes candidate Git/LFS fields and all private ciphertext; those are Task 11 inputs.
- [ ] Run focused GREEN and Shared Verification; commit `迁移: 完成全历史确定性脱敏重建`.
- [ ] Pass the Per-Task TDD and Review Gate.

---

### Task 8: Create byte-complete primary and recovery backups

**Files:**
- Modify: `ai_session_sync_migration_backup.py`
- Create: `test_ai_session_sync_migration_backup_snapshot.py`
- Modify: `ai_session_sync_migrate.py`

**Interfaces:** `discover_git_layout`, `capture_source_manifest`, `prepare_primary_backup`, `clone_recovery_backup`, `include_registered_audit_artifacts`, and `verify_backup_content`.

- [ ] Create synthetic normal/linked-worktree/separate-git-dir/shallow/packed-ref/reflog/hook/config/index/LFS/additional-worktree repositories. Discover actual common/worktree gitdirs through Git, not `.git` assumptions.
- [ ] Require frozen session prefixes; complete Vault filesystem including nonmanaged/untracked and `.git` file/dir; actual common/worktree metadata, index/config/hooks/logs/objects/refs/packed-refs/shallow/worktrees/LFS; bundle `--all`; AI state/instructions/Claude settings/jobs/runtime/modes/journal.
- [ ] Backup session bytes only through the activation-held `JsonlSnapshot` high-water descriptors, then revalidate device/inode/prefix; derive recovery copies only from authenticated primary content. Include every `registered_for_backup` remote-object/LFS audit root, its state journal, and complete authenticated manifest in both backup copies before marking it `backed_up`.
- [ ] Include the same authenticated recovery-only escrow envelopes for route and pseudonym/HMAC keys in both backups so empty-Keychain restore can recreate exact labels. Define an identical `snapshot_content_manifest` copied from primary to recovery; each copy then has a different backup UUID/salt/HMAC wrapper that binds the same content-manifest digest. Equality checks compare snapshot content, while per-copy authentication metadata is intentionally distinct. External Git authentication credentials remain outside.
- [ ] Test copy fallback with pre-source → one source read → primary → post-source equality, including “changed during copy then restored”; any drift/special file/symlink/owner change aborts before live mutation.
- [ ] Require the recovery copy to derive only from authenticated primary content and have an equal content manifest. Exclude recursive migration-backups while recording prior backup integrity references.
- [ ] Run `/usr/bin/python3 -m unittest -q test_ai_session_sync_migration_backup.py test_ai_session_sync_migration_backup_snapshot.py test_ai_session_sync_migrate.py`; expected RED is incomplete v1 target coverage.
- [ ] Implement full durable snapshots with `0700/0600`, SHA-256, HMAC, modes, `fsync`, and no raw path/body in ordinary output.
- [ ] Run focused GREEN and Shared Verification; commit `恢复: 建立完整双份封存备份`.
- [ ] Pass the Per-Task TDD and Review Gate.

---

### Task 9: Verify independent restore drills and v1/v2 compatibility

**Files:**
- Create: `ai_session_sync_migration_restore.py`
- Create: `test_ai_session_sync_migration_restore.py`
- Create: `testdata/migration/v1/manifest.json`
- Create: `testdata/migration/v2/manifest.json`
- Modify: `ai_session_sync_migrate.py`

**Interfaces:** `verify_backup`, `drill_restore`, `verify_git_metadata_restore`, `build_backup_verification_evidence`, `restore_manifest_v1`, `restore_manifest_v2`, and `dispatch_restore`.

- [ ] Write failing tests that restore each backup independently in an authenticated drill root; verify HMAC/SHA, bundle clone + `git fsck --full`, actual Git metadata, HEAD/all refs/reflog/index/config/hooks/LFS/object set/worktree status, state/jobs/instructions, and seeded byte samples.
- [ ] Simulate a fresh process with persistent Keychain credential and an empty-Keychain process using only the offline code. Wrong version/checksum/key and unknown version fail.
- [ ] In the empty-Keychain drill, use the offline recovery root to authenticate the backup, open the bound escrow envelopes, recreate route and pseudonym/HMAC Keychain labels, decrypt the private route store, and complete the same local restore. Never use either recovered subkey as the backup HMAC root; remote authentication is reported as requiring credential-store reauthentication when absent.
- [ ] Freeze v1/v2 fixture hashes. Prove v1 uses only the v1 restorer, v2 only v2, and upgrades do not strand prior backups.
- [ ] Produce strict `BackupVerificationEvidence` only when both independently authenticated backups and both drills cover the HMAC legacy-key source ref and all required Git/Vault/state artifacts. Wrong UUID/digest/source-ref or one missing drill cannot satisfy `BackupVerificationPort`.
- [ ] Inject changes during the drill and cleanup failures; drill roots remain authenticated, residue is reported safely, and no live target changes.
- [ ] Run `/usr/bin/python3 -m unittest -q test_ai_session_sync_migration_restore.py test_ai_session_sync_migrate.py`; expected RED is missing full drills/versioned restore.
- [ ] Implement drills and version dispatch without adding disaster-recovery mutation; that is Task 15.
- [ ] Run focused GREEN and Shared Verification; commit `恢复: 验证双备份演练与版本兼容`.
- [ ] Pass the Per-Task TDD and Review Gate.

---

### Task 10: Build a single-parentless clean root and audit its full object graph

**Files:**
- Create: `ai_session_sync_migration_git.py`
- Create: `test_ai_session_sync_migration_git.py`
- Modify: `ai_session_sync_audit.py`

**Interfaces:** `build_clean_root`, `audit_candidate_tree`, `audit_reachable_objects`, `AuditedObjectReuse`, and `verify_clean_root`.

- [ ] Write failing repositories with safe/unsafe managed and nonmanaged files, modes, symlinks, submodules, unsafe commit metadata, reused safe blobs/trees, old commits, and untracked files.
- [ ] Require the locked source to have no tracked worktree/index changes, local `main` equal frozen `origin/main`, and a frozen ref set. Untracked files may remain only when they do not collide with candidate tracked paths; dirty/alignment/ref/collision cases fail before root construction.
- [ ] Reject every gitlink/submodule before root construction with closed gate reason `candidate_gitlink_unsupported` plus HMAC path ref. Do not fetch submodules, parse `.gitmodules` URLs, or claim a complete candidate while a gitlink exists.
- [ ] Require exactly one parentless `main`, byte/mode equality for every locked nonmanaged tracked file, substitution only under two managed prefixes, exclusion/preservation of untracked live files, and sanitized fixed commit metadata.
- [ ] Independently scan the complete candidate tree, commit metadata, and every reachable object. S0/S1 reports contain only HMAC path/object ref + category/count. No old commit, unsafe managed tree/blob, or forbidden object is reachable.
- [ ] Record safe nonmanaged blob/tree reuse only after all descendants pass audit. Recognize LFS pointers and mark the candidate incomplete until Task 11 verifies bodies.
- [ ] Run `/usr/bin/python3 -m unittest -q test_ai_session_sync_migration_git.py`; expected RED is missing clean-root/full-object behavior.
- [ ] Implement in a new private object database without copying source refs/commits/reflogs/config/alternates/packs. Raw Git blobs remain streamed.
- [ ] Run focused GREEN and Shared Verification; commit `迁移: 构建无父干净Git根并审计对象图`.
- [ ] Pass the Per-Task TDD and Review Gate.

---

### Task 11: Audit actual LFS bodies and assemble the final deterministic rebuild manifest

**Files:**
- Create: `ai_session_sync_lfs_provider.py`
- Create: `test_ai_session_sync_migration_lfs.py`
- Modify: `ai_session_sync_migration_git.py`
- Modify: `ai_session_sync_git_transport.py`
- Modify: `ai_session_sync_migration_models.py`
- Modify: `ai_session_sync_migration_rebuild.py`

**Interfaces:** `LfsBodyProvider.preflight/fetch`, `LocalLfsProvider`, `HttpBatchLfsProvider`, `SshAuthenticateLfsProvider`, optional `ExternalGitLfsProvider`, `select_lfs_provider`, `parse_lfs_pointer`, `stream_lfs_body`, `audit_lfs_objects`, and `assemble_rebuild_manifest`.

- [ ] Test valid/duplicate/malformed pointers, wrong OID/declared size, missing local/remote body, chunk-boundary secrets, body S0/S1, interrupted reads, authenticated cache cleanup, and residual scans.
- [ ] Exercise a dedicated LFS custom-transfer-agent protocol separately from Git transport. Real URL/auth remain private and absent from argv/env/config/log/output.
- [ ] Do not assume `git-lfs` exists. Test this host's missing-binary case, unsupported external versions, provider errors, and deterministic provider selection. The standard-library local + HTTP batch provider is the default; SSH uses a safe host alias and `git-lfs-authenticate` response without exposing route/auth. If a required remote has no supported provider, return closed reason `lfs_provider_unavailable` before readiness. Never auto-install software during plan/stage.
- [ ] Require safe pointer ref, LFS OID, declared size, actual body SHA-256, and scan result in evidence; never body or matched text. Missing/mismatch/unscannable/unsafe blocks readiness.
- [ ] Assemble final `RebuildManifest` from `RebuildArtifactsManifest`, clean tree/root, candidate object audit, LFS evidence, merged remote-source inventory, and audited reuse. Identical inputs are byte-identical; ciphertext/Keychain/nonce/time/token remain excluded.
- [ ] Run `/usr/bin/python3 -m unittest -q test_ai_session_sync_migration_lfs.py test_ai_session_sync_migration_git.py test_ai_session_sync_git_transport.py test_ai_session_sync_migration_rebuild.py`; expected RED is missing provider preflight, actual-body audit, and final assembly.
- [ ] Implement streaming/canonical assembly and compare two complete rebuild passes with zero Hermes and zero changed deterministic bytes.
- [ ] Run focused GREEN and Shared Verification; commit `迁移: 完成LFS正文审计与确定性重建清单`.
- [ ] Pass the Per-Task TDD and Review Gate.

---

### Task 12: Enforce all-ref reachability, forward/reverse leases, and remote readback

**Files:**
- Create: `test_ai_session_sync_migration_remote.py`
- Modify: `ai_session_sync_migration_git.py`
- Modify: `ai_session_sync_git_transport.py`

**Interfaces:** `assert_no_extra_ref_reaches_managed_history`, `freeze_remote_lease`, `lease_update_main`, `lease_restore_main`, and `read_back_remote`.

- [ ] Use local bare remotes with heads/tags/notes/custom refs. Block when any non-target ref reaches old managed history; do not delete it automatically. Report provider-hidden retention as residual risk, never purge proof.
- [ ] Test exact URL/ref keyed HMAC, old/new OID and full refs digest binding; concurrent advancement, refs drift, network failure, and readback mismatch. Forward update is equivalent to `--force-with-lease=refs/heads/main:old_oid`; reverse uses `clean_oid`.
- [ ] Inspect every executed command: no unconditional force, plaintext route, auth, or complete ref map appears. Successful readback recomputes URL HMAC, target OID, refs digest, and candidate tree.
- [ ] Run `/usr/bin/python3 -m unittest -q test_ai_session_sync_migration_remote.py test_ai_session_sync_git_transport.py test_ai_session_sync_migration_git.py`; expected RED is missing blocker/lease/readback.
- [ ] Implement typed success/conflict/network outcomes without performing any real remote operation in tests.
- [ ] Run focused GREEN and Shared Verification; commit `迁移: 加入远端全引用与双向租约硬门`.
- [ ] Pass the Per-Task TDD and Review Gate.

---

### Task 13: Track registered desktop/mobile/publisher clients and reconnect receipts

**Files:**
- Create: `ai_session_sync_migration_clients.py`
- Create: `test_ai_session_sync_migration_clients.py`
- Modify: `ai_session_sync_migration_models.py`
- Modify: `ai_session_sync_ledger.py`

**Interfaces:** `ClientRegistry.register`, `ClientRegistry.expected`, `ClientRegistry.record_receipt`, `ClientRegistry.status`, and `render_client_status`.

- [ ] Write failing tests for stable safe client IDs, encrypted private routes, expected ref/OID, idempotent signed receipts, stale/wrong-device/wrong-OID receipts, desktop/mobile/publisher types, and corrupt registry recovery.
- [ ] Status is complete only for registered clients with matching receipts. Unregistered devices never count; wording distinguishes remote switched, partial reconnect, desktop complete/mobile pending, and complete registered set without saying “all devices”.
- [ ] Run `/usr/bin/python3 -m unittest -q test_ai_session_sync_migration_clients.py test_ai_session_sync_ledger.py`; expected RED is missing client registry/receipts.
- [ ] Implement separate migration client receipts; do not overload the eight-stage runtime `Receipt` or publish raw device names/routes.
- [ ] Run focused GREEN and Shared Verification; commit `迁移: 建立客户端注册与重连回执`.
- [ ] Pass the Per-Task TDD and Review Gate.

---

### Task 14: Implement the fixed activation transaction and conflict-safe transaction rollback

**Files:**
- Create: `ai_session_sync_migration_transaction.py`
- Create: `test_ai_session_sync_migration_transaction.py`
- Modify: `ai_session_sync_migrate.py`
- Modify: `ai_session_sync_migration_private.py`
- Modify: `ai_session_sync_core.py`
- Modify: `obsidian_sync.py`
- Modify: `obsidian_sync_core.py`
- Modify: `test_ai_session_sync_migrate.py`
- Modify: `test_ai_session_sync_migration_private.py`
- Modify: `test_ai_session_sync.py`
- Modify: `test_obsidian_sync.py`

**Interfaces:** `verify_activation_gate`, `capture_live_fingerprints`, `ActivationTransaction.run`, `ActivationTransaction.rollback`, `restore_if_owned`, and `PublisherControl`.

- [ ] Write the complete failure-injection matrix for all ten phases and only allowed §12.3 outcomes.
- [ ] Bind/retest source snapshots, append observations, coverage/unsupported acceptance, policies, both deterministic manifests, all four named private-route activation fields, safe route/URL/ref HMAC, Git layout/tree/object/LFS/remote evidence, old/new OIDs, refs digest, backup/drill credentials, publisher lock, token, and `PrivacyEvidenceManifest`.
- [ ] Treat the private-route envelope as its own activation surface: decrypt with the bound key label/AAD, authenticate the tag, rerun the exact route-schema/selected-field allowlist, reject credentials/free text/unknown fields/schema downgrade, and compare the recomputed canonical allowed-field digest to the value bound in `ActivationManifest`. Test wrong key/tag/AAD, missing envelope fields, validly encrypted forbidden fields, digest mismatch, and ciphertext/key-label substitution.
- [ ] Add one rejection test for each required activation surface: managed Vault, ordinary state, Hermes request evidence, receipts, logs, commit metadata, every reachable candidate object, and actual LFS bodies. Each surface must have bound hash, independent S0/S1 result, exhaustive S2 proof, and exhaustive S3 proof where structured. Missing/tampered proof digest blocks.
- [ ] Recompute live fingerprints before backup and immediately before mutation. Only authenticated high-water appends are allowed; backup/drill-time Vault/index/commit/ref/object drift returns `aborted_no_live_change`.
- [ ] After both backups and drills produce matching `BackupVerificationEvidence`, journal the Keychain label's old fingerprint, call `migrate_legacy_pseudonym_key`, journal the installed fingerprint, and include it in the local-switch ownership set. On rollback, remove/restore only when current equals installed; if current is a third value, leave it intact, write only its safe fingerprint to the authenticated conflict record, and return `rollback_local_conflict`.
- [ ] Before any local switch, require every raw remote-object/LFS root to be `backed_up`, advance it through `cleanup_pending` to `removed`, and pass a residual scan. Interrupted cleanup or unauthenticated/unregistered residue returns `aborted_no_live_change`. Rollback/remote-pending/committed states never recreate the working audit root; raw history remains only inside both backups.
- [ ] Test local old/installed/third-value cases. Restore only migration-installed values; preserve third values in authenticated conflict root and return `rollback_local_conflict`. Never restore whole Vault or overwrite nonmanaged content.
- [ ] Test reverse-lease success/conflict/network; conflict/pending states keep publisher disabled and never force unconditionally. After `committed`, collector/publisher start failure returns `activated_service_degraded`; publisher starts last.
- [ ] Run `/usr/bin/python3 -m unittest -q test_ai_session_sync_migration_transaction.py test_ai_session_sync_migration_private.py test_ai_session_sync.py test_ai_session_sync_migrate.py test_obsidian_sync.py`; expected RED is missing v2 state machine/gates/private-route revalidation/Keychain ownership/conflict rollback.
- [ ] Implement injected filesystem/launchd/Git/clock/nonce ports, durable journal-before-transition, and exact §12.1 order. Do not add disaster recovery here.
- [ ] Run focused GREEN and Shared Verification; commit `迁移: 完成固定阶段激活与冲突安全回滚`.
- [ ] Pass the Per-Task TDD and Review Gate.

---

### Task 15: Implement separately authorized disaster recovery

**Files:**
- Create: `ai_session_sync_migration_disaster.py`
- Create: `test_ai_session_sync_migration_disaster.py`
- Modify: `ai_session_sync_migration_restore.py`
- Modify: `ai_session_sync_migration_models.py`
- Modify: `ai_session_sync_migrate.py`

**Interfaces:** `prepare_disaster_challenge`, strict `DisasterRecoveryChallenge`, `consume_disaster_challenge`, `DisasterRecovery.run`, `capture_incident_snapshot`, and `verify_recovered_system`.

- [ ] Write RED tests for a separate one-time challenge bound to authenticated backup UUID/version/manifest digest, current target-root fingerprints, validated incident-snapshot destination, issued/expiry time, and random token hash. Activation tokens are always invalid; stale, mismatched, replayed, altered-target, and wrong-recovery-key challenges fail.
- [ ] The prepare operation writes an authenticated `0600` challenge document and sends the plaintext challenge only to an explicit no-log output FD/TTY. Recovery reads recovery code and confirmation from separate inherited FDs or sequential no-echo TTY prompts; neither value is accepted in argv/env/JSON/logs.
- [ ] Write RED tests before implementation for scoped confirmation, stopping every writer, authenticated third现场 snapshot before overwrite, refusal if snapshot fails, v1/v2 dispatch, and unknown-version rejection.
- [ ] Test full Vault + actual Git metadata + state/instructions/jobs/runtime restore and verification of HEAD/all refs/reflog/index/config/hooks/LFS/objects/worktree state. Inject crashes/failures at each destructive boundary and preserve incident evidence.
- [ ] Raw remote-object/LFS audit artifacts inside backups remain backup-only evidence; disaster recovery verifies them but never restores them into the live Vault, runtime state, clean repository, or ordinary audit workspace.
- [ ] Empty-Keychain disaster recovery restores only the separately escrowed route/pseudonym keys after backup/challenge authentication and journals their old/installed fingerprints; it never fabricates or exports Git authentication credentials.
- [ ] Prove `ActivationTransaction.rollback` has no import/call path to disaster recovery; transaction rollback cannot accept arbitrary backup roots or complete-Vault overwrite authority.
- [ ] Run `/usr/bin/python3 -m unittest -q test_ai_session_sync_migration_disaster.py test_ai_session_sync_migration_restore.py test_ai_session_sync_migration_transaction.py`; expected RED is missing separately authorized destructive restore.
- [ ] Implement with injected writer/snapshot/restore ports. Tests use disposable roots only; no user backup is restored during implementation.
- [ ] Run focused GREEN and Shared Verification; commit `恢复: 实现独立授权的完整灾难恢复`.
- [ ] Pass the Per-Task TDD and Review Gate.

---

### Task 16: Expose gate v2, unsupported acceptance, all-history shadow, and safe recovery input through CLI

**Files:**
- Modify: `ai_session_sync.py`
- Modify: `ai_session_sync_migrate.py`
- Modify: `ai_session_sync_migration_backup.py`
- Modify: `ai_session_sync_migration_disaster.py`
- Modify: `test_ai_session_sync_migrate.py`
- Modify: `test_ai_session_sync_migration_shadow.py`
- Modify: `test_ai_session_sync_migration_disaster.py`

**CLI contracts:**
- `migrate plan --all-history --audit-root ABS --audit-recovery-code-output-fd FD`; `plan --since YYYY-MM-DD` remains local bounded preview.
- `migrate stage --all-history --staging-vault SAFE_ABS --audit-root RAW_ABS --recovery-code-output-fd FD`; when unsupported exists, rerun with both `--accept-unsupported --unsupported-digest SHA256`.
- `migrate stage --all-history --staging-vault SAFE_ABS --audit-root RAW_ABS --audit-recovery-code-output-fd FD --dry-run` performs full two-pass shadow only.
- `migrate stage --since YYYY-MM-DD --staging-vault ABS --dry-run` remains bounded preview and can never activate.
- `migrate activate|rollback --manifest PATH --confirm TOKEN`.
- `migrate cleanup-stage --manifest PATH --recovery-code-fd FD` resumes authenticated removal of failed/abandoned staged raw audit artifacts and succeeds only at state `removed` with zero residue.
- `migrate disaster-prepare --backup PATH --target-root PATH --incident-root PATH --recovery-code-fd FD --challenge-output-fd FD`.
- `migrate disaster-recover --challenge PATH --recovery-code-fd FD --confirmation-fd FD`; interactive variants read the two values sequentially without echo from `/dev/tty`. Activation tokens and argv/env secret values are rejected.
- `migrate client register|receipt|status` exposes only safe client IDs publicly.

- [ ] Write parser/output RED tests for mutual exclusion, `gate_version: 2`, `inventory_complete`, `staging_allowed`, input risk/rotation, and `activation.ready/reason`. Raw input S0/S1 is risk, not a staging blocker; source/object/coverage faults are blockers.
- [ ] Define the plan-only workspace: the absolute root is prevalidated empty/non-live/`0700`; an ephemeral key authenticates it; its recovery code is emitted once to the supplied no-log FD before remote fetch. Success must remove it and pass a residual scan; failure preserves only an authenticated artifact plus safe state/reason. No ordinary state/Keychain/system-temp write or remote mutation occurs, and `inventory_complete` is false if remote fetch/audit is incomplete.
- [ ] Define two disjoint stage roots with bidirectional containment/symlink/live-boundary checks. `--staging-vault` contains sanitized outputs only; every fetched raw remote Git/LFS byte exists only under the authenticated `--audit-root`. Non-dry stage binds the persistent recovery credential/audit lifecycle into activation evidence; dry-run uses an ephemeral credential/output FD and must remove the raw root with zero residue. Neither mode may place raw bytes in the safe staging root or system temp.
- [ ] Test unsupported flow: first non-dry all-history stage may create safe stage/private evidence but emits no token and returns exact authoritative unsupported digest; missing acceptance, one-sided flags, mismatch, and stale digest reject. Exact flag+digest is bound into activation manifest/token. Preview/dry-run acceptance is non-authoritative and emits no token.
- [ ] Define all-history dry-run side effects: it uses only the two caller-supplied prevalidated empty disposable roots and ephemeral in-memory route/recovery keys, removes raw audit artifacts after hashing, and emits safe counts/hashes. It may read existing Git configuration, credential entries, and remote refs through the read-only transport, but it cannot create/update Keychain entries, mutate a remote, or retain plaintext. It touches no live state/Vault, backups, jobs, or Hermes and creates no activation manifest/token.
- [ ] Test recovery/challenge values absent from captured argv/env/stdout/stderr/log/error paths. Test challenge prepare/consume expiry, exact bindings, replay prevention, and activation-token rejection. One-time codes go only to selected TTY/FD and never JSON/logs.
- [ ] Test `cleanup-stage` for every audit-artifact state and interrupted cleanup; unauthenticated/unregistered residue remains a hard blocker.
- [ ] Run `/usr/bin/python3 -m unittest -q test_ai_session_sync_migrate.py test_ai_session_sync_migration_shadow.py test_ai_session_sync_migration_disaster.py`; expected RED is current `--since`/`safe_to_activate` CLI and missing workspace/challenge commands.
- [ ] Implement safe JSON/errors without printing `str(exc)` when it may contain content.
- [ ] Run focused GREEN and Shared Verification; commit `集成: 接入完整历史v2迁移命令`.
- [ ] Pass the Per-Task TDD and Review Gate.

---

### Task 17: Integrate installation, single-publisher control, and the first post-activation epoch

**Files:**
- Modify: `ai_session_sync.py`
- Modify: `ai_session_sync_classify.py`
- Modify: `install_ai_session_sync.sh`
- Modify: `install_obsidian_sync.sh`
- Modify: `obsidian_sync.py`
- Modify: `obsidian_sync_core.py`
- Modify: `ai_session_handoff.py`
- Modify: `ai_session_sync_ledger.py`
- Modify: `test_ai_session_sync_ledger.py`
- Modify: `test_ai_session_sync_classify.py`
- Modify: `test_ai_session_sync_publish_barrier.py`
- Modify: `test_ai_session_sync_e2e.py`
- Modify: `test_obsidian_sync.py`

- [ ] Write RED tests that installer stages every new module, compiles/hashes/modes the Swift helper, preserves global managed blocks/Claude hook, supports install-disabled mode, and keeps dry-run byte-identical without Keychain access.
- [ ] Test one authoritative migration lock/disabled journal shared with the existing publisher. Publisher cannot run between `locked` and `services_started`, never treats clean-root switch as an ordinary cycle, and starts only after collector health.
- [ ] Test the first collector run consumes authenticated deferred appends under a new epoch without modifying stage manifests. Existing catch-up, published barrier, consumed, and feedback semantics remain unchanged; no plaintext pseudonym key returns.
- [ ] Test the post-activation pending-classification worker sends only audited safe prompts, retries within its limit, moves only indexes/project links, and deterministically assigns confidence below `0.85` to `general-temporary`; the queue has no permanently dangling record.
- [ ] Run `/usr/bin/python3 -m unittest -q test_ai_session_sync_classify.py test_ai_session_sync_ledger.py test_ai_session_sync_publish_barrier.py test_ai_session_sync_e2e.py test_obsidian_sync.py`; expected RED is missing install-disabled/publisher/epoch/queue integration.
- [ ] Implement installation and service control without a second publisher or changes to trading jobs.
- [ ] Run focused GREEN and Shared Verification; commit `集成: 封闭安装发布器与激活后首轮采集`.
- [ ] Pass the Per-Task TDD and Review Gate.

---

### Task 18: Prove the complete isolated history loop and all adversarial paths

**Files:**
- Create: `test_ai_session_sync_complete_history_e2e.py`
- Create: `tests/fixtures/complete_history/README.md`
- Modify: `test_ai_session_sync_migration_shadow.py`
- Modify: `test_ai_session_sync_e2e.py`
- Modify: `test_obsidian_sync.py`

- [ ] Build a disposable harness covering both platforms, old/active/corrupt/deferred records, manual Vault remnants, duplicate/conflicting/local/remote-only Git history, S0–S3, nonmanaged blockers, LFS, all ref namespaces, linked gitdirs, v1/v2 backups, clients, and local bare remote.
- [ ] Run plan, two-pass stage, unsupported exact acceptance, both backup drills, clean-root/object/LFS audit, and every activation failure. Before-local-switch failures leave live/remote hashes equal; conflict/pending outcomes preserve evidence and publisher-disable state.
- [ ] Complete synthetic successful activation/readback/client receipts and Codex → published handoff → Claude consumed/feedback → published handoff → new Codex consumed. Require eight parent-linked stages and consistent CLI/Vault/local/remote/client status.
- [ ] Include the remote-only safe managed blob as ledgered/imported history and an unsafe extra-ref case as activation blocker.
- [ ] Repeat rebuild twice with zero Hermes and identical deterministic evidence. Scan every non-backup output plus captured argv/env/stdout/stderr; raw data exists only in authenticated backup/audit roots.
- [ ] Run `/usr/bin/python3 -m unittest -q test_ai_session_sync_complete_history_e2e.py test_ai_session_sync_migration_shadow.py test_ai_session_sync_e2e.py test_obsidian_sync.py`; expected RED identifies integration defects.
- [ ] For each defect, open a named remediation task in the owning module, make a separate focused commit, and repeat its Per-Task Review Gate. Do not bundle fixes into this test commit.
- [ ] When GREEN, run Shared Verification; commit `测试: 证明完整历史跨软件闭环`.
- [ ] Pass the Per-Task TDD and Review Gate for the harness itself.

---

### Task 19: Document, independently review, fully verify, and run the read-only readiness shadow

**Files:**
- Modify: `README.md`
- Modify: `AGENTS.md`
- Modify: `MEMORY.md`
- Create: `docs/交付总结-完整历史安全重建-20260722.md`
- Runtime-only ignored output: `.superpowers/sdd/complete-history-shadow-report.json`

- [ ] Document exact plan/audit-workspace/stage/accept/cleanup/activate/rollback/disaster-prepare/disaster-recover/client commands; six delivery states; recovery/challenge FD handling; Keychain/private route; rotation residual risk; lease conflicts; reconnect procedure; 5–15 minute sync; degraded modes; mobile limitation; Kimi exclusion.
- [ ] State plainly: code ≠ stage ≠ local activation ≠ remote switch ≠ client reconnect ≠ real bidirectional closure; clean refs do not prove provider physical purge; unregistered devices are unknown.
- [ ] Dispatch a read-only whole-branch reviewer for `b0e702d..HEAD` against every specification section. Any Critical/Important finding opens a named remediation task/commit with focused tests and review. Repeat whole-branch review until Critical=0 and Important=0; keep the final docs commit documentation-only.
- [ ] Run the complete fresh suite:

```bash
/usr/bin/python3 -m unittest -q test_ai_session_sync.py test_ai_session_sync_sources.py test_ai_session_sync_privacy.py test_ai_session_sync_hermes.py test_ai_session_sync_projects.py test_ai_session_sync_classify.py test_ai_session_sync_archive.py test_ai_session_sync_handoff.py test_ai_session_sync_ledger.py test_ai_session_sync_migration_models.py test_ai_session_sync_migration_inventory.py test_ai_session_sync_migration_private.py test_ai_session_sync_git_transport.py test_ai_session_sync_migration_rebuild.py test_ai_session_sync_migration_backup.py test_ai_session_sync_migration_backup_snapshot.py test_ai_session_sync_migration_restore.py test_ai_session_sync_migration_git.py test_ai_session_sync_migration_lfs.py test_ai_session_sync_migration_remote.py test_ai_session_sync_migration_clients.py test_ai_session_sync_migration_transaction.py test_ai_session_sync_migration_disaster.py test_ai_session_sync_migrate.py test_ai_session_sync_migration_shadow.py test_ai_session_sync_publish_barrier.py test_ai_session_sync_e2e.py test_ai_session_sync_complete_history_e2e.py test_obsidian_sync.py
.venv/bin/python test_rules.py
.venv/bin/python test_rules_mid.py
.venv/bin/python test_web.py
.venv/bin/python test_regime.py
/usr/bin/python3 -m py_compile ai_session_sync*.py obsidian_sync.py obsidian_sync_core.py obsidian_sync_digest.py
/bin/bash -n install_ai_session_sync.sh install_obsidian_sync.sh
git diff --check
git diff --exit-code b0e702d -- screener.py rules_mid.py params.py storage.py service.py
```

- [ ] Record command/exit/test count/duration/commit SHA/fixture hashes/protected diff in the delivery note.
- [ ] Run real-source `migrate plan --all-history --audit-root PLAN_RAW_ABS --audit-recovery-code-output-fd FD`, verify/remove that authenticated plan root, then run `migrate stage --all-history --staging-vault SAFE_ABS --audit-root STAGE_RAW_ABS --audit-recovery-code-output-fd FD --dry-run` using disjoint private disposable non-live roots. Compare before/after manifests of source roots, live managed paths, ordinary state, instructions, launchd, runtime, Vault Git metadata, Keychain inventory, and configured remotes. Read-only remote enumeration is allowed; no live byte/ref/credential/job/Keychain/remote change and no Hermes request are allowed.
- [ ] Record the selected internal/external LFS provider and version/preflight result. Do not auto-install `git-lfs`; if a required object has no supported provider, readiness is false with the closed reason and no raw residue.
- [ ] Save only safe counts/hashes/reasons/booleans in the ignored report; do not commit machine-specific identifiers. Remove the disposable shadow root after its manifest is recorded.
- [ ] If readiness passes, report exactly `代码完成，影子全历史暂存通过，尚未激活`. Stop for separate production activation authority; do not infer it from this plan.
- [ ] Commit reusable documentation only as `文档: 固化完整历史闭环验收与恢复手册` and pass the Per-Task Review Gate for documentation claims.

## Completion Definition

Implementation readiness requires all 19 tasks complete, every task review and the whole-branch review at Critical=0/Important=0, fresh aggregate tests green, protected trading files unchanged, and a read-only real-source shadow with no live mutation. Production activation, real remote switch, registered-client reconnect, and real user-data Codex → Claude → Codex closure remain later separately evidenced states.
