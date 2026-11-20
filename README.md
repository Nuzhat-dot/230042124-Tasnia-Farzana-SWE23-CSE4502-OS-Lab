# CSE 4502 Operating Systems Lab: mCertiKOS

**Student:** Tasnia Farzana (230042124), SWE 23

This repository holds my work for the CSE 4502 OS Lab. All labs use one
project, **mCertiKOS**, a small x86 teaching kernel. Each lab builds on the
code from the previous labs. The original course notes are in [`README`](README).
The lab sheets are in [`labs/`](labs/).

## Progress

| Lab | Topic | Status |
|---|---|---|
| 1 | Physical memory allocator | ✅ Done: 3 / 3 test groups pass |
| 2 | Memory containers and page tables | ✅ Done: 2 / 2 test groups pass |
| 3 | Virtual memory management | Not started |
| 4 | Kernel threads and scheduling | Not started |
| 5 | Processes, traps and system calls | Not started |
| 6 | Multiprocessor and synchronization | Not started |
| 7 | Disk and file system | Not started |

## Collaboration

I worked alone. All the code in this repository was written by me.

## Lab 1: Physical memory allocator

The kernel keeps an allocation table (`AT`) with one entry per 4 KB physical
page. Each entry records the page's permission and whether the page is
allocated.

**MATIntro** ([`kern/pmm/MATIntro/MATIntro.c`](kern/pmm/MATIntro/MATIntro.c))
- `at_is_norm` returns true when the page's permission is greater than 1,
  meaning the page is available to processes.
- `at_set_perm` sets a page's permission and marks it unallocated.
- `at_is_allocated` and `at_set_allocated` read and set the allocated flag.

**MATInit** ([`kern/pmm/MATInit/MATInit.c`](kern/pmm/MATInit/MATInit.c))
- `pmem_init` finds the highest address in the BIOS memory map and divides it
  by the page size to get the number of pages, `nps`.
- Pages below `VM_USERLO` or at or above `VM_USERHI` get permission 1
  (kernel only).
- Every other page gets permission 2 (normal) only when a usable range covers
  the whole page. Otherwise it gets permission 0, because part of the page
  may be reserved.

**MATOp** ([`kern/pmm/MATOp/MATOp.c`](kern/pmm/MATOp/MATOp.c))
- `palloc` searches only the user range of pages and returns the first normal
  page that is not allocated, or 0 when memory is full.
- It uses **memoization**: a static `next_page` remembers where the last
  search ended, so the next search starts there and wraps around once.
  A typical call checks only a few pages instead of scanning from the start.
- `pfree` marks the page as unallocated.
- Both functions keep their work between `mem_lock()` and `mem_unlock()`.

The optional parts (the `%o` printf format and the `backtrace` command) were
not done.

### Test result

```
Testing the MATIntro layer...  All tests passed.
Testing the MATInit layer...   All tests passed.
Testing the MATOp layer...     All tests passed.
LAB 1 RESULT: 3 / 3 test groups passed.
```

## Lab 2: Memory containers and page tables

**MContainer** ([`kern/pmm/MContainer/MContainer.c`](kern/pmm/MContainer/MContainer.c))

Each process has a container that limits how many physical pages it may use.
A container records its quota, its current usage, its parent and its number
of children.
- `container_init` counts every normal page that is not allocated. That count
  becomes the quota of the root container (process 0).
- The getters return the `parent`, `nchildren`, `quota` and `usage` fields.
- `container_can_consume` returns 1 only when `n` more pages still fit within
  the quota (`n <= quota - usage`).
- `container_split` creates a child container with the given quota and no
  usage. The child's quota is added to the parent's usage, and the parent's
  child count goes up by one.
- `container_alloc` calls `palloc` only when the quota allows one more page,
  and counts the page against the container's usage when it succeeds.
- `container_free` frees the page with `pfree` only when it is allocated, and
  lowers the usage.

**MPTIntro** ([`kern/vmm/MPTIntro/MPTIntro.c`](kern/vmm/MPTIntro/MPTIntro.c))

These functions work directly on the two-level x86 page tables. `PDirPool`
holds one page directory per process. `IDPTbl` is a page table set that maps
every virtual address to the same physical address (the identity map).
- `set_pdir_base` loads a process's page directory into `CR3`.
- `get_pdir_entry`, `set_pdir_entry` and `rmv_pdir_entry` read, set and
  clear a page directory entry. `set_pdir_entry` stores
  `(page_index << 12) | PT_PERM_PTU`.
- `set_pdir_entry_identity` points a page directory entry at the matching
  `IDPTbl` page table.
- `get_ptbl_entry`, `set_ptbl_entry` and `rmv_ptbl_entry` find the page table
  by masking off the low 12 permission bits of the page directory entry
  (`& 0xFFFFF000`). They then read, set or clear the entry at `pte_index`.
- `set_ptbl_entry_identity` sets an `IDPTbl` entry to
  `(pde_index << 22) | (pte_index << 12) | perm`, so the entry maps its
  virtual address to the same physical address.

### Test result

```
Testing the MContainer layer...  All tests passed.
Testing the MPTIntro layer...    All tests passed.
LAB 2 RESULT: 2 / 2 test groups passed.
```

## Building and running

The project builds on Linux or Windows through WSL 2. Install the tools with
`setup.sh` (see [`SETUP.md`](SETUP.md)).

```bash
make LAB=1 TEST=1            # build and run the Lab 1 tests
make LAB=2 TEST=1            # build and run the Lab 2 tests
make LAB=2 && make qemu-nox  # boot the kernel in the terminal
```

To exit QEMU, press **Ctrl-a**, then **x**.

To debug, run `make qemu-nox-gdb` in one terminal and `make gdb` in a second
one. QEMU waits for GDB on a TCP port based on your user ID
(`id -u % 5000 + 25000`), so several users on one machine do not clash. The
generated `.gdbinit` connects GDB to that port.
