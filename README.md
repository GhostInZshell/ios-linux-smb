# iOS-Linux-SMB
For iOS Files Full Read/Write Samba share on Linux

## Who this is for
- iOS Files can connect to an SMB share
- iOS Files can browse folders / list files
- iOS Files _cannot_ **Paste / Move / New Folder**
- iOS Files shows **“Read-only”**
- If that’s you, this README is for you.

## Quick fix
- **Step 1:** Install required packages (critical on minimal installs)
  - Debian/Ubuntu/DietPi:
    - `sudo apt update`
    - `sudo apt install samba samba-vfs-modules`
- **Step 2:**  Use the working `smb.conf` example below
  - Must-haves:
    - `vfs objects = fruit streams_xattr`
    - `ea support = yes` (inside the share)
- **Step 3:**  Restart Samba
  - `sudo systemctl restart smbd`
- **Step 4:** Verify from iOS Files (success criteria)
  - Create a folder
  - Paste a file into the share
  - Move a file into a subfolder

## Working `smb.conf` example (Linux)
- Replace placeholders:
  - `<SHARE_NAME>`: the bracket name (e.g. `SMB_SHARE`)
  - `<LINUX_USER>`: the Samba user (e.g. `smbuser`)
  - `<SHARE_PATH>`: the share directory (e.g. `/srv/samba/SMB_SHARE`)
- Practical rule for iOS Files:
  - Treat `<SHARE_NAME>` as an _exact match_ with what you type in the iOS URL.

```ini
[global]
   workgroup = WORKGROUP
   server role = standalone server
   min protocol = SMB3
   server signing = disabled

   # Apple compatibility (often requires samba-vfs-modules on minimal distros)
   vfs objects = fruit streams_xattr
   fruit:metadata = stream
   fruit:aapl = yes

[<SHARE_NAME>]
   path = <SHARE_PATH>

   # Auth (recommended)
   valid users = <LINUX_USER>
   guest ok = no

   # Writable
   read only = no
   writable = yes

   # Critical for iOS Files write behavior in this issue
   ea support = yes

   # Optional guardrails
   force create mode = 0660
   force directory mode = 0770
```

## Full setup (Linux)
- **Step 1:** Install Samba + VFS modules
  - `sudo apt update`
  - `sudo apt install samba samba-vfs-modules`

- **Step 2:** Create (or reuse) a Linux user
  - `sudo adduser <LINUX_USER>`

- **Step 3:** Set the Samba password for that user
  - `sudo smbpasswd -a <LINUX_USER>`

- **Step 4:** Create the share directory and set ownership/permissions
  - `sudo mkdir -p <SHARE_PATH>`
  - `sudo chown -R <LINUX_USER>:<LINUX_USER> <SHARE_PATH>`
  - `sudo chmod 770 <SHARE_PATH>`

- **Step 5:** Edit config and validate
  - Edit: `/etc/samba/smb.conf`
  - Validate: `testparm -s`

- **Step 6:** Restart Samba
  - `sudo systemctl restart smbd`

## iOS connection (Files app)
  - Files → Browse → (…) → Connect to Server
  - `smb://<SERVER_IP>/<SHARE_NAME>`
  - **Username:** `<LINUX_USER>`
  - **Password:** (the one you set with `smbpasswd`)

## If it still shows “Read-only” (checklist)
- Share name match (this bit matters)
  - `[<SHARE_NAME>]` in `smb.conf` must match the URL segment exactly:
    - `smb://<SERVER_IP>/<SHARE_NAME>`
- Confirm VFS modules are installed
  - Debian-based:
    - `dpkg -l | grep samba-vfs-modules`
- Confirm `ea support = yes` is in the share section
- Confirm the Samba user can write to the directory
 - `sudo -u <LINUX_USER> bash -lc 'touch <SHARE_PATH>/.__write_test && rm <SHARE_PATH>/.__write_test'`
- Restart `smbd` after changes
  - `sudo systemctl restart smbd`

## What fixed it (high level)
- Installing `samba-vfs-modules` (minimal distros may not include Apple-related VFS modules by default)
- Enabling Apple-compatible VFS settings:
  - `vfs objects = fruit streams_xattr`
- Enabling extended attributes in the share:
  - `ea support = yes`
- Ensuring filesystem permissions actually allow writes for the Samba user
