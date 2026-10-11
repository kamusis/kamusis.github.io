---
title: How to move WSL distro in Windows 11 to another drive
date: 2024-05-27 15:17:24
tags: [Windows, WSL]
---

## Introduction:

In the world of development and system administration, Windows Subsystem for Linux (WSL) has become a valuable tool. It allows users to run a Linux distribution alongside their Windows environment, opening up a world of possibilities for developers and administrators. In this article, we’ll guide you through the process of migrating a WSL instance, using a real-world example, step by step.

## Prerequisites:

Before we begin, ensure that you have the following prerequisites in place:

- Windows 10 or later with WSL installed.
- An existing WSL instance (in our case, Ubuntu).
- Sufficient storage space for the migration.

## Step 1: Create a Target Directory

To start the migration process, we need a target directory to store the migrated WSL instance. In PowerShell, use the ‘mkdir’ command to create this directory. In our example, we create a directory named ‘D:\WSL\Ubuntu’:

```powershell
mkdir -p D:\WSL\Ubuntu
```

## Step 2: List All Running WSL Instances

Before we proceed further, let’s list all the running WSL instances. The following command will display a list of all WSL instances, including their state and version:

```powershell
wsl -l --all -v
```

## Step 3: Export the Source WSL Instance

Now, let’s export the source WSL instance (in our case, ‘Ubuntu’) into a tar file. This step automatically shuts down the WSL instance and restarts it after the export:

```powershell
wsl --export Ubuntu D:\WSL\Ubuntu.tar
```

## Step 4: Unregister the Source WSL Instance

Once the export is complete, we need to unregister the source WSL instance to avoid conflicts. Use the following command:

```powershell
wsl --unregister Ubuntu
```

## Step 5: Confirm Unregistration

To confirm that the source WSL instance has been successfully unregistered, run the following command:

```powershell
wsl -l --all -v
```

## Step 6: Import into the Target Directory

Now it’s time to import the previously exported WSL instance into the target directory. In this step, we specify the target directory and version (in our case, version 2):

```powershell
wsl --import Ubuntu D:\WSL\Ubuntu D:\WSL\Ubuntu.tar --version 2
```

## Step 7: Verify the Migration

To ensure that the migration was successful, list all WSL instances once again:

```powershell
wsl -l --all -v
```

## Step 8: Access the Migrated WSL Instance

Now, you can access the migrated WSL instance by using the following command:

```powershell
wsl -d Ubuntu
```

## Conclusion:

Migrating WSL instances is a powerful way to manage and organize your development environments. By following these steps, you can seamlessly move your WSL instances to different directories or machines, ensuring flexibility and efficiency in your development workflow. Keep in mind that WSL provides a bridge between Windows and Linux, allowing you to enjoy the best of both worlds.

Check the all steps screenshot as below.

```
# create target directory
PS C:\Users\kamus> mkdir -p D:\WSL\Ubuntu

# List all the wsl running
PS C:\Users\kamus> wsl -l --all -v
  NAME                   STATE           VERSION
* Ubuntu                 Running         2
  docker-desktop         Stopped         2
  docker-desktop-data    Stopped         2
  
# Export source wsl
PS C:\Users\kamus> wsl --export Ubuntu D:\WSL\Ubuntu.tar

# When doing export, wsl will be shutdown automatically and restart after exporting
PS C:\Users\kamus> wsl -l --all -v
  NAME                   STATE           VERSION
* Ubuntu                 Running         2
  docker-desktop         Stopped         2
  docker-desktop-data    Stopped         2
  
# Unregister the source wsl
PS C:\Users\kamus> wsl --unregister Ubuntu
正在注销...

# Check unregister is successful
PS C:\Users\kamus> wsl -l --all -v
  NAME                   STATE           VERSION
* docker-desktop         Stopped         2
  docker-desktop-data    Stopped         2
  
# Import into the target directory
PS C:\Users\kamus> wsl --import Ubuntu D:\WSL\Ubuntu D:\WSL\Ubuntu.tar --version 2

# Check results
PS C:\Users\kamus> wsl -l --all -v
  NAME                   STATE           VERSION
* docker-desktop         Stopped         2
  Ubuntu                 Stopped         2
  docker-desktop-data    Stopped         2
PS C:\Users\kamus> wsl -d Ubuntu
Welcome to Ubuntu 20.04.5 LTS (GNU/Linux 5.10.102.1-microsoft-standard-WSL2 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Jan  5 14:40:25 JST 2024

  System load:  0.68               Processes:             8
  Usage of /:   2.0% of 250.98GB   Users logged in:       0
  Memory usage: 4%                 IPv4 address for eth0: 172.28.208.11
  Swap usage:   0%


0 updates can be applied immediately.


The list of available updates is more than a week old.
To check for new updates run: sudo apt update


This message is shown once a day. To disable it please create the
/root/.hushlogin file.
root@Kamus-Trident:/mnt/c/Users/kamus# cat /etc/os-release
NAME="Ubuntu"
VERSION="20.04.5 LTS (Focal Fossa)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 20.04.5 LTS"
VERSION_ID="20.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=focal
UBUNTU_CODENAME=focal
root@Kamus-Trident:/mnt/c/Users/kamus#
```

