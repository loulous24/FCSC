# Sous-marin
Write-up by *loulous24* **CC-BY-NC-SA 3.0**

***HARDWARE*** challenge

## At the dock

Sous-marin, submarine in English, is a hardware challenge of the FCSC 2023. Let's take a look at the description (in French).

```
Vous êtes invité par un collègue à venir tester son nouveau modèle miniature de
sous-marin. Ce modèle est équipé d'une caméra « vue à la première personne » et
d'un système adéquat pour diffuser ce flux vidéo.

Pour vous appâter, votre collègue précise que sa carte embarquée est basée sur
**un cœur Risc-V**. Vous acceptez donc immédiatement. Le soir venu, vous passez
plusieurs minutes à explorer les fonds marins du bassin du campus.
Néanmoins, au bout d'une demi-heure, le système de supervision panique et vous
perdez la communication. Vous vous jetez dans le bassin pour récupérer
le modèle. Votre collègue voit au loin des étudiants partir en courant avec du
matériel radio en main. Mais que s'est-il passé ? Est-ce que ces étudiants
auraient compromis le système du sous-marin à distance ? Vous proposez d'aider
votre collègue à extraire le contenu de sa mémoire Flash et à l'analyser.

Le lendemain, après une douche et une courte nuit de repos, votre collègue
dépose la carte électronique du sous-marin sur votre bureau.
Il vous confirme qu'il a implémenté
[un port série en mode 8-N-1](https://en.wikipedia.org/wiki/8-N-1), mais il
précise qu'il a potentiellement fait **une erreur d'implémentation dans la
logique du port série synthétisé**, mais il ne s'en souvient plus trop.

Parce que le sous-marin n'est pas sous l'eau, son électronique n'est pas
correctement refroidie. Vous ne pouvez pas le garder allumé plus que quelques
minutes avant qu'il ne s'éteigne par sécurité.

Vous prenez un adaptateur port série vers USB pour vous connecter sur le système
et vous commencez à investiguer...

**Pour s'interfacer avec le port série à distance, vous aurez besoin de Telnet.**

`HOST:PORT` : `challenges.france-cybersecurity-challenge.fr:2303`

Votre collègue a réussi à retrouver une sauvegarde du chargeur de démarrage
`bootloader.bin` et vous indique que vous pouvez émuler la séquence de démarrage
avec la commande `qemu-system-riscv64 -M sifive_u -m 45M -kernel bootloader.bin`
(port série disponible dans `View > Serial 0`). Cela vous permet de prototyper
des idées avant de risquer d'abîmer le sous-marin.

SHA256(`bootloader.bin`) = `e06c7b272736c0d34617e9f62fd4e4c1a8d56d6df6e4f1ee83492999c4a65e6c`.
```

## Casting off the moorings

