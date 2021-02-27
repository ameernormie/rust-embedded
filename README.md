# Embedded-rust

Practicing embedded rust following the [rust embedded book](https://docs.rust-embedded.org/discovery/)

#### Testing discovery board and bluetooth module

Board name is `STM32F3DISCOVERY`

For stm board run command
`lsusb | grep -i stm`

For serial module run command
`lsusb | grep -i ft232`

```
openocd -f interface/stlink-v2-1.cfg -f target/stm32f3x.cfg
```

### STEPS TO FLASH CODE TO MICROCONTROLLER

##### Build-it

First step is to `build` binary crate. Microcontroller has different architecture than computer, we'll need to cross compile. Pass `--target` flag to **rustc** or **cargo** cross compile. The microcontroller in the `F3` has a `Cortex-M4F` processor in it.

- target will be`thumbv7em-none-eabihf`, for the Cortex-M4F and Cortex-M7F processors

Before cross compiling, download pre-compiled version for the target. (one time operation)

```rust
rustup target add thumbv7em-none-eabihf
```

Now, cross compile using

```rust
cargo build --target thumbv7em-none-eabihf
```

Now, an executable is produced. This executable won't blink any leds. Verify produced executable is actually an ARM binary:

```rust
cargo readobj --target thumbv7em-none-eabihf --bin led-roulette -- -file-headers
```

##### Flash-it

Flashing is the process of moving our program into the microcontroller's (persistent) memory. Once flashed, the microcontroller will execute the flashed program every time it is powered on.

- Firstly, launch `openocd` using `openocd -f interface/stlink-v2-1.cfg -f target/stm32f3x.cfg` in the `/tmp` directory. The "6 breakpoints, 4 watchpoints" part indicates the debugging features the processor has available.

```rust
openocd -f interface/stlink-v2-1.cfg -f target/stm32f3x.cfg
```

OpenOCD is a software that provides some services like a GDB server on top of USB devices that expose a debugging protocol like SWD or JTAG.
Leave that `openocd` process running, and open a new terminal. Make sure you are in source directory.
`gdb` represents a program capable of debugging ARM binaries.

```rust
arm-none-eabi-gdb -q target/thumbv7em-none-eabihf/debug/led-roulette
```

This only opens a GDB shell. To actually connect to the OpenOCD GDB server, use the following command within the GDB shell:

```rust
(gdb) target remote :3333
Remote debugging using :3333
0x00000000 in ?? ()
```

If you are getting errors like `undefined debug reason 7 - target needs reset`, you can try running `monitor reset halt`.
By default OpenOCD's GDB server listens on TCP port 3333 (localhost). This command is connecting to that port.

Almost there. To flash the device, we'll use the load command inside the GDB shell:

```
(gdb) load
Loading section .vector_table, size 0x188 lma 0x8000000
Loading section .text, size 0x38a lma 0x8000188
Loading section .rodata, size 0x8 lma 0x8000514
Start address 0x8000188, load size 1306
Transfer rate: 6 KB/sec, 435 bytes/write.
```

##### Debug-it
