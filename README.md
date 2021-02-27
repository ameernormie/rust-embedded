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

**Build-it**

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

**Flash-it**

Flashing is the process of moving our program into the microcontroller's (persistent) memory. Once flashed, the microcontroller will execute the flashed program every time it is powered on.

- Firstly, launch `openocd` using `openocd -f interface/stlink-v2-1.cfg -f target/stm32f3x.cfg` in the `/tmp` directory. The "6 breakpoints, 4 watchpoints" part indicates the debugging features the processor has available.

```rust
// Be sure you are in /tmp directory
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

**Debug-it**

```javascript
(gdb) break main
Breakpoint 1 at 0x800018c: file src/05-led-roulette/src/main.rs, line 10.

(gdb) continue
Continuing.
Note: automatically using hardware breakpoints for read-only addresses.

Breakpoint 1, main () at src/05-led-roulette/src/main.rs:10
10          let x = 42;
```

Breakpoints can be used to stop the normal flow of a program. The `continue` command will let the program run freely until it reaches a breakpoint. In this case, until it reaches the main function because there's a breakpoint there.

Note that GDB output says "Breakpoint 1". Remember that our processor can only use six of these breakpoints so it's a good idea to pay attention to these messages.

For a nicer debugging experience, we'll be using GDB's Text User Interface (TUI). To enter into that mode, on the GDB shell enter the following command:

```javascript
(gdb) layout src
```

At any point you can leave the TUI mode using the following command:

```javascript
(gdb) tui disable
```

OK. We are now at the beginning of main. We can advance the program statement by statement using the `step` command.

Let's inspect those stack/local variables using the `print` command:

```javascript
(gdb) print x
$1 = 42

(gdb) print &x
$2 = (i32 *) 0x10001ff4

(gdb) print _y
$3 = -536810104

(gdb) print &_y
$4 = (i32 *) 0x10001ff0
```

Instead of printing the local variables one by one, you can also use the `info locals` command:

```javascript
(gdb) info locals
x = 42
_y = -536810104
```

We can switch to the disassemble view with the `layout asm` command and advance one instruction at a time using `stepi`. You can always switch back into Rust source code view later by issuing the layout src command again.

One last trick before we move to something more interesting. Enter the following commands into GDB:

```javascript
(gdb) monitor reset halt
Unable to match requested speed 1000 kHz, using 950 kHz
Unable to match requested speed 1000 kHz, using 950 kHz
adapter speed: 950 kHz
target halted due to debug-request, current mode: Thread
xPSR: 0x01000000 pc: 0x08000196 msp: 0x10002000

(gdb) continue
Continuing.
```

`monitor reset halt` will reset the microcontroller and stop it right at the program entry point. The following `continue` command will let the program run freely until it reaches the main function that has a breakpoint on it.

This combo is handy when you, by mistake, skipped over a part of the program that you were interested in inspecting. You can easily roll back the state of your program back to its very beginning.

You can end debug session with the `quit` command.
