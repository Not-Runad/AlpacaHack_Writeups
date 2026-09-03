# ■ Let's Shut Down Linux
[Let's Shut Down Linux - AlpacaHack](https://alpacahack.com/daily/challenges/lets-shut-down-linux)

## 1. Writeup
[kernel/reboot.c#L728-L729](https://github.com/torvalds/linux/blob/940de590b839f71d6dc846160534bf202401b8b7/kernel/reboot.c#L728-L729):
```C:kernel/reboot.c
SYSCALL_DEFINE4(reboot, int, magic1, int, magic2, unsigned int, cmd, void __user *, arg)
```

- rebootシステムコールは`magic1`, `magic2`, `cmd`を受け取る.

[kernel/reboot.c#L739-L745](https://github.com/torvalds/linux/blob/940de590b839f71d6dc846160534bf202401b8b7/kernel/reboot.c#L739-L745):
```C:kernel/reboot.c
	/* For safety, we require "magic" arguments. */
	if (magic1 != LINUX_REBOOT_MAGIC1 ||
			(magic2 != LINUX_REBOOT_MAGIC1 &&
			magic2 != LINUX_REBOOT_MAGIC2A &&
			magic2 != LINUX_REBOOT_MAGIC2B &&
			magic2 != LINUX_REBOOT_MAGIC2C))
		return -EINVAL;

```

- `magic1`, `magic2`は特定の数値(`LINUX_REBOOT_MAGIC1`, `LINUX_REBOOT_MAGIC1`, `LINUX_REBOOT_MAGIC2A`, `LINUX_REBOOT_MAGIC2B`, `LINUX_REBOOT_MAGIC2C`)に一致している必要がある.

[include/uapi/linux/reboot.h](https://github.com/torvalds/linux/blob/940de590b839f71d6dc846160534bf202401b8b7/include/uapi/linux/reboot.h):
```C:include/uapi/linux/reboot.h
/* SPDX-License-Identifier: GPL-2.0 WITH Linux-syscall-note */
#ifndef _UAPI_LINUX_REBOOT_H
#define _UAPI_LINUX_REBOOT_H

/*
 * Magic values required to use _reboot() system call.
 */

#define	LINUX_REBOOT_MAGIC1	    0xfee1dead
#define	LINUX_REBOOT_MAGIC2	    672274793
#define	LINUX_REBOOT_MAGIC2A	85072278
#define	LINUX_REBOOT_MAGIC2B	369367448
#define	LINUX_REBOOT_MAGIC2C	537993216


/*
 * Commands accepted by the _reboot() system call.
 *
 * RESTART     Restart system using default command and mode.
 * HALT        Stop OS and give system control to ROM monitor, if any.
 * CAD_ON      Ctrl-Alt-Del sequence causes RESTART command.
 * CAD_OFF     Ctrl-Alt-Del sequence sends SIGINT to init task.
 * POWER_OFF   Stop OS and remove all power from system, if possible.
 * RESTART2    Restart system using given command string.
 * SW_SUSPEND  Suspend system using software suspend if compiled in.
 * KEXEC       Restart system using a previously loaded Linux kernel
 */

#define	LINUX_REBOOT_CMD_RESTART	0x01234567
#define	LINUX_REBOOT_CMD_HALT		0xCDEF0123
#define	LINUX_REBOOT_CMD_CAD_ON		0x89ABCDEF
#define	LINUX_REBOOT_CMD_CAD_OFF	0x00000000
#define	LINUX_REBOOT_CMD_POWER_OFF	0x4321FEDC
#define	LINUX_REBOOT_CMD_RESTART2	0xA1B2C3D4
#define	LINUX_REBOOT_CMD_SW_SUSPEND	0xD000FCE2
#define	LINUX_REBOOT_CMD_KEXEC		0x45584543



#endif /* _UAPI_LINUX_REBOOT_H */
```

- 各マジックナンバーおよび`cmd`対応は`include/uapi/linux/reboot.h`に定義されている.
- `LINUX_REBOOT_MAGIC1`: `0xfee1dead`
- `LINUX_REBOOT_MAGIC2`: `672274793`
- `LINUX_REBOOT_MAGIC2A`: `85072278`
- `LINUX_REBOOT_MAGIC2B`: `369367448`
- `LINUX_REBOOT_MAGIC2C`: `537993216`
- `LINUX_REBOOT_CMD_POWER_OFF`: `0x4321FEDC`

---

これらをもとに`arg1`(`magic1`), `arg2`(`magic2`), `arg3`(`cmd`)を10進数の整数で渡す.

`solve.py`:
```python:solve.py
from pwn import *
import sys

HOST = "34.170.146.252"
PORT = 35059

context(os='linux', arch='amd64')
io = None

def convert(hex_num):
	return str(hex_num).encode()

def exploit():
	arg1 = 0xfee1dead
	arg2 = 672274793
	arg3 = 0x4321fedc
	io.recvuntil(b"arg1: ")
	io.sendline(convert(arg1))
	io.recvuntil(b"arg2: ")
	io.sendline(convert(arg2))
	io.recvuntil(b"arg3: ")
	io.sendline(convert(arg3))

if __name__ == "__main__":
	io = remote(HOST, PORT)
	exploit()
	io.interactive()
```

---
## 2. 蛇足

- `LINUX_REBOOT_MAGIC2`: `672274793` -> `0x28121969`
- `LINUX_REBOOT_MAGIC2A`: `85072278` -> `0x05121996`
- `LINUX_REBOOT_MAGIC2B`: `369367448` -> `0x16041998`
- `LINUX_REBOOT_MAGIC2C`: `537993216` -> `0x20112000`

`LINUX_REBOOT_MAGIC2`(28/12/1969)はLinus Torvaldsの誕生日で, ほかは娘たちの誕生日みたい.