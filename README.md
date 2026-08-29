# debian-macbook-2012-fixes
Definitive troubleshooting guide for Debian 13, SDDM, and internal keyboard failures on 2012 Intel MacBook Pros

# 🍏 Debian 13 + SDDM on 2012 Intel MacBook Pro: The Definitive Internal Keyboard Bug & Workaround Guide

A comprehensive technical documentation report mapping out a hyper-specific hardware initialization race condition on **Mid/Late 2012 Intel MacBook Pros** running **Debian 13 (Trixie)** with **KDE Plasma and SDDM**, detailing why standard fixes fail and how to successfully bypass the login barrier.

---

## 📋 Table of Contents
1. [Affected Hardware & Software](#-affected-hardware--software)
2. [The Symptom](#-the-symptom)
3. [Root Cause Analysis](#-root-cause-analysis)
4. [Exhausted Remediation Steps (Why Standard Fixes Fail)](#-exhausted-remediation-steps-why-standard-fixes-fail)
  * [Attempt 1: Forcing `hid_apple` into Initramfs](#attempt-1-forcing-hid_apple-into-initramfs)
  * [Attempt 2: Systemd Dependency Delay (`systemd-udev-settle`)](#attempt-2-systemd-dependency-delay-systemd-udev-settle)
5. [The Permanent Workaround: Auto-Login](#-the-permanent-workaround-auto-login)
  * [Method A: The Command-Line Way](#method-a-the-command-line-way)
  * [Method B: The Graphical Way (KDE Plasma Settings)](#method-b-the-graphical-way-kde-plasma-settings)

---

## 💻 Affected Hardware & Software

* **Hardware:** Apple MacBook Pro (Mid/Late 2012, Intel Architecture, e.g., `MacBookPro9.1` / `MacBookPro9.2`)
* **Operating System:** Debian 13 (Trixie)
* **Desktop Environment:** KDE Plasma
* **Display Manager:** SDDM (Simple Desktop Display Manager)
* **Bootloader:** GRUB

---

## 🔍 The Symptom

Upon booting the machine:
* **What works:** The built-in keyboard and trackpad function perfectly inside **GRUB**, in **emergency/TTY modes**, and immediately post-login. An external USB keyboard also works anywhere.
* **What fails:** The exact second the graphical display manager (**SDDM**) loads the login prompt, the internal keyboard and trackpad inputs go entirely dead (ignoring all keystrokes).

---

## ⚙️ Root Cause Analysis

The issue stems from an unresolvable initialization race condition and architecture mismatch at the display manager layer:
1. **The Apple USB Bus Topology:** On 2012 MacBooks, the internal keyboard and trackpad are routed through a legacy, proprietary internal USB hub controller (`idVendor=05ac`).
2. **SDDM / Libinput Early Polling:** Modern display managers initialize their input stack (`libinput`) asynchronously and aggressively fast upon reaching graphical targets.
3. **The Timing Mismatch:** SDDM polls input devices a split-second *before* the Linux kernel's `usbhid` subsystem has finished establishing a bound device node communication pipe with the legacy Apple internal USB hub. Because the login manager completely misses the hardware registration window, the input stream is dropped at the prompt. Once a desktop session forces a secondary re-probe, the hardware wakes up.

---

## ❌ Exhausted Remediation Steps (Why They Fail)

If you are facing this exact bug, save your time—the two most common online community suggestions **will not work**:

### Attempt 1: Forcing `hid_apple` into Initramfs
* **The Action:**
 1. Adding `hid_apple` as a raw line inside `/etc/initramfs-tools/modules`.
 2. Executing `sudo update-initramfs -u` to compile it directly into the early-boot RAM disk image.
* **Why it Fails:** The bottleneck is *not* a missing kernel module. The module loads successfully, but the higher-level USB device enumeration socket binding and SDDM's graphical input polling still outpace the legacy hardware hub's response time.

### Attempt 2: Systemd Dependency Delay (`systemd-udev-settle`)
* **The Action:**
 1. Creating a systemd override directory via `sudo mkdir -p /etc/systemd/system/sddm.service.d`.
 2. Writing a configuration file (`/etc/systemd/system/sddm.service.d/wait-for-input.conf`) forcing SDDM to explicitly wait for `systemd-udev-settle.service` or hardware device settling flags before launching.
* **Why it Fails:** Modern systemd configurations deprecate or bypass aggressive device settling to maximize boot speeds. Even when forced, SDDM's internal input polling loop still fails to establish a hook with the specific legacy Apple USB descriptor during that tight initialization window.

---

## ✅ The Permanent Workaround: Auto-Login

Because low-level hardware communication at the display manager login layer cannot natively reconcile with 2012 Apple controller polling under modern display stacks, the only reliable solution is an architectural bypass via **Auto-Login**. This allows GRUB and the kernel to complete their boot sequence and hands control directly to KDE Plasma, where the input subsystem fully refreshes and natively claims the hardware.

You can configure this using either the command line or the graphical settings menu:


### Method A: The Command-Line Way
1. Create the configuration directory for SDDM overrides:
  ```bash
  sudo mkdir -p /etc/sddm.conf.d
  ```
2. Create and open an autologin configuration file using a text editor (like nano):  
```bash
sudo nano /etc/sddm.conf.d/autologin.conf
```
3. Paste the following configuration block (make sure to replace your-username with your actual Debian login username):
```bash
[Autologin]
User=your-username
Session=plasma
```
4. Save and exit (In nano: press Ctrl+O, hit Enter, then press Ctrl+X).  
Reboot your system:
```bash
sudo reboot
```
---

### Method B: The Graphical Way (KDE Plasma Settings)
**Note: Since your internal keyboard works fine once you are logged in (or if you temporarily plug in a cheap USB mouse/keyboard to navigate), you can set this up directly through the GUI:**

1. Open **System Settings** from your application menu.  
2. Scroll down on the left sidebar to **Startup and Shutdown**.  
3. Click on **Login Screen (SDDM)**.  
4. Click the **"Behavior"** tab at the top.  
5. Check the box for **"Automatically log in"**.  
6. Select your user account from the dropdown menu and ensure the session type is set to **Plasma**.  
7. Click **Apply** in the bottom-right corner (it will prompt you for your administrator password).  
8. Reboot your Mac—you will now bypass the broken login screen completely and drop straight into a fully operational desktop with a working keyboard.
---
