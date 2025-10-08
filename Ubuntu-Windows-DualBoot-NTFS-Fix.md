---

# Ubuntu & Windows Dual-Boot – NTFS Mount Fix

## 🛑 Problem

When booting into Ubuntu, you might see this error when accessing Windows files:

```
Error mounting /dev/sda2: wrong fs type, bad option, bad superblock...
```

**Cause:**
Windows didn’t fully shut down or is using Fast Startup / Hibernation. This “locks” the NTFS partition, and Ubuntu won’t mount it to prevent data corruption.

---

## ✅ Permanent Fix (Do This Once in Windows)

1. **Disable Hibernation**

   ```cmd
   powercfg /h off
   ```

   * Opens CMD as Administrator and disables hibernation and Fast Startup.

2. **Run Disk Check (Recommended)**

   ```cmd
   chkdsk C: /f
   ```

   * If prompted to schedule a check on next restart, type `Y`.
   * Restart Windows and let it scan & repair the drive.

---

## ✅ Daily Use – Safe Switching Between Windows & Ubuntu

1. In **Windows**:

   * Always shut down completely:

     * Start Menu → Power → Shut Down
     * OR Alt + F4 on desktop → Shut Down
     * OR Hold **Shift** while clicking Shut Down (extra safe)

2. In **Ubuntu**:

   * You can access your Windows partition safely via the file manager (e.g., `Other Locations` → `425 GB Volume`).

---

## 🛠 Emergency Recovery in Ubuntu

1. Open terminal.
2. Run:

```bash
sudo ntfsfix /dev/sda2
```

* Clears leftover NTFS “dirty” flags.

3. Mount the partition:

```bash
sudo mkdir -p /mnt/windows
sudo mount -t ntfs-3g /dev/sda2 /mnt/windows
ls /mnt/windows
```

---

## 🧠 Tips & Best Practices

* Never **Sleep** or **Hibernate** Windows before booting Ubuntu.
* Always use **proper shutdown**.
* If the partition still won’t mount, boot into Windows and repeat the disk check (`chkdsk /f`).

---
