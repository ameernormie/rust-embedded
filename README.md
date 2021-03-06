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

| Commands                                                             | Description                                                                                                                                            |
| -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| cargo build --target thumbv7em-none-eabihf                           | cross compile the code                                                                                                                                 |
| arm-none-eabi-gdb -q target/thumbv7em-none-eabihf/debug/led-roulette | open `gdb`                                                                                                                                             |
| load                                                                 | Flash the code to microcontroller                                                                                                                      |
| break main                                                           | Add a breakpoint at `main`                                                                                                                             |
| continue                                                             | run the program freely until breakpoint is reached                                                                                                     |
| layout src                                                           | For nicer debugging experience,GDB's Text User Interface (TUI)                                                                                         |
| tui disable                                                          | Exit tui mode                                                                                                                                          |
| print <variable>                                                     | find out the value of variable while debugging                                                                                                         |
| info locals                                                          | print all local variables                                                                                                                              |
| monitor reset halt && continue                                       | Roll back debugging to go to start of breakpoint                                                                                                       |
| step                                                                 | `step` command will go inside function them.                                                                                                           |
| next                                                                 | `next` command will step over function calls instead of going inside them.                                                                             |
| quit                                                                 | quit from `gdb`                                                                                                                                        |
| info args                                                            | show the arguments of a function                                                                                                                       |
| backtrace (or bt)                                                    | Regardless of where your program may have stopped you can always look at the output of the backtrace command (bt for short) to learn how it got there: |
| finish                                                               | return to main function                                                                                                                                |

`Trick`

> Our GDB sessions always involve entering the same commands at the beginning. We can use a .gdb file to execute some commands right after GDB is started. This way you can save yourself the effort of having to enter them manually on each GDB session.

> Place this openocd.gdb file in the root of the Cargo project, right next to the Cargo.toml:

```javascript
// openocd.gdb
target remote :3333
load
break main
continue
```

> Then modify the second line of the .cargo/config file:

```javascript
// .cargo/config
[target.thumbv7em-none-eabihf]
runner = "arm-none-eabi-gdb -q -x openocd.gdb" # <-
rustflags = [
  "-C", "link-arg=-Tlink.x",
]
```

After that just run: `cargo run --target thumbv7em-none-eabihf`

### ITM (Instrumentation Trace Macrocell)

The `iprintln` macro will format messages and output them to the microcontroller's ITM. ITM stands for Instrumentation Trace Macrocell and it's a communication protocol on top of SWD (Serial Wire Debug) which can be used to send messages from the microcontroller to the debugging host. This communication is only one way: the debugging host can't send data to the microcontroller.

OpenOCD, which is managing the debug session, can receive data sent through this ITM channel and redirect it to a file.
The ITM protocol works with frames (you can think of them as Ethernet frames). Each frame has a header and a variable length payload. OpenOCD will receive these frames and write them directly to a file without parsing them. So, if the microntroller sends the string "Hello, world!" using the `iprintln` macro, OpenOCD's output file won't exactly contain that string.
To retrieve the original string, OpenOCD's output file will have to be parsed. We'll use the itmdump program to perform the parsing as new data arrives.

```console
$ # itmdump terminal

$ cd /tmp && touch itm.txt

$ # This command will block as itmdump is now watching the itm.txt file. Leave this terminal open.
$ itmdump -F -f itm.txt
```

```console
(gdb) # globally enable the ITM and redirect all output to itm.txt
(gdb) monitor tpiu config internal itm.txt uart off 8000000

(gdb) # enable the ITM port 0
(gdb) monitor itm port 0 on
```
