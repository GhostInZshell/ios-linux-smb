# iOS Files ↔ Linux SMB Fix
iOS Files Full Read/Write Samba on Linux

# About The Project
I ran into isses when setting up a local Samba (SMB) share on Linux that works with iOS Files. I could 'read' existing files and traverse sub-directories in the share just fine but could never 'write' new ones to it. Anytime I tried move or copy files to the share (or even just create new sub-directories), the options would always be grayed out in the Files app and would always show "Read-only" label at the bottom. I checked POSIX perms, file owners, smb.conf, removed/re-added the share countless times. I checked multiple blogs, looking for an all-in-one solution but couldn't find anything that worked. For the life of me I just couldn't figure it out.

Until now.

## What I Figured Out
1. **The Share name IS Case Sensitive**: make sure whatever name you have for your share in smb.conf is _exactly_ the same as the one you put in the smb url. For example, if your share is [SMB_SHARE] in smb.conf file, in the url you put `smb://<server-ip-address>/SMB_SHARE`; `smb_share` or anything else won't work.
2. **samba-vfs-modules Missing**: many forums I read online said to add things like `vfs objects = streams_xattr` to the smb.conf but these won't work if `samba-vfs-modules` is not installed (and isn't by default on distros like DietPi).
3. **ea support = yes** in config: without this in share section, Samba will return `STATUS_OBJECT_NAME_NOT_FOUND` (verified during pcap).
4. **Folder Perms**: `775` is needed if iOS shows "Read-only" label. Otherwise if you have `guest ok = no` in your share section, `770` will work.

## Enviroments I Tested In
**Servers**
- Raspberry Pi 3+ running DietPi (Debian 12)
- Pop!_OS 22.04 LTS
- Ubuntu 24.04.4 LTS

**Client**
- iPhone iOS 26.3

# Getting Started

## Setup
### Step 1. Install samba + samba-vfs-modules
   
  `sudo apt update && sudo apt install samba samba-vfs-modules`
  
### Step 2. Create your SMB user
  
  You can use whatever method you like, but I used:
  ```
  sudo adduser --no-create-home smbuser
  sudo smbpasswd -a smbuser
  ```

### Step 3. Create your share directory
  You can name it whatever you like, but I'm a simple guy, so I just went with "share":

  ```
  sudo mkdir -p /home/smbuser/share
  sudo chown -R smbuser:smbuser /home/smbuser/share
  ```

### Step 4. Edit `/etc/samba/smb.conf`

  This is the part the I struggled with the most. The exact settings will depend on your enviroment but at a bare minimum, you should have something like this:
  ```
  [global]
     workgroup = WORKGROUP # default
     server role = standalone server # default
     min protocol = SMB3 # optional
     server signing = disabled # optional
     vfs objects = fruit streams_xattr
     fruit:metadata = stream
     fruit:aapl = yes

  [shared]
     path = /home/smbuser/share
     browseable = yes
     read only = no
     guest ok = no
     writable = yes
     valid users = smbuser
     ea support = yes
  ```

  If you plan on having multiple SMB shares, you can (and should) name your share something more distictive than the `[shared]` I did above. But whatever you end up calling it, please remeber that is IS case sensitive! (This will save you alot of headaches down the line.)

  Also just make sure:
  - `valid users =` whatever username you chose in Step 2.
  - `path =` the absolute path of whatever you chose in Step 3.
  - `ea support = yes`: This enables Samba's Extended Attributes (xattr) handling, which is critical for iOS/macOS SMB interoperability.
 
### Step 5. Restart smbd

  `sudo systemctl restart smbd`
  
## Usage
Now that you got server setup out the way, all that's left is to set up your iOS device as a client.

Here's how I did it:

- Go to iOS Files app > Triple dots > "Connect to server"

- Server: enter your smb Linux server IP followed by your share name from Step 4 (without the brackets, case sensitive):
  
  `smb://<server-ip-address>/shared`

- User/Pass: Enter the credentials for the smb user you created, in my case it's `smbuser`

And if all goes well, your SMB share should be fully-writeable from your iOS device!

# Proof

TBD

