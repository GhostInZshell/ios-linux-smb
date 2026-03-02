# iOS Files → Samba SMB share is “Read-only” (Browse works, writes disabled)

This guide documents a reproducible fix for a specific failure mode:

- iOS **Files** can connect to a Samba (SMB) share
- It can browse directories and list files
- But it **cannot create folders** or **copy/move files**
- iOS Files UI shows **“Read-only”** and disables **New Folder / Paste / Move**

It also includes the Wireshark/tcpdump workflow that made the root cause obvious.

---

## Environment (tested)

- **Server 1:** Raspberry Pi running **DietPi** 
- **Server 2:** **Pop!_OS** Linux PC 
- **Client:** iPhone running **iOS 26**
- **Debugging:** Wireshark on Linux laptop via `tcpdump` over SSH

Behavior was consistent across servers: same iOS Files “read-only” symptoms before the fix, full read/write after.

---

## Quick Fix
<details>
<summary><strong>Show Steps</strong></summary>

### 1) Install required Samba VFS modules (DietPi/minimal Debian especially)
On minimal installs, `fruit` / `streams_xattr` may not exist until you install the package.

```bash
sudo apt update
sudo apt install samba samba-vfs-modules
```

### 2) Add Apple-compatible VFS + enable extended attributes (xattrs)

At minimum you want:

- `vfs objects = fruit streams_xattr`
- `ea support = yes` (in each share)

### 3) Restart Samba
```
sudo systemctl restart smbd
sudo systemctl restart nmbd 2>/dev/null || true
```
### 4) Test from iOS Files

In Files → Browse → … → Connect to Server:

`smb://<SERVER_IP>/<SHARE_NAME>`

Verify:

- New Folder works
- Copy a file into the share
- Move it into a subfolder

If those work, you’re done.
</details>

---

## Key fixes discovered (what mattered)
### 1) Share name matching (treat as case-sensitive for iOS Files)

In practice, iOS Files behaved as if the share name had to match the `smb.conf` share header exactly.

Example:
- Share defined as `[SMB_SHARE]`
- iOS Files URL that worked: `smb://<SERVER_IP>/SMB_SHARE`

If the name didn’t match, the server returned `STATUS_BAD_NETWORK_NAME` and the connection/write flow failed.

> Note: SMB share names are often described as case-insensitive in general, but iOS Files behaved like exact matching mattered here. The practical advice is: match it _exactly_.

### 2) `samba-vfs-modules` missing on DietPi

DietPi ships a minimal Samba setup. Without `samba-vfs-modules`, `fruit` / `streams_xattr` support may not be available.
```
sudo apt install samba-vfs-modules
```

### 3) `ea support = yes` — the magic line

iOS Files attempts operations involving Apple metadata (e.g., xattrs like `com.apple.FinderInfo`).

Without `ea support = yes`, Samba returned errors like:

- `STATUS_OBJECT_NAME_NOT_FOUND` during xattr-related operations (ex: `*:com.apple.FinderInfo`)

Result: iOS Files treated the share as effectively read-only and disabled write UI.

**Recommendation:** Put `ea support = yes` in every iOS-targeted share section.

### 4) Folder permissions

Once Samba is configured correctly, the underlying filesystem must still allow writes.

Observations:

- `770` works with `guest ok = no` (when the Samba user owns the directory / correct group)
- `775` may help if you’re still seeing the iOS “Read-only” label (usually indicates user/group mismatch)

<details>
<summary><strong>Working smb.conf (DietPi)</strong></summary>
  
```
[global]
   workgroup = WORKGROUP
   server role = standalone server
   min protocol = SMB3
   server signing = disabled

   vfs objects = fruit streams_xattr
   fruit:metadata = stream
   fruit:aapl = yes

[SMB_SHARE]
   path = /home/smbtest/SMB_SHARE
   valid users = smbtest

   read only = no
   writable = yes
   guest ok = no

   ea support = yes

   force create mode = 0660
   force directory mode = 0770
```
</details>

<details>
  <summary><strong>Working smb.conf (Pop!_OS)</strong></summary>

```
[global]
   workgroup = WORKGROUP
   server role = standalone server

   vfs objects = fruit streams_xattr
   fruit:metadata = stream
   fruit:aapl = yes

[shared]
   path = /home/<user>/Mounts/smb_share
   browseable = yes

   read only = no
   writable = yes
   guest ok = no
   valid users = <user>

   ea support = yes
```
</details>

## Prerequisites / full setup
### DietPi
```
sudo apt update
sudo apt install samba samba-vfs-modules

sudo adduser smbtest
sudo smbpasswd -a smbtest

sudo mkdir -p /home/smbtest/SMB_SHARE
sudo chown -R smbtest:smbtest /home/smbtest/SMB_SHARE
sudo chmod 770 /home/smbtest/SMB_SHARE

sudo systemctl restart smbd
```

### Pop!_OS / Ubuntu
```
sudo apt update
sudo apt install samba samba-vfs-modules

sudo smbpasswd -a <username>
sudo systemctl restart smbd
```

### iOS connection instructions

Files app → Browse → (…) → Connect to Server

`smb://<SERVER_IP>/<SHARE_NAME>`

Credentials:

- **Username:** your Samba user (ex: smbtest)
- **Password:** the password set via smbpasswd

## Debugging workflow (Wireshark + tcpdump over SSH)
### Capture
```
ssh <server_user>@<SERVER_IP> \
  "sudo tcpdump -i <iface> -s 65535 -U -w - port 445" \
  > capture.pcap
```
### Open
```
wireshark capture.pcap
```

### Useful display filters

- `smb2` — SMB2/3 traffic
- `smb2.nt_status != 0` — errors only
- `ip.addr == <IOS_IP>` — isolate the iPhone

### Error codes encountered
| NTSTATUS | What it meant (in this case) | Fix |
| --- | --- | --- |
| `STATUS_BAD_NETWORK_NAME` | Share name in URL didn’t match the configured share | Match [share] name exactly in the URL | 
| `STATUS_OBJECT_NAME_NOT_FOUND` on `*:com.apple.FinderInfo` | xattr/ADS metadata ops failed | Install `samba-vfs-modules`; set `vfs objects = fruit streams_xattr`; set `ea support = yes` | 

## Postmortem / RCA (why iOS shows “Read-only”)

This failure mode looks like a basic permissions issue but isn’t.

## What happened:

- iOS Files initiated create/write flows
- Those flows included Apple metadata operations that map to xattrs/ADS
- Without the right VFS modules + EA support, Samba returned errors on those metadata operations
- iOS Files reacted by labeling the share “Read-only” and disabling write UI

## The fix wasn't "more chmod", it was:

- ensuring the server actually had the VFS modules installed (`samba-vfs-modules`)
- enabling Apple-compatible VFS objects (`fruit streams_xattr`)
- explicitly enabling EA support per share (`ea support = yes`)
- then verifying normal filesystem write permissions for the Samba user
