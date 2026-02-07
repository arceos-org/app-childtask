# arceos-childtask

A standalone multi-task application running on [ArceOS](https://github.com/arceos-org/arceos) unikernel, with all dependencies sourced from [crates.io](https://crates.io). Demonstrates spawning a child task (thread) that accesses QEMU PFlash via MMIO, establishing the basic multi-task framework: **task** and **run queue**.

## What It Does

This application introduces multi-task concepts on top of ArceOS:

1. **Task framework**: The `multitask` feature in `axstd` enables the task scheduler with run queues, allowing `thread::spawn` and `thread::join`.
2. **Page table setup**: The `paging` feature enables kernel page tables that map MMIO regions (including PFlash) into the virtual address space.
3. **Child task**: The main task spawns a worker thread that reads a 4-byte magic string (`"PFLA"`) from QEMU PFlash via direct MMIO access.
4. **Task synchronization**: The main task waits for the child task to finish via `join()` and verifies the result.

## PFlash Address Map

| Architecture | PFlash Unit | Physical Address | QEMU Option |
|---|---|---|---|
| riscv64 | pflash1 | `0x22000000` | `-drive if=pflash,unit=1` |
| aarch64 | pflash1 | `0x04000000` | `-drive if=pflash,unit=1` |
| x86_64 | pflash0 | `0xFFC00000` | `-drive if=pflash,unit=0` (with embedded SeaBIOS) |
| loongarch64 | pflash1 | `0x1D000000` | `-drive if=pflash,unit=1` |

## Supported Architectures

| Architecture | Rust Target | QEMU Machine | Platform |
|---|---|---|---|
| riscv64 | `riscv64gc-unknown-none-elf` | `qemu-system-riscv64 -machine virt` | riscv64-qemu-virt |
| aarch64 | `aarch64-unknown-none-softfloat` | `qemu-system-aarch64 -machine virt` | aarch64-qemu-virt |
| x86_64 | `x86_64-unknown-none` | `qemu-system-x86_64 -machine q35` | x86-pc |
| loongarch64 | `loongarch64-unknown-none` | `qemu-system-loongarch64 -machine virt` | loongarch64-qemu-virt |

## Prerequisites

- **Rust nightly toolchain** (edition 2024)

  ```bash
  rustup install nightly
  rustup default nightly
  ```

- **Bare-metal targets** (install the ones you need)

  ```bash
  rustup target add riscv64gc-unknown-none-elf
  rustup target add aarch64-unknown-none-softfloat
  rustup target add x86_64-unknown-none
  rustup target add loongarch64-unknown-none
  ```

- **QEMU** (install the emulators for your target architectures)

  ```bash
  # Ubuntu/Debian
  sudo apt install qemu-system-riscv64 qemu-system-aarch64 \
                   qemu-system-x86 qemu-system-loongarch64  # OR qemu-system-misc

  # macOS (Homebrew)
  brew install qemu
  ```

- **SeaBIOS** (required for x86_64 only)

  ```bash
  # Ubuntu/Debian
  sudo apt install seabios
  ```

- **rust-objcopy** (from `cargo-binutils`, required for non-x86_64 targets)

  ```bash
  cargo install cargo-binutils
  rustup component add llvm-tools
  ```

## Quick Start

```bash
# install cargo-clone sub-command
cargo install cargo-clone
# get source code of arceos-childtask crate from crates.io
cargo clone arceos-childtask
# into crate dir
cd arceos-childtask
# Build and run on RISC-V 64 QEMU (default)
cargo xtask run

# Build and run on other architectures
cargo xtask run --arch aarch64
cargo xtask run --arch x86_64
cargo xtask run --arch loongarch64

# Build only (no QEMU)
cargo xtask build --arch riscv64
cargo xtask build --arch aarch64
```

Expected output (riscv64 example):

```
       d8888                            .d88888b.   .d8888b.
      d88888                           d88P" "Y88b d88P  Y88b
     ...
d88P     888 888      "Y8888P  "Y8888   "Y88888P"   "Y8888P"

arch = riscv64
platform = riscv64-qemu-virt
...
smp = 1

Multi-task is starting ...
Spawned-thread is running ...
Try to access pflash dev region [0xFFFF_FFC0_2200_0000], got 0x414C4650
Got pflash magic: PFLA
Multi-task OK!
```

QEMU will automatically exit after printing the message.

## Project Structure

```
app-childtask/
├── .cargo/
│   └── config.toml       # cargo xtask alias & AX_CONFIG_PATH
├── xtask/
│   └── src/
│       └── main.rs       # build/run tool (pflash image creation + QEMU launch)
├── configs/
│   ├── riscv64.toml      # Platform config with PFlash MMIO range
│   ├── aarch64.toml      # Platform config with PFlash MMIO range
│   ├── x86_64.toml       # Platform config with PFlash MMIO range
│   └── loongarch64.toml  # Platform config with PFlash MMIO range
├── src/
│   └── main.rs           # Application entry: spawns child task to read PFlash
├── build.rs              # Linker script path setup (auto-detects arch)
├── Cargo.toml            # Dependencies (axstd with paging + multitask features)
└── README.md
```

## How It Works

The `cargo xtask` pattern uses a host-native helper crate (`xtask/`) to orchestrate
cross-compilation and QEMU execution:

1. **`cargo xtask build --arch <ARCH>`**
   - Copies `configs/<ARCH>.toml` to `.axconfig.toml` (platform configuration with PFlash MMIO range)
   - Runs `cargo build --release --target <TARGET>`
   - `build.rs` auto-detects the architecture and locates the correct linker script

2. **`cargo xtask run --arch <ARCH>`**
   - Performs the build step above
   - Creates a PFlash image with magic string `"PFLA"` at offset 0
   - For x86_64: embeds SeaBIOS at the end of the pflash image (combined BIOS + data)
   - Converts ELF to raw binary via `rust-objcopy` (except x86_64)
   - Launches QEMU with the PFlash image attached

## Key Components

| Component | Role |
|---|---|
| `axstd` | ArceOS standard library (replaces Rust's `std` in `no_std` environment) |
| `axhal` | Hardware abstraction layer, provides `phys_to_virt` for address translation |
| `axtask` | Task scheduler with run queues, enabled by `multitask` feature |
| `axplat-*` | Platform-specific support crates (one per target board/VM) |
| `axruntime` | Kernel initialization and runtime setup (including page table creation) |
| `paging` feature | Enables page table management; maps MMIO regions listed in config |
| `multitask` feature | Enables multi-task scheduler with `thread::spawn` / `thread::join` |
| `build.rs` | Locates the linker script generated by `axhal` and passes it to the linker |
| `configs/*.toml` | Pre-generated platform configuration with PFlash MMIO ranges |

## License

GPL-3.0-or-later OR Apache-2.0 OR MulanPSL-2.0
