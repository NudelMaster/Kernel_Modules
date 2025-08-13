# 📬 Message Slot IPC Mechanism — Linux Kernel Module

A compact, production-style side project showcasing a **message-based IPC** implemented as a **Linux character device driver**. Multiple processes can exchange short messages via **per-device, per-channel** mailboxes with **atomic** reads/writes and **message persistence** (messages remain until overwritten).

> Built as part of an Operating Systems course at Tel Aviv University. Packaged and documented here as a portfolio-ready side project.

---

## ✨ Features

- **Character device driver** with fixed **major number `235`**
- Multiple **message slots** (device files, distinguished by **minor numbers**)
- Each slot supports many **channels** (user-selected via `ioctl`)
- **Persistent per-channel message** (up to **128 bytes**)
- **Atomic** `read(2)`/`write(2)` semantics
- Minimal userland tools: `message_sender` & `message_reader`

---

## 🧠 Concept & Design

A *message slot* is a pseudo-device (no hardware) represented by `/dev/slotX`. Each open file descriptor can **select a channel** with `ioctl`. Messages are written/read **per channel**:

- **Device file** → one *slot* (identified by minor number)
- **Channel** → mailbox within that slot
- **Message** → last written payload for that channel (overwritten on next write)

Memory usage scales with the number of used channels and the largest message size.

---

## 🛠 How It Works

### 1) Message Slot Devices

Use `mknod` to create device files managed by this driver:

~~~bash
sudo mknod /dev/slot0 c 235 0    # minor = 0
sudo mknod /dev/slot1 c 235 1    # minor = 1
sudo chmod 666 /dev/slot0 /dev/slot1
~~~

- **Major:** `235` (fixed by the assignment)
- **Minor:** identifies the specific device instance (slot)

---

### 2) File Operations

#### `ioctl(fd, MSG_SLOT_CHANNEL, channel_id)`
Selects the **active channel** for subsequent `read`/`write` on `fd`.

- `channel_id`: **non-zero** unsigned integer
- Errors:
  - `EINVAL` — unknown command or `channel_id == 0`

#### `write(fd, buf, len)`
Writes a **non-empty** message **(1–128 bytes)** to the selected channel. Entire message is written **atomically**; it **replaces** any previous message.

- Returns: number of bytes written (on success)
- Errors:
  - `EINVAL` — no channel selected yet
  - `EMSGSIZE` — `len == 0` or `len > 128`
  - Other errors (e.g., allocation failure) → `-1` with an appropriate errno

#### `read(fd, buf, len)`
Reads the **last written** message from the selected channel (atomic).

- Returns: number of bytes read (message length)
- Errors:
  - `EINVAL` — no channel selected yet
  - `EWOULDBLOCK` — no message exists on this channel yet
  - `ENOSPC` — `len` is smaller than stored message size
  - Other errors (e.g., allocation failure) → `-1` with an appropriate errno

> **Atomicity:** Successful `write` writes all bytes; successful `read` returns the **entire stored message** length.

---

## 📁 Project Structure

~~~
.
├─ message_slot.c       # Kernel module implementation
├─ message_slot.h       # Public kernel/user shared definitions (ioctl, constants)
├─ message_sender.c     # User-space message writer (CLI)
├─ message_reader.c     # User-space message reader (CLI)
└─ Makefile             # Builds kernel module and/or utilities (targets may vary)
~~~

---

## ⚙️ Build

### Build the kernel module (Kbuild style)
~~~bash
make           # expects a standard Kbuild Makefile pointing to your kernel headers
# produces message_slot.ko
~~~

### Build user-space tools
~~~bash
gcc -O3 -Wall -std=c11 message_sender.c -o message_sender
gcc -O3 -Wall -std=c11 message_reader.c -o message_reader
~~~

> Ensure kernel headers for your running kernel are installed (e.g., `linux-headers-$(uname -r)`).

---

## 🚀 Quick Start

1) **Load the module**
~~~bash
sudo insmod message_slot.ko
dmesg | tail
~~~

2) **Create a device file & set permissions**
~~~bash
sudo mknod /dev/slot0 c 235 0
sudo chmod 666 /dev/slot0
~~~

3) **Write a message to channel 7**
~~~bash
./message_sender /dev/slot0 7 "Hello, World!"
~~~

4) **Read the message from the same channel**
~~~bash
./message_reader /dev/slot0 7
# Output: Hello, World!
~~~

5) **Unload the module**
~~~bash
sudo rmmod message_slot
~~~

---

## 🧩 API & Header Snippets

In `message_slot.h` (example layout):

~~~c
#ifndef MESSAGE_SLOT_H
#define MESSAGE_SLOT_H

#include <linux/ioctl.h>

#define MSG_SLOT_MAJOR      235
#define MSG_SLOT_MAX_LEN    128

/* Choose a command number and direction; W = write arg from user to kernel */
#define MSG_SLOT_CHANNEL    _IOW(MSG_SLOT_MAJOR, 0, unsigned long)

#endif /* MESSAGE_SLOT_H */
~~~

> The exact `_IOW` macro usage can vary by design; above is a common pattern for passing an unsigned long channel id.

---

## 🧱 Design Notes

- **Per-file selected channel** is typically tracked via `file->private_data`.
- Use `iminor(inode)` to distinguish device instances (slots).
- Data structure: map **(minor, channel_id)** → **message buffer** (and size).
- **Memory complexity:** `O(C · M + F)`  
  - `C`: total channels used (across devices)  
  - `M`: size of the largest stored message  
  - `F`: number of currently open file objects
- Allocate with `kmalloc(..., GFP_KERNEL)` and **free** in:
  - `release` (per-open-file allocations)
  - module **exit** (global structures)
- Defensive checks on all user pointers and sizes.

---

## 🧪 Testing Ideas

- Multiple processes writing/reading different channels on the **same** device.
- Multiple device files (different minors) with overlapping channel IDs.
- Edge cases:
  - Write `len == 0` → `EMSGSIZE`
  - Read with small buffer → `ENOSPC`
  - Read before any write → `EWOULDBLOCK`
  - `ioctl` with `channel_id == 0` → `EINVAL`

---

## 🛟 Troubleshooting

- **`No such device`**: device file not created with correct major/minor → re-run `mknod`.
- **`Invalid argument` on read/write**: ensure you called `ioctl(..., MSG_SLOT_CHANNEL, ...)` first.
- **`Operation would block`** (`EWOULDBLOCK`): write a message to that channel before reading.
- **Kernel build errors**: verify kernel headers match your running kernel; check Kbuild paths in `Makefile`.

---

## 📄 License & Attribution

Educational project adapted and packaged for portfolio use. Original concept was developed for an Operating Systems course at **Tel Aviv University**.  
Use at your own risk; intended for learning and demonstration.

---

## 📚 What I Learned

- Character device drivers: `struct file_operations`, `open/read/write/unlocked_ioctl/release`
- Safe user-kernel data exchange & argument validation
- Device major/minor management & per-file state via `private_data`
- Atomic message semantics and per-channel storage design

