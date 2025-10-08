# Postman UI Fix on Ubuntu (Squares Issue)  

Sometimes, when using Postman installed via **Snap on Ubuntu**, the file picker or other UI dialogs may display **squares (`[][][]`) instead of text**. This issue is caused by a corrupted or outdated **font cache** in Postman’s Snap configuration.  

This guide walks you through **safely fixing the problem** without affecting your collections, environments, or endpoints.  

> **Works on all Ubuntu versions** with Postman installed via Snap (tested on Ubuntu 24.04.2 LTS).  

---

## Steps to Fix the Issue

### 1️⃣ Locate Postman Snap Configuration

Postman Snap stores its data under:

```bash
~/snap/postman/current/.config/
````

* **fontconfig/** → contains font cache
* **Postman/** → contains your collections, environments, and saved requests (do not touch)

---

### 2️⃣ Backup the Font Cache (Optional but Recommended)

Before deleting, create a backup of the font cache:

```bash
cp -r ~/snap/postman/current/.config/fontconfig ~/snap/postman/current/.config/fontconfig_backup
```

* This ensures you can restore it if needed.

---

### 3️⃣ Delete the Corrupted Font Cache

```bash
rm -rf ~/snap/postman/current/.config/fontconfig
```

* This removes only the font cache, leaving your projects intact.

---

### 4️⃣ Restart Postman

1. Close Postman completely.
2. Reopen Postman — it will automatically rebuild the font cache.
3. File picker dialogs should now display text normally.

---

### 5️⃣ Restore Backup (Optional)

If anything goes wrong:

```bash
mv ~/snap/postman/current/.config/fontconfig_backup ~/snap/postman/current/.config/fontconfig
```

* Restart Postman after restoring.

---

## ✅ Check Your Versions (Optional)

It’s useful to know your Ubuntu and Postman versions to confirm compatibility:

```bash
# Check Ubuntu version
lsb_release -a

# Check Postman Snap version
snap list postman
```

**Example Output:**

```
Ubuntu 24.04.2 LTS
Postman 11.64.7
```

---

### Notes

* This fix **does not affect your collections, environments, or requests**.
* Only removes the cached font configuration causing UI issues.
* Works for Ubuntu Snap installs of Postman; paths may differ for other install methods.

```
