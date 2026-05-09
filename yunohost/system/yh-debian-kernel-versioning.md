# Understanding Debian Kernel Version Numbers

**Version:** May 2026  
**YunoHost:** 12.1.39 (stable)  
**Debian:** 12 Bookworm

---

## The Problem

When Debian patches a Linux kernel vulnerability, it publishes the patched versions in the [Debian Security Tracker](https://security-tracker.debian.org) as **package versions** — for example `6.1.170-3`.

The common command to check your kernel version is:

```bash
uname -r
```

Output:
```
6.1.0-47-amd64
```

Comparing `6.1.0-47` with `6.1.170-3` makes it look like the system is still vulnerable — but it is not. Both numbers describe the same thing, but speak a completely different language.

---

## Two Different Version Schemes

### Kernel Build Name (`uname -r`)

```
6.1.0-47-amd64
│   │  │   └── Architecture (amd64 = 64-bit)
│   │  └────── Build number (assigned by Debian)
│   └───────── Upstream kernel patch level
└───────────── Upstream kernel version (from the Linux project)
```

This number comes from a combination of the Linux project and Debian — it says nothing about the security status.

### Debian Package Version (`dpkg`)

```
6.1.170-3
│   │   └── Debian revision (how many times Debian revised the package)
│   └─────── Upstream kernel version
└─────────── Major kernel version
```

This is the official Debian package version — it appears in the Security Tracker and is the correct value to compare against.

---

## The Mapping

The connection between both numbers can be seen with:

```bash
dpkg -l | grep linux-image
```

Example output:

```
ii  linux-image-6.1.0-42-amd64    6.1.159-1    amd64    Linux 6.1 for 64-bit PCs (signed)
ii  linux-image-6.1.0-47-amd64    6.1.170-3    amd64    Linux 6.1 for 64-bit PCs (signed)
ii  linux-image-amd64              6.1.170-3    amd64    Linux for 64-bit PCs (meta-package)
```

| Kernel Build Name | Package Version |
|-------------------|----------------|
| `6.1.0-42-amd64` | `6.1.159-1` |
| `6.1.0-47-amd64` | `6.1.170-3` |

`6.1.0-47-amd64` = package `6.1.170-3` — they belong together.

---

## How to Check Correctly

Do not compare `uname -r` with the Security Tracker. Instead, check the package version directly:

```bash
apt-cache policy linux-image-amd64
```

Output:
```
linux-image-amd64:
  Installed: 6.1.170-3
  Candidate: 6.1.170-3
```

If `Installed` and `Candidate` are identical, the system is up to date. Compare this value with the Security Tracker — not `uname -r`.

---

## Why Does Nobody Explain This?

Debian does not document this mapping anywhere directly. It can only be read indirectly from the `dpkg` output. Security advisories — including from YunoHost — reference `uname -r` without pointing out this difference, which leads to unnecessary confusion.
