---
title: Setting up Rust for ESP32-C3 on macOS
tags:
  - programming
  - embedded-systems
  - esp32
---
Install the stable [Rust toolchain](https://rust-lang.org/tools/install/) on your Mac. As the ESP32-C3 uses the RISCV ISA, we need to add the [`riscv32imc` target](https://doc.rust-lang.org/rustc/platform-support/riscv32-unknown-none-elf.html) to the toolchain:

```bash
rustup target add riscv32imc-unknown-none-elf
```

Next, install
* `esp-generate` to generate ESP32 Rust projects
* `espflash` to manage programs on the ESP32 device
* `probe-rs` for debugging capabilities

```
cargo install esp-generate --locked
cargo install espflash --locked
cargo install probe-rs-tools --locked
```

Use `esp-generate` to generate a project targetting `esp32-c3`:

```
$ esp-generate <project-name>
? Select your target chip:  
  esp32
  esp32c2
> esp32c3
  esp32c5
  esp32c6
  esp32c61
v esp32h2
[↑↓ to move, enter to select, type to filter]
```

On selecting the device, the project configuration menu opens up. Modify the configuration if required, else press `S` to save the configuration and generate the project:

![[Pasted image 20260503080809.png]]

The project is generated in `<project-name>` directory:

```
$ esp-generate <project-name>
> Select your target chip: esp32c3
Selected options: --chip esp32c3

Checking installed versions
🆗 Rust (stable): 1.94.0
🆗 espflash: 4.4.0
🆗 probe-rs: 0.31.0
🆗 esp-config: 0.7.0
```

Run `cargo run --release` to build, flash and monitor the application on the C3:

```
cd <project-name>
cargo run --release
```

