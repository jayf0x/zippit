# Zippit

Drag-drop file encryption with excellent UX on top of standard, modern, well-reviewed primitives. Goal: make strong encryption the default for non-experts, without inventing new crypto.

## What it is

A Tauri desktop app (macOS first) that encrypts files and folders using the `age` file format, with a profile system, keychain-backed key storage, and optional USB-key workflow.

Think of it as: "Encrypto, but with key management, profiles, and a CLI."

## What it is not

- Not a full-disk / container encryption tool. Use VeraCrypt or FileVault for that.
- Not a new crypto format. We do not design algorithms or file formats.
- Not a secret manager. Use 1Password / Bitwarden for secrets.
- Not quantum-safe in public-key mode. See threat model below.

## Threat model

Be explicit about what each feature defends against. If a feature cannot answer "what attack am I defending against?", it gets dropped.

### Defends against
- **Data at rest on untrusted storage** (cloud drives, GitHub, USB sticks, stolen laptop while locked). Primary use case.
- **Data in transit over untrusted channels** (email attachments, Slack, SFTP). Ciphertext is safe to share.
- **Accidental disclosure** (wrong cloud folder, wrong recipient). Encrypted blob is opaque.

### Does NOT defend against
- **Compromised endpoint while unlocked** (malware, keylogger, screen-reader). Out of scope; OS problem.
- **Physical access to an unlocked device** with mounted plaintext. Out of scope.
- **Rubber-hose / legal coercion** to reveal passphrase. Out of scope.
- **Metadata leakage**: filename, size, modification time of the ciphertext blob itself can still leak info. We wrap folders in tar before encryption so internal filenames are hidden, but the outer `.age` file size/name is not protected.
- **Harvest-now-decrypt-later against X25519 recipients.** `age` uses X25519 for asymmetric recipients, which a future CRQC (cryptographically relevant quantum computer) could break. Symmetric passphrase mode (scrypt + ChaCha20-Poly1305) remains secure against Grover at 256-bit key strength. If the user is worried about HNDL, recommend passphrase mode.

## Core architecture

### Crypto engine: `age` (via the `age` Rust crate)

**Why:** `age` is the modern, minimal replacement for GPG. Designed by Filippo Valsorda. Well-reviewed spec. Small attack surface. Supports passphrase and recipient modes. Streaming (safe for large files). Has a native Rust implementation. Any other tool we build today will be measurably worse than age for no benefit.

- Data encryption: ChaCha20-Poly1305 (AEAD) via the STREAM construction for chunked streaming.
- Passphrase mode: scrypt with work factor ≥ 18 (age default). Memory-hard. Argon2id-level security in practice.
- Recipient mode: X25519 public keys.
- Key wrapping: HKDF-SHA256.

We do not modify the format. A `.age` file produced by Zippit is readable by `rage` / `age` CLI and vice versa. This is a feature: no lock-in, no dependency on us being maintained.

### Container format: `tar` + `age`

**Why:** Folders need to be flattened to a single stream before encryption, and we need to hide internal filenames (metadata). Tar is universal, streaming, and does not re-compress already-compressed files. We pipe `tar → age` without a temp file.

Single files skip tar and go straight to age.

- Folder in → `folder.tar.age`
- File in → `file.ext.age`
- Extension `.age` (not a custom one) so any age-compatible tool can decrypt. Interoperability > branding.

### Key sources

Three supported sources, ranked by recommendation:

1. **macOS Keychain (default).** Private age identity (X25519) stored as a generic password item, access-controlled by Touch ID / device passcode.
   **Defends against:** casual device theft, accidental plaintext key copies, shoulder-surfed passphrase typing. OS enforces biometric gate per use.
2. **Passphrase (fallback / portable).** Direct scrypt passphrase mode. No key file at rest.
   **Defends against:** no device binding; good when the user must open files on a different machine or OS. HNDL-safe because it is symmetric.
3. **Identity file on external storage (USB).** Plain `age` identity file on a removable drive. App only reads it when the drive is mounted.
   **Defends against:** attacker with software access to the main device but no physical access to the USB key. Air-gapped-ish key material.

Keys live in memory for the duration of a single encrypt/decrypt operation and are zeroized after (via the `zeroize` crate). No background key caching.

### Minimal key handling

**Why:** Keys in memory are the main leak surface after crypto is chosen. Rule: keys exist for one operation, then vanish.

- Passphrases read via secure prompt, never stored, zeroized after derivation.
- Private identities loaded on demand, zeroized after file encrypt/decrypt.
- No "unlock for 5 minutes" caches in v1. OS keychain + Touch ID is the cache.
- No passphrase echoing, clipboard, or history.

### OS-level protections we rely on

**Why:** We do not out-engineer the OS. The OS already solves many problems; we should use those primitives, not reinvent them.

