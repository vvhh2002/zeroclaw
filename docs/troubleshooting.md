# ZeroClaw Troubleshooting

This guide focuses on common setup/runtime failures and fast resolution paths.

Last verified: **February 19, 2026**.

## Installation / Bootstrap

### `cargo` not found

Symptom:

- bootstrap exits with `cargo is not installed`

Fix:

```bash
./bootstrap.sh --install-rust
```

Or install from <https://rustup.rs/>.

### Missing system build dependencies

Symptom:

- build fails due to compiler or `pkg-config` issues

Fix:

```bash
./bootstrap.sh --install-system-deps
```

### Build fails with out-of-memory (OOM)

Symptoms:

- `cargo build --release` is killed by the OS (signal 9 / OOM killer)
- system becomes unresponsive during compilation
- error messages mentioning `cannot allocate memory`

Why this happens:

- Compiling ZeroClaw from source requires approximately **2 GB of RAM + swap** (minimum).
- The Rust compiler and linker (especially with LTO) are memory-intensive.
- Heavy dependency trees (`matrix-sdk`, `reqwest`, crypto stacks) add to peak memory.
- The runtime binary uses < 5 MB, but compilation needs significantly more.

Workarounds (try in order):

1. **Add swap space** (recommended for devices with 1 GB RAM):

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
# Then retry: cargo build --release --locked
```

2. **Limit parallel compilation** to reduce peak memory:

```bash
CARGO_BUILD_JOBS=1 cargo build --release --locked
```

3. **Skip heavy optional features** (Matrix E2EE stack is the largest dependency):

```bash
cargo build --release --locked --no-default-features --features hardware
```

4. **Cross-compile from a more powerful machine:**

```bash
# On a build machine (targeting Raspberry Pi / ARM64 Linux):
rustup target add aarch64-unknown-linux-gnu
cargo build --release --locked --target aarch64-unknown-linux-gnu
# Then copy target/aarch64-unknown-linux-gnu/release/zeroclaw to the device
```

5. **Download a pre-built binary** from [GitHub Releases](https://github.com/zeroclaw-labs/zeroclaw/releases) instead of compiling on the device.

### Build is very slow or appears stuck

Symptoms:

- `cargo check` / `cargo build` appears stuck at `Checking zeroclaw` for a long time
- repeated `Blocking waiting for file lock on package cache` or `build directory`

Why this happens in ZeroClaw:

- Matrix E2EE stack (`matrix-sdk`, `ruma`, `vodozemac`) is large and expensive to type-check.
- TLS + crypto native build scripts (`aws-lc-sys`, `ring`) add noticeable compile time.
- `rusqlite` with bundled SQLite compiles C code locally.
- Running multiple cargo jobs/worktrees in parallel causes lock contention.

Fast checks:

```bash
cargo check --timings
cargo tree -d
```

The timing report is written to `target/cargo-timings/cargo-timing.html`.

Faster local iteration (when Matrix channel is not needed):

```bash
cargo check --no-default-features --features hardware
```

This skips `channel-matrix` and can significantly reduce compile time.

To build with Matrix support explicitly enabled:

```bash
cargo check --no-default-features --features hardware,channel-matrix
```

Lock-contention mitigation:

```bash
pgrep -af "cargo (check|build|test)|cargo check|cargo build|cargo test"
```

Stop unrelated cargo jobs before running your own build.

### `zeroclaw` command not found after install

Symptom:

- install succeeds but shell cannot find `zeroclaw`

Fix:

```bash
export PATH="$HOME/.cargo/bin:$PATH"
which zeroclaw
```

Persist in your shell profile if needed.

## Runtime / Gateway

### Gateway unreachable

Checks:

```bash
zeroclaw status
zeroclaw doctor
```

Verify `~/.zeroclaw/config.toml`:

- `[gateway].host` (default `127.0.0.1`)
- `[gateway].port` (default `3000`)
- `allow_public_bind` only when intentionally exposing LAN/public interfaces

### Pairing / auth failures on webhook

Checks:

1. Ensure pairing completed (`/pair` flow)
2. Ensure bearer token is current
3. Re-run diagnostics:

```bash
zeroclaw doctor
```

## Channel Issues

### Telegram conflict: `terminated by other getUpdates request`

Cause:

- multiple pollers using same bot token

Fix:

- keep only one active runtime for that token
- stop extra `zeroclaw daemon` / `zeroclaw channel start` processes

### Channel unhealthy in `channel doctor`

Checks:

```bash
zeroclaw channel doctor
```

Then verify channel-specific credentials + allowlist fields in config.

## Service Mode

### Service installed but not running

Checks:

```bash
zeroclaw service status
```

Recovery:

```bash
zeroclaw service stop
zeroclaw service start
```

Linux logs:

```bash
journalctl --user -u zeroclaw.service -f
```

## Legacy Installer Compatibility

Both still work:

```bash
curl -fsSL https://raw.githubusercontent.com/zeroclaw-labs/zeroclaw/main/scripts/bootstrap.sh | bash
curl -fsSL https://raw.githubusercontent.com/zeroclaw-labs/zeroclaw/main/scripts/install.sh | bash
```

`install.sh` is a compatibility entry and forwards/falls back to bootstrap behavior.

## Still Stuck?

Collect and include these outputs when filing an issue:

```bash
zeroclaw --version
zeroclaw status
zeroclaw doctor
zeroclaw channel doctor
```

Also include OS, install method, and sanitized config snippets (no secrets).

## Related Docs

- [operations-runbook.md](operations-runbook.md)
- [one-click-bootstrap.md](one-click-bootstrap.md)
- [channels-reference.md](channels-reference.md)
- [network-deployment.md](network-deployment.md)
