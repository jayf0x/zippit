# Zippit Backlog

Core features only. Each item is sized for a single agent session. See `zippit.md` for architecture and rationale.

## Conventions

- `status: open` — scoped, ready to implement.
- `status: research` — needs investigation before coding (library choice, API shape, edge cases).
- `status: blocked` — depends on another item.
- `depends:` — list of item IDs that must be done first.

---

## Core

### C1. Crypto engine wrapper

Wrap the `age` Rust crate behind a thin internal API: `encrypt(reader, recipients) -> writer`, `decrypt(reader, identities) -> writer`. Streaming. No temp files. Zeroize key material on drop.

status: open

---

### C2. Container: single file vs folder

Detect input type. Single file → stream straight to age. Folder → stream through `tar` into age (no temp archive on disk). Output paths: `file.ext.age` and `folder.tar.age`.

status: open
depends: C1

---

### C3. Passphrase encryption path

Secure passphrase prompt (no echo, no clipboard, zeroized buffer). Feed to age scrypt recipient. Verify round-trip with the `age` CLI for interop.

status: open
depends: C1

---

### C4. Identity file loading (local path + USB)

Load an age identity (`AGE-SECRET-KEY-...`) from a user-picked file path. Detect removable volumes on macOS so the UI can surface "key on USB" as a first-class option. Never copy the identity to local disk.

status: research

---

### C5. macOS Keychain integration

Store a generated age identity in the macOS Keychain as a generic password, access-controlled with `kSecAccessControlUserPresence` (Touch ID / passcode). Retrieve on demand. Create / rotate / delete flows.

status: research

---

### C6. Identity generation flow

Generate a new X25519 age identity. Present the public key (recipient) to the user. Offer three destinations: keychain, identity file on disk, identity file on USB. Warn about backup responsibility.

status: open
depends: C4, C5

---

### C7. Decrypt: format + key-source detection

On decrypt, sniff the age header to determine passphrase vs recipient mode. Prompt the user for the matching key source (passphrase / keychain / identity file). If recipient-mode, try available keychain identities first before prompting.

status: open
depends: C1, C3, C5

---

### C8. Drag-drop UI + encrypt flow

Tauri window with drop zone. On drop: list inputs, pick profile (or default), show progress, write output next to input by default. No configuration required for the happy path.

status: open
depends: C2, C6

---

### C9. Decrypt UI + file association

Double-click `.age` / `.tar.age` opens Zippit decrypt dialog. Ask for destination, run decrypt, reveal in Finder on completion. Register file handler in Tauri / Info.plist.

status: research

---

### C10. Profile system

Load / save / edit profiles in `$APP_CONFIG_DIR/zippit/profiles.json`. Profile = `{name, key_source_ref, output_policy, container_hint}`. Contains references only, no secret material. Migration-safe schema with a version field.

status: open

---

### C11. Secure memory handling

Audit all paths touching passphrases, identities, derived keys. Use `zeroize::Zeroizing` for buffers. Add a clippy / test pass that fails if plaintext key types escape the crypto module.

status: open
depends: C1, C3, C5

---

### C12. Error handling + user-facing messages

Map every error to a user-friendly message without leaking crypto detail that could help an attacker (e.g., do not distinguish "wrong passphrase" from "corrupt file" beyond what age itself reveals). Centralized error type in the core crate.

status: open
depends: C1

---

### C13. Large-file / large-folder streaming verification

Test with 10 GB file and 100k-file folder. Verify constant memory use, no temp files, progress reporting works. Add a benchmark to CI so regressions are caught.

status: open
depends: C2, C8

---

### C14. Code signing + notarization pipeline

Developer ID cert, notarization, stapling. Required for keychain entitlements to work outside dev builds. Document the release command.

status: research

---

### C15. Threat model doc + in-app "what this protects" panel

One-screen plain-language summary of what Zippit defends against and what it does not. Linked from the main UI. Content lives in `zippit.md`; UI reads from a shared source of truth.

status: open

---

## Out of core (tracked for later)

Moved here so they do not get lost, but not part of v1.

- CLI parity (shares the Rust core).
- Auto-encrypt on folder change.
- Paranoid / multi-layer mode.
- External tool plugins (VeraCrypt, etc.).
- Obsidian vault plugin.
- Recipient-mode sharing UI (encrypt to someone else's public key).
- Windows / Linux keychain backends via `keyring` crate.