- macOS Keychain for identity storage (hardware-backed on Apple Silicon via Secure Enclave where available).
- Touch ID / device passcode as the access gate.
- App sandbox + entitlements to limit filesystem and keychain scope.
- Code signing + notarization required before shipping (keychain access demands it anyway).

## Powerful features

Features beyond the minimum that earn their keep.

### Drag-drop UI

**Why:** The whole value proposition. Encryption is a commodity; UX is what people actually pay for. Drop files in → pick a profile → press encrypt. No terminology, no configuration on first use.

### External / USB key support

**Why:** Physical separation of key material from the device. Defends against malware that can read the filesystem but cannot access an unplugged USB. This is a meaningful and well-defined attack reduction, not theater.

Implementation: detect mount, read identity file, do not copy to local disk, zeroize on completion.

### Profile system

**Why:** A profile is a saved `{key_source, output_target, options}` bundle. Users do not re-decide how to encrypt their Obsidian vault every time. One click, same choices as last time. Reduces user error (wrong key, wrong output location), which is a real threat.

Stored as plain JSON in app config; contains no secret material, only references (keychain item ID, path patterns).

### Decrypt-by-double-click

**Why:** `.age` files on disk should just open. App registers as handler, prompts for key/passphrase via secure dialog, extracts to a target directory, done. No CLI needed for the common path.

## Deferred / optional features

Parked until after core ships. Each must still answer "what attack does this mitigate" or "what real UX gap does this close" before being built.

### CLI parity

Scriptable encrypt/decrypt with the same profiles as the GUI. Defense gain: none. UX gain: real, for power users and automation. Low priority but cheap to add.

### Auto-encrypt on change (watched folders)

Watch a plaintext working folder, auto-produce an encrypted mirror on change.
**Attack defended:** reduces the window where plaintext exists in sync-mounted storage (Dropbox, iCloud). Real but narrow.
**Risks:** plaintext still lives somewhere on disk; false sense of security; FS-watcher edge cases. Defer until core is rock-solid.

### Multi-layer / chained algorithms ("paranoid mode")

Originally a headline feature. Cut from core.
**Why cut:** Stacking AES on ChaCha on AES does not meaningfully raise the bar against a capable attacker. Two correctly-implemented AEADs at 256-bit are already beyond brute force. The weak link is key management, which chaining does not help. Chaining also increases implementation surface and bug risk.
**If users want defense in depth:** use Zippit (age) on top of VeraCrypt container. Composition across tools is safer than chaining inside one tool.
**Could return as:** an explicit "paranoid" toggle that wraps the age output in a second independent primitive (e.g., XSalsa20-Poly1305 via libsodium). Only if users request it and understand it is belt-and-suspenders, not extra security.

### Plugin system for external tools (VeraCrypt, etc.)

Originally proposed as JSON-configured external binary invocation.
**Why deferred:** hard to do safely (arbitrary subprocess with user files), hard to explain, easy for users to misconfigure into insecurity. If a user needs VeraCrypt, they should open VeraCrypt. Plugin registry is also a supply-chain attack surface we do not want in v1.

### Obsidian / vault plugin

Community plugin that encrypts a vault with Zippit. Natural fit because the app was born from encrypting Obsidian notes. Defer until core engine stabilizes; then build as a thin wrapper around the CLI.

### Recipient-mode sharing UI

Encrypt a file to someone else's age public key. Useful for sharing but non-trivial UX (key exchange, trust). Defer.

## Tech stack

- Shell: Tauri 2 (macOS target first; Windows / Linux follow automatically for the core).
- Frontend: whatever the user prefers (SolidJS / Svelte / React). No security implications.
- Core crypto: `age` Rust crate.
- Streaming I/O: `tokio` + `tar` crate.
- Secure memory: `zeroize` crate.
- Keychain: `security-framework` crate (macOS). Later: `keyring` crate for cross-platform.
- Config / profiles: `serde_json`, stored in `$APP_CONFIG_DIR/zippit/profiles.json`.
- CLI: `clap`. Shares the same Rust core as the GUI.

## Non-goals

- Inventing file formats.
- Building our own cipher, KDF, or AEAD mode.
- Post-quantum cryptography in v1. Revisit when age adds PQ recipients, or offer passphrase mode as the HNDL-safe path.
- Cross-user key escrow / recovery. Users are responsible for backing up their identity.
- Secure deletion of plaintext inputs. Different problem; out of scope.

## Success criteria for v1

- A non-technical user can encrypt a folder with a Touch ID press and no reading.
- Resulting `.age` / `.tar.age` file decrypts with the standard `age` CLI on any OS.
- No key material persists on disk unencrypted except user-chosen identity files.
- Zero custom cryptography.
