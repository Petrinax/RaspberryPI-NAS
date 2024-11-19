# RaspberryPI-NAS

**Overview:**

This document covers creating Network Attached Storage (NAS) out of Raspberry Pi running in home network. NAS to be accessed remotely from the internet.

## Table of Content

- [RaspberryPI-NAS](#raspberrypi-nas)
  - [Table of Content](#table-of-content)
  - [Setting Up Raspberry Pi as a NAS](#setting-up-raspberry-pi-as-a-nas)
    - [Recommendations](#recommendations)
  - [Samba Setup](#samba-setup)
    - [Steps to Configure Samba for File Sharing:](#steps-to-configure-samba-for-file-sharing)
      - [Configured Storage Drive](#configured-storage-drive)
  - [Access Shared Storage:](#access-shared-storage)
      - [Windows:](#windows)
      - [Linux:](#linux)
      - [MacOS:](#macos)
      - [Android:](#android)
  - [Mount Drives](#mount-drives)
    - [1. Check file systems](#1-check-file-systems)
    - [2. Mount Storage Drive](#2-mount-storage-drive)
    - [3. Add mount automation on reboot:](#3-add-mount-automation-on-reboot)
    - [4. Verify](#4-verify)
    - [Re-Mount:](#re-mount)
    - [1. Unmount existing storage:](#1-unmount-existing-storage)
    - [2. Reload `systemd` and Apply `fstab`](#2-reload-systemd-and-apply-fstab)
    - [3. Debugging:](#3-debugging)
  - [Shared Storage Access Management](#shared-storage-access-management)

## Setting Up Raspberry Pi as a NAS
### Recommendations
To turn your Raspberry Pi into a Network Attached Storage (NAS) that can also be accessed remotely, consider the following options:

1. **Install an Operating System for NAS:**  
   Raspberry Pi can run various NAS operating systems or configurations:
   - **OpenMediaVault (Recommended)**: Lightweight, with features similar to TrueNAS.
   - **DietPi**: A lightweight OS where you can configure Samba for file sharing.
   - **Raspberry Pi OS + Samba**: A simple and flexible setup.
   - **Nextcloud**: If you want a personal cloud storage solution.

2. **Enable Remote Access:**  
   - Use **Dynamic DNS** (e.g., No-IP, DuckDNS) to access your NAS from the internet.  
   - Configure port forwarding on your router to allow external access.  
   - For security, set up a VPN on your Raspberry Pi or use SSH tunneling.

3. **Considerations:**  
   - Ensure your Pi has adequate storage attached (via USB).  
   - Secure your system with strong passwords and keep software updated.

Here we will setup the NAS using RaspberryPi OS + Samba approach.


## Samba Setup

### Steps to Configure Samba for File Sharing:

1. **Install Samba:**  
   Run the following command on your Raspberry Pi:  
   ```bash
   sudo apt update
   sudo apt install samba samba-common-bin
   ```

2. **Create a Directory to Share:**  
   For example, create a directory named `nas` in the `pi` user's home directory:  
   ```bash
   mkdir /mnt/storage
   ```

3. **Configure Samba:**  
   Open the Samba configuration file:  
   ```bash
   sudo nano /etc/samba/smb.conf
   ```

   Add the following to the end of the file to share the `nas` directory:  

   #### Configured Storage Drive
   ```conf
   [Shared]
   path = /mnt/storage/shared
   available = yes
   valid users = petrinax, @nas_admin, @nas_rw, @nas_ro
   read only = no
   browsable = yes
   public = no
   read list = @nas_ro
   write list = petrinax, @nas_admin, @nas_rw
   create mask = 0777
   directory mask = 0777
   ```
   More details on configurations [here](https://www.samba.org/samba/docs/4.9/man-html/smb.conf.5.html).

4. **Set a Samba Password for the User:**  
   ```bash
   sudo smbpasswd -a <existing_linux_user>
   sudo smbpasswd -e <existing_linux_user>
   ```

5. **Restart Samba:**  
   ```bash
   sudo systemctl restart smbd
   ```

## Access Shared Storage:

   #### Windows:
   - Open File Explorer and type `\\<Raspberry_Pi_IP>\NAS`.

   #### Linux:
   - Use a file manager or mount the share with `smbclient`.

   #### MacOS:
   - Open Finder and press `Command + K` to open the `Connect to Server` window
   - Enter the SMB address of your Raspberry Pi
      ```bash
      smb://100.x.x.x/shared_folder
      ```
   - Replace 100.x.x.x with your Raspberry Pi's Tailscale IP and shared_folder with your Samba share name.
   - Click Connect and enter your Samba username and password.
   - The shared folder will appear in Finder under Locations.

   #### Android:
   Use a File Manager App with SMB Support.
   - Open the file manager app and configure a new Network Storage (SMB) connection.
   - Use the IP of your Raspberry Pi (e.g., `100.x.x.x`).
   - Enter your Samba credentials.
   - Provide the path to the shared folder (e.g., shared_folder).
   - Save the connection and browse your NAS files through the file manager.


## Mount Drives

### 1. Check file systems
List information about block devices, their file systems, and mount points:
```bash
lsblk -f
```


### 2. Mount Storage Drive
Mount FAT32 drive with specific ownership and permission settings:
```bash
sudo mount -o uid=1000,gid=1005,umask=0000 /dev/sda1 /mnt/storage
```
More details on filesystem specific mounts [here](https://man7.org/linux/man-pages/man8/mount.8.html#FILESYSTEM-SPECIFIC_MOUNT_OPTIONS).

Command Breakdown:

- `sudo`: Runs the command with superuser privileges.
mount: Mounts the file system.

- `-o`: Specifies additional options for the mount command.

Options Passed:

- `uid=1000`:
Sets the user ID for ownership of the files and directories.
1000 is the default UID for the first user (e.g., pi on Raspberry Pi). You can check your UID with id username.

- `gid=1000`:
Sets the group ID for ownership of the files and directories.
1000 is the default GID for the primary group of the first user.

- `umask=0000`:
Sets the permissions mask for files and directories.
0000 results in:
Directories & Files: rwxrwxrwx (Owner, group and others have full access read/write/execute).

- `/dev/sda1`:
Specifies the device name of the FAT32 drive to mount.

- `/mnt/storage`:
Specifies the directory where the drive will be mounted.

### 3. Add mount automation on reboot:
Adding an entry for your FAT32 drive in `/etc/fstab` ensures it's mounted automatically at boot with the desired options.

```bash
/dev/sda1 /mnt/storage vfat uid=1000,gid=1000,umask=0022 0 0
```

Explanation:

- `/dev/sda1`: The device name of the drive.

- `/mnt/storage`: The mount point for the drive.

- `vfat`: The file system type (used for FAT32).

- `uid=1000,gid=1000,umask=0022`: Ownership and permission options.

- `0 0`:
The first `0` disables file system dumps.
The second `0` disables file system checks during boot.

### 4. Verify

Check the ownership and permissions of the mount point:

```bash
ls -ld /mnt/storage
```

- `ls`: Lists files and directories.
- `-l`: Displays detailed information, including permissions, owner, and group.
- `-d`: Shows information about the directory itself (not its contents).

### Re-Mount:

### 1. Unmount existing storage:
```bash
sudo umount /mnt/storage
```

### 2. Reload `systemd` and Apply `fstab`
```bash
sudo systemctl daemon-reload
```

Remount all file systems based on the updated /etc/fstab:
```bash
sudo mount -a
```
It will remount all the file systems defined in /etc/fstab that are not already mounted.

### 3. Debugging:
If unmounting fails with `umount: /mnt/storage/: target is busy.` error. Try stopping Samba service:
```bash
sudo systemctl stop smbd
```
Once unmounted restart smb daemon:
```bash
sudo systemctl start smbd
```


## Shared Storage Access Management

- Configured NAS drive: [Shared](#configured-storage-drive)
- Mount path: `/mnt/storage/shared`

Users needs to be added to following groups for Shared Storage access.


| Group | Description |
| - | - |
| `nas_admin` | Full access to all files and configurations. |
| `nas_rw` | Can read and write to shared directories. |
| `nas_ro` | Can only read files in shared directories. |


---
