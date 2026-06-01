# Zippit

> Depricated concept. Easy to replace with a tool like [majic](https://github.com/jayf0x/majic).

---

A drag-and-drop file encryption desktop app for macOS. Encrypts files and folders using the [age](https://age-encryption.org/) format — interoperable with standard `age`/`rage` CLI tools.

## Features

- Drag-and-drop encryption and decryption of files and folders
- Profile-based workflows for managing multiple recipients or key sets
- macOS Keychain integration with Touch ID biometric access
- Optional USB key storage for portable decryption keys
- Outputs standard `.age` files compatible with any `age`-compatible tool

## Requirements

- macOS
- Rust toolchain (`rustc`, `cargo`)
- Xcode Command Line Tools

## Setup

```bash
# Install dependencies
cargo build --release
```

Or run in development mode:

```bash
cargo tauri dev
```

## Tech Stack

- [Tauri 2](https://tauri.app/) — desktop shell
- Rust — encryption logic (`age`, `tokio`, `zeroize`)
- macOS Security framework — Keychain and Touch ID
