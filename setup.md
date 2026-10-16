
## IDE

I prefer using VS Code.

Install these extensions:

- C/C++ Extension Pack
- Makefile Tools
- Cortex-Debug

Don't be misled by its name - Cortex-Debug also supports RISC-V.

Install the RISC-V32 GCC provided by xPack.

Set the terminal `PATH` environment variable to include the toolchain and openocd paths, etc. In my configuration:

```json
    "terminal.integrated.env.linux": {
        "PATH": "/home/user1/devel/riscv/xpack-riscv-none-elf-gcc-15.2.0-1/bin:/home/user1/devel/esp32/idf/tools/openocd-esp32/v0.12.0-esp32-20260703/openocd-esp32/bin:${env:PATH}"
    }
```

Here OpenOCD is provided by IDF and supports the ESP32-C3 USB serial/JTAG interface. 

It seems that the official OpenOCD release also supports USB serial/JTAG; for example, it contains the configuration file `interface/esp_usb_jtag.cfg`. However, it doesn't support ESP32-C3 targets (lacking `target/esp32c3.cfg`).

Makefile Tools builds within a terminal session, so `PATH` is also effective for it.

Cortex-Debug also requires some settings, including tools paths (for example, to avoid including local paths in the project's `.vscode/launch.json`). Like this:

```json
    "cortex-debug.armToolchainPath.linux": "/home/user1/devel/riscv/xpack-riscv-none-elf-gcc-15.2.0-1/bin",
    "cortex-debug.armToolchainPrefix": "riscv-none-elf",
    "cortex-debug.openocdPath.linux": "/home/user1/devel/esp32/idf/tools/openocd-esp32/v0.12.0-esp32-20260703/openocd-esp32/bin/openocd"
```

Note that although some items include "arm" in their names, they also work for RISC-V.

Additionally, regarding `cortex-debug.armToolchainPrefix`: first, once it's set correctly, there's no need to set paths for tools like gdb, objdump, etc. The most important point to note is that it indeed doesn't apply to RISC-V projects. Instead, you need to set it also in the project's `.vscode/launch.json`, with a similar but **different** name:

```json
            "toolchainPrefix": "riscv-none-elf",
```

This is a heads-up before starting actual projects.

I don't (currently) plan to use tools from the IDF Python venv within VS Code.


## OpenOCD

Let me try out OpenOCD early:

```
$ openocd -f interface/esp_usb_jtag.cfg -f target/esp32c3.cfg
Open On-Chip Debugger v0.12.0-esp32-20260703 (2026-07-03-13:49)
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
Info : esp_usb_jtag: capabilities descriptor set to 0x2000
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Info : esp_usb_jtag: serial (90:64:9B:0A:C7:3C)
Info : esp_usb_jtag: Device found. Base speed 40000KHz, div range 1 to 255
Info : clock speed 40000 kHz
Info : JTAG tap: esp32c3.tap0 tap/device found: 0x00005c25 (mfg: 0x612 (Espressif Systems), part: 0x0005, ver: 0x0)
Info : [esp32c3] datacount=2 progbufsize=16
Info : [esp32c3] Examined RISC-V core
Info : [esp32c3]  XLEN=32, misa=0x40101104
Info : [esp32c3] Chip revision v0.4
Info : [esp32c3] Examination succeed
Info : [esp32c3] starting gdb server on 3333
Info : Listening on port 3333 for gdb connections
```

Open a new terminal and telnet to the OpenOCD server:

```
$ telnet localhost 4444
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Open On-Chip Debugger
> 
> 
> reset halt
JTAG tap: esp32c3.tap0 tap/device found: 0x00005c25 (mfg: 0x612 (Espressif Systems), part: 0x0005, ver: 0x0)
[esp32c3] Reset cause (3) - (Software core reset)
> 
> targets
    TargetName         Type       Endian TapName            State       
--  ------------------ ---------- ------ ------------------ ------------
 0* esp32c3            esp32c3    little esp32c3.tap0       halted
> 
> reg pc
pc (/32): 0x40000000
```

0x40000000 is the boot ROM start address.

Check the boot ROM:

```
> dump_image boot_rom_v0.4.bin 0x40000000 0x60000
dumped 393216 bytes in 13.617372s (28.199 KiB/s)
```

```bash
$ riscv-none-elf-objdump -D -b binary -m riscv --adjust-vma=0x40000000 boot_rom_v0.4.bin > boot_rom_v0.4.bin.asm
```

From the disassembly result, it looks like a valid RISC-V binary. A snippet is shown below. However, I'm stopping here. The information currently known should be sufficient for me to start direct boot bare-metal programming; there's no need to delve deeper into the boot ROM implementation.

```
0000000040000000 <.data>:
    40000000:	6910106f          	j	0x40001e90
	...
    40000010:	0005                	c.nop	1
    40000012:	0000                	unimp
    40000014:	00000003          	lb	zero,0(zero) # 0x0
    40000018:	2774806f          	j	0x40048a8e
    4000001c:	20b4806f          	j	0x40048a26
    40000020:	23d4806f          	j	0x40048a5c
    40000024:	27f4806f          	j	0x40048aa2
    40000028:	2f54806f          	j	0x40048b1c
    4000002c:	36f4806f          	j	0x40048b9a
    40000030:	3bb4706f          	j	0x40047bea
    40000034:	2cf4806f          	j	0x40048b02
    40000038:	27b4806f          	j	0x40048ab2
	...
    40001e8e:	0000                	unimp
    40001e90:	30001073          	csrw	mstatus,zero
    40001e94:	00001073          	csrw	0x0,zero
    40001e98:	600c22b7          	lui	t0,0x600c2
    40001e9c:	10428293          	addi	t0,t0,260 # 0x600c2104
    40001ea0:	0002a023          	sw	zero,0(t0)
    40001ea4:	4285                	li	t0,1
    40001ea6:	80029073          	csrw	0x800,t0
    40001eaa:	4285                	li	t0,1
    40001eac:	80129073          	csrw	0x801,t0
    40001eb0:	4181                	li	gp,0
    40001eb2:	00000297          	auipc	t0,0x0
    40001eb6:	e4e28293          	addi	t0,t0,-434 # 0x40001d00
    40001eba:	0012e293          	ori	t0,t0,1
    40001ebe:	30529073          	csrw	mtvec,t0
    40001ec2:	4305                	li	t1,1
    40001ec4:	600c22b7          	lui	t0,0x600c2
    40001ec8:	19428293          	addi	t0,t0,404 # 0x600c2194
    40001ecc:	0062a023          	sw	t1,0(t0)
    40001ed0:	42a5                	li	t0,9
    40001ed2:	4305                	li	t1,1
    40001ed4:	30029073          	csrw	mstatus,t0
    40001ed8:	00031073          	csrw	0x0,t1
    40001edc:	ffcdd117          	auipc	sp,0xffcdd
    40001ee0:	83410113          	addi	sp,sp,-1996 # 0x3fcde710
    40001ee4:	00057597          	auipc	a1,0x57
    40001ee8:	31c58593          	addi	a1,a1,796 # 0x40059200
    40001eec:	00057617          	auipc	a2,0x57
    40001ef0:	51460613          	addi	a2,a2,1300 # 0x40059400
    40001ef4:	02c5d163          	bge	a1,a2,0x40001f16
    40001ef8:	4194                	lw	a3,0(a1)
    40001efa:	41d8                	lw	a4,4(a1)
    40001efc:	459c                	lw	a5,8(a1)
    40001efe:	a039                	j	0x40001f0c
    40001f00:	0007a283          	lw	t0,0(a5)
    40001f04:	0056a023          	sw	t0,0(a3)
    40001f08:	0691                	addi	a3,a3,4
    40001f0a:	0791                	addi	a5,a5,4
    40001f0c:	fee6eae3          	bltu	a3,a4,0x40001f00
    40001f10:	05c1                	addi	a1,a1,16
    40001f12:	fec5e3e3          	bltu	a1,a2,0x40001ef8
    40001f16:	00057597          	auipc	a1,0x57
    40001f1a:	4fa58593          	addi	a1,a1,1274 # 0x40059410
    40001f1e:	00057617          	auipc	a2,0x57
    40001f22:	67260613          	addi	a2,a2,1650 # 0x40059590
    40001f26:	00c5dd63          	bge	a1,a2,0x40001f40
    40001f2a:	4194                	lw	a3,0(a1)
    40001f2c:	41d8                	lw	a4,4(a1)
    40001f2e:	a021                	j	0x40001f36
    40001f30:	0006a023          	sw	zero,0(a3)
    40001f34:	0691                	addi	a3,a3,4
    40001f36:	fee6cde3          	blt	a3,a4,0x40001f30
    40001f3a:	05b1                	addi	a1,a1,12
    40001f3c:	fec5e7e3          	bltu	a1,a2,0x40001f2a
    40001f40:	6fd4506f          	j	0x40047e3c
	...
```

By the way, one can check file formats (`-b`) and architectures (`-m`) supported by objdump with the `-i` (`--info`) option: 

```
$ riscv-none-elf-objdump -i
BFD header file version (xPack GNU RISC-V Embedded GCC x86_64) 2.45
...
binary
 (header endianness unknown, data endianness unknown)
  riscv
...
```