So is the challenge really about Submarine ? Perhaps taking a beer with a german friend can help you find something about [U-Boot](https://en.wikipedia.org/wiki/Das_U-Boot).
U-Boot is a bootloader which is a piece of software that is used in embedded devices for booting.

A bootloader is include in the ROM memory and is launched to perfom the loading of a more complicated software, such as a kernel for exemple. It copies it into the memory and gives the execution flow to it. It also can perform checks to detect abuses of the process.

So here, we have the binary of the bootloader, let's try to understand what it does and keep into your mind that the serial port can be broken.

## Loading the bootloader

To load the bootloader, the command given with qemu is useful but you can use this one if you want to do everything in your terminal (after a proper install of qemu)

```
qemu-system-riscv64 -M sifive_u -m 45M -kernel bootloader.bin -display none -serial stdio
```

We can see that opensbi is used as an intermediary between the hypervisor and the bootloader. It is not important for our challenge but it is another layout of abstraction.

The bootloader tries to load the kernel but it does not work.

To interact with the bootloader, some commands are userful as described [in the documentation](https://u-boot.readthedocs.io/en/latest/). The first one is `help` to have the list and a short description of the other one.

`env print -a` is very useful to understand what happens.
```
arch=riscv
baudrate=115200
board=unleashed
board_name=unleashed
bootargs=console=ttyS0 autoplay=https://youtu.be/m2uTFF_3MaA
bootcmd=sf probe; echo [+] Copying kernel from flash, please wait...; sf read $kernel_addr_r 0 9bc8b; echo [+] Decompressing and booting kernel...; booti $kernel_addr_r - $fdtcontroladdr
bootdelay=2
cpu=fu540
ethaddr=70:b3:d5:92:f0:01
fdt_addr=824907b0
fdt_addr_r=0x8c000000
fdt_high=0xffffffffffffffff
fdtaddr=824907b0
fdtcontroladdr=824907b0
fdtfile=sifive/hifive-unleashed-a00.dtb
initrd_high=0xffffffffffffffff
kernel_addr_r=0x80d00000
kernel_comp_addr_r=0x81800000
kernel_comp_size=0xb00000
loadaddr=0x80200000
preboot=setenv fdt_addr ${fdtcontroladdr};fdt addr ${fdtcontroladdr};
serial#=00000001
stderr=serial@10010000
stdin=serial@10010000
stdout=serial@10010000
vendor=sifive
```

Among the parameter, the `bootcmd`, the `preboot` commands are interesting. The `bootargs` is a nice song from the Beatles. Take a listen if you do not know it, it is nice.

`preboot` set something about `fdt` and the `bootcmd` is doing the copy and the looding of the kernel.

`fdt` is called `flattened device tree` and is useful for describing the different component of a system without complex encoding. If you execute `fdt print` command, it will show the devices connected to the bootloader.

`sf` stands for [SPI](https://en.wikipedia.org/wiki/Serial_Peripheral_Interface) Flash. SPI is an old serial synchronous interface with a master-slave architecture.

Here it is used to initialise the SPI with `sf probe` and copy the data of the SPI onto the memory with `sf read`.

After that, the kernel is booted with `booti`.

So what is the problem ? If you try to see what is inside the SPI, you will find an interesting thing.

```
sf probe
sf read $kernel_addr_r 0 9bc8b
md 0x80d00000 1000
```

It will print only `0xff` because there is no kernel is the `SPI`. It is time to move to the real submarine. It has to be noted that for now, no problem with the serial communication has shown. So this is not a problem of the implementation of the serial in the bootloader probably.

## Diving into the ocean

Now it is time to use the broken submarine. After having a lot of troubles with the Proof-of-Work of the challenge, I launched the given python script given for the pow.

And you can see in the logbook that it works, the kernel is sending something to us but a reboot is performed at the end after having printing that we will never get the flag.

### Calibration of the periscope

This is were the troubles begin. I tried to use the [Telnet python library](https://docs.python.org/3/library/telnetlib.html) given and do a call to Telnet.interact but it does not print anything (this is because the endlines are `b'\r\r'`)  and when trying to write something, it gives us an error about `telnetlib`. Even if it is a module in the standard, it is broken and deprecated.

As I have only one tool, I try to make `pwnlib.remote` working. Unfortunately, if you try to connect directly, it detects that you are not using Telnet.

```
b'\xff\xfd"\xff\xfa"\x01\x00\xff\xf0\xff\xfb\x01\rPlease use Telnet.\r\n'
```

I do not want to use Telnet but it is a basic protocol, I will try to mimick it. I launch [Wireshark](https://www.wireshark.org/), a marvellous tool for doing packet sniffing. I see that the beginning of a connection is always the same and after, it is only raw data.

The server begins with the message of hex `ff fd 22 ff fa 22 01 00 ff f0 ff fb 01` and the client replies `ff fc 22` and `ff fe 01`.

The Telnet documentation is [here](https://www.rfc-editor.org/rfc/rfc854.html), it is described that the `0xff` byte is to be interpreted as the beginning of a command. `0xfe` is `DO` code, according to [this](https://users.cs.cf.ac.uk/Dave.Marshall/Internet/node141.html), `0x22` is used for specifying the linemode. `0xfa` is for subnegotiation of the parameter `0x22` with parameter `0x0100`. The `0xf0` is for ending the subnegotiation. `0xfb` is `WILL` code, `0x01` here means echo so the server wants to begin echo. The client explains that it `WONT` (code `0xfc`) applies the `linemode` parameter and that it `DONT` (code `0xfe`) wants that the server performs echo.

To make it shorter, if we send the command that the client has sent, we won't get annoyed by telnet anymore with a classic TCP socket (like `pwnlib.remote`).

```python3
from pwn import args, log, remote

HOST = "challenges.france-cybersecurity-challenge.fr"
PORT = 2303

io = remote(HOST, PORT)
io.readuntil(bytes.fromhex("fffd22fffa220100fff0fffb01"))
io.send(bytes.fromhex("fffc22"))
io.send(bytes.fromhex("fffe01"))
```

And to send a key before autobooting.

```
while True:
	r = io.readuntil(b"\n")
	print(r)
	if r == b'Err:   serial@10010000\r\n':
		io.readuntil(b"Hit any key to stop autoboot:")
		io.write(b"\r\n")
		break
io.readuntil(b"\r\n=> ")
```

### Damage management

This is finally were we found that the submarine is broken and the serial connection does not work. Because when you send something to the server, it seems to not do anything. In fact it sends your data back but it seems to be random.

![Lost Connection](./wu_files/submarine_lost.jpg)

After several tests, I have found that the downlink serial connection is working properly but the uplink is not working well and messes our bytes up. And when sending a character, it will be echoed back. This code is useful to get a map of the bytes that we send and the bytes received.

```python3
serial_change = {}
for i in range(256):
	if i in {3}:
		continue
	byteI = bytes((i,))
	io.send(byteI)
	s = io.read(timeout=0.5)
	serial_change[byteI] = s
print(serial_change)
print(*map(lambda x: f"{x[0][0]:08b}" + f" {x[1][0]:08b}" if len(x[1]) > 0 else "", serial_change.items()), sep='\n')
```

The byte 3 is messing everything up. This is why I remove it. This is only when I was writing this write-up that I figure that the transformation fonction was `~(x ror 1)` which means that if bits sent are `b7b6b5b4b3b2b1b0`, the bits received are `b̃0b̃7b̃6b̃5b̃4b̃3b̃2b̃1` where `b̃` is the opposite bit. During the CTF, I used a more complex function below for the translation between my commands and what should be sent to the server.

```
def convert_serial(s):
    return bytes(map(lambda x: 1+(127-x)*2, s))
```

### Spying on the enemy

Now that everything is set up, we can try to extract the linux kernel. I used the command `md.q` to get an hexdump the parameter are the address and the number of bytes read.

```
io.send(convert_serial(b"setenv fdt_addr ${fdtcontroladdr};fdt addr ${fdtcontroladdr};sf probe;sf read $kernel_addr_r 0 9bc8b;md.b $kernel_addr_r 9bc8b\n"))

io.readuntil(b'Read: OK\r\n')
with open("kernel_dump.dmp", "ab") as f:
    while True:
        f.write(io.readline().replace(b'\r\n', b'\n'))
```

Unfortunately, the connection timed out before the end. So I do it again from the address of the last bytes read before the timeout.

Now, a hexdump of the kernel is extracted. Use the following command to get the kernel.

```sh
xxd -r --seek -0x80d00000 kernel_dump.dmp > kernel.img
```

## Infiltrating the enemy base

Now that we have the kernel, running `file kernel.img` on it says that it is a gzip compressed file. With `7z x kernel.img`, we extract the kernel.

I know that the linux kernel is here to set up the hardware components, mounting the file system but when it finishes the boot, it gives the control to the `/init` process. I was thinking that the init file was stored in a file system on an external hardware peripheral like the `SPI` or another thing that could have be connected to the qemu on the server (MMC, external flash...).

However, I did not find any external file system. The file was embedded inside the kernel. [This question](https://unix.stackexchange.com/questions/163346/why-is-it-that-my-initrd-only-has-one-directory-namely-kernel) helps me understanding this and I used binwalk to get it.

Using binwalk with `binwalk -e kernel` gives a CPIO archive. Doing it again on the extracted CPIO will give the `init` file.

## Sinking the ship

![U See the flag ?](./wu_files/flag_stealing.jpeg)

For the last step, I use [Ghidra](https://ghidra-sre.org/).

The main function is

```C
void main(void) {
  puts("Muahahaha, you won\'t get my flag!");
  reboot(0x4321fedc);
}
```

It explains the previous behaviour of the kernel. So where FLAG ? Another interesting function is defined but not called.

```C
void print_secret(void) {
  printf("FCSC{n3X7_t1M3_Us3_a_h4rDw4R3_1nv3R53r_37438181892ff90d}");
  return;
}
```

It is also possible to use `strings` to see the result.

## Back to earth

So this was a really nice challenge ! A big thanks to `erdnaxe` for the challenge (and to the organisers).

To sum up, I would leave this to keep in mind the boot process.

```
Boot firmware (OpenSBI and the Hypervisor level)
|||||||||||||
vvvvvvvvvvvvv
   U-Boot     (Initializes the SPI, the device tree and boot Linux)
   ||||||    <------ SPI Flash memory 
   vvvvvv
Linux kernel  (Mounts the file system and runs init)
||||||||||||
vvvvvvvvvvvv
   /init      (Starts the services)
```

## Great harbours to get information

- [A useful question to explain why using binwalk](https://unix.stackexchange.com/questions/163346/why-is-it-that-my-initrd-only-has-one-directory-namely-kernel)
- [List of kernel parameters for 5.15 kernel](https://www.kernel.org/doc/html/v5.15/admin-guide/kernel-parameters.html)
- [Explanation of what is a flattened device tree](https://stackoverflow.com/questions/21808984/what-is-the-use-of-flattened-device-tree-linux-kernel)
- [Guide about Linux on U-boot boot process](https://www.embeddedartists.com/wp-content/uploads/2019/03/iMX_Working_with_Linux_and_uboot.pdf)
- [The U-Boot documentation](https://u-boot.readthedocs.io/en/latest/)
- [Telnet protocol specification](https://www.rfc-editor.org/rfc/rfc854.html)
- [Telnet protocol explanation](https://users.cs.cf.ac.uk/Dave.Marshall/Internet/node141.html)
- [A possible playground for U-boot with QEMU](https://tech.io/playgrounds/84444/running-u-boot-linux-kernel-in-qemu/prologue)


## Logbook of the captain

Linux kernel message

```
[    0.000000] Linux version 5.15.90 (fcsc@fcsc) (riscv64-unknown-linux-gnu-gcc (GCC) 11.3.0, GNU ld (GNU Binutils) 2.39) #1 Wed Feb 1 11:36:50 CET 2023
[    0.000000] OF: fdt: Ignoring memory range 0x80000000 - 0x80200000
[    0.000000] Machine model: SiFive HiFive Unleashed A00
[    0.000000] Forcing kernel command line to:  
[    0.000000] Zone ranges:
[    0.000000]   DMA32    [mem 0x0000000080200000-0x0000000082cfffff]
[    0.000000]   Normal   empty
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000080200000-0x0000000082cfffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000080200000-0x0000000082cfffff]
[    0.000000] SBI specification v0.3 detected
[    0.000000] SBI implementation ID=0x1 Version=0x10000
[    0.000000] SBI TIME extension detected
[    0.000000] SBI IPI extension detected
[    0.000000] SBI RFENCE extension detected
[    0.000000] riscv: ISA extensions acdefimnrs
[    0.000000] riscv: ELF capabilities acdfim
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 10857
[    0.000000] Kernel command line:  
[    0.000000] Dentry cache hash table entries: 8192 (order: 4, 65536 bytes, linear)
[    0.000000] Inode-cache hash table entries: 4096 (order: 3, 32768 bytes, linear)
[    0.000000] mem auto-init: stack:off, heap alloc:off, heap free:off
[    0.000000] Memory: 30824K/44032K available (688K kernel code, 4565K rwdata, 2048K rodata, 2112K init, 197K bss, 13208K reserved, 0K cma-reserved)
[    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
[    0.000000] riscv-intc: unable to find hart id for /reserved-memory/mmode_resv0@80000000/interrupt-controller
[    0.000000] riscv-intc: 64 local interrupts mapped
[    0.000000] plic: interrupt-controller@c000000: mapped 53 interrupts with 1 handlers for 3 contexts.
[    0.000000] riscv_timer_init_dt: Registering clocksource cpuid [0] hartid [1]
[    0.000000] clocksource: riscv_clocksource: mask: 0xffffffffffffffff max_cycles: 0x1d854df40, max_idle_ns: 3526361616960 ns
[    0.000224] sched_clock: 64 bits at 1000kHz, resolution 1000ns, wraps every 2199023255500ns
[    0.096081] Calibrating delay loop (skipped), value calculated using timer frequency.. 2.00 BogoMIPS (lpj=4000)
[    0.096248] pid_max: default: 4096 minimum: 301
[    0.097081] Mount-cache hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.097144] Mountpoint-cache hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.292614] ASID allocator using 16 bits (65536 entries)
[    0.299005] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.587751] clocksource: Switched to clocksource riscv_clocksource
[    0.690863] workingset: timestamp_bits=62 max_order=13 bucket_order=0
[    0.794257] 10010000.serial: ttySIF0 at MMIO 0x10010000 (irq = 1, base_baud = 4166666) is a SiFive UART v0
[    1.282216] printk: console [ttySIF0] enabled
[    1.283586] 10011000.serial: ttySIF1 at MMIO 0x10011000 (irq = 2, base_baud = 4166666) is a SiFive UART v0
[    1.496242] Freeing unused kernel image (initmem) memory: 2112K
[    1.497912] Run /init as init process
Muahahaha, you won't get my flag!
[    1.790295] reboot: System halted
```

Flag `FCSC{n3X7_t1M3_Us3_a_h4rDw4R3_1nv3R53r_37438181892ff90d}`
