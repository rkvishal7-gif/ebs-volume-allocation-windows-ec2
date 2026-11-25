# Allocating Additional Storage on a Windows EC2 Instance (Amazon EBS)

Author: Vishal, BCA graduate, learning AWS cloud

## Overview

This document describes how to attach an additional Amazon EBS volume to a Windows EC2 instance and convert its **unallocated** space into a usable, formatted **allocated** volume with a drive letter. It also covers how to extend the volume later when more capacity is needed.

**Terms used**

| Term | Meaning |
|---|---|
| EBS volume | A network-attached block storage disk that can be attached to an EC2 instance |
| Unallocated space | Disk space that has not yet been assigned to a partition, so Windows cannot store files on it |
| Allocated space | Space that has been partitioned, formatted, and given a drive letter, so it is ready to use |
| Disk Management | The built-in Windows tool for managing disks and partitions (`diskmgmt.msc`) |

## Prerequisites

- A running Windows EC2 instance
- Access to the EC2 console with permission to create and attach volumes
- The key pair used to launch the instance, needed to retrieve the Administrator password for RDP

## Procedure

### Part 1: Create and attach the EBS volume (AWS Console)

1. Open the **EC2 console** and select **Volumes**, then **Create volume**.
2. Choose the volume type (for example, General Purpose SSD `gp3`) and set the size (for example, 12 GiB).
3. Select the **same Availability Zone** as the instance. A volume can only be attached to an instance in its own Availability Zone.
4. Choose **Create volume** and wait until the state is **Available**.
5. Select the volume, then **Actions**, then **Attach volume**.
6. Select the target instance and a device name, then choose **Attach volume**.

### Part 2: Connect to the instance

1. In the EC2 console, select the instance and choose **Connect**, then the **RDP client** tab.
2. Choose **Get password**, upload the key pair file, and decrypt the Administrator password.
3. Open **Remote Desktop Connection** on the local computer, enter the instance's public address, and sign in as `Administrator`.

### Part 3: Allocate the storage (Disk Management)

1. On the instance, right-click **Start** and open **Disk Management** (or run `diskmgmt.msc`).
2. Locate the new disk. It is shown as **Unallocated** and may be **Offline**.
3. If the disk is offline, right-click the disk label and choose **Online**.
4. Right-click the disk label and choose **Initialize Disk**. Select the partition style (**GPT** is recommended) and click **OK**.
5. Right-click the unallocated area and choose **New Simple Volume**.
6. In the wizard:
   - Keep the default volume size to use all available space.
   - Assign an available drive letter.
   - Format with **NTFS**, set a volume label, and select **Perform a quick format**.
7. Click **Finish**. The space now shows as a **Healthy (Primary Partition)**.

### Alternative: PowerShell

Run PowerShell as Administrator. Check the disk number first, because it can differ between systems.

```powershell
Get-Disk
Initialize-Disk -Number 1 -PartitionStyle GPT
New-Partition -DiskNumber 1 -UseMaximumSize -AssignDriveLetter | Format-Volume -FileSystem NTFS -NewFileSystemLabel "Data" -Confirm:$false
```

## Verification

Open **File Explorer** and select **This PC**. The new volume should appear under **Devices and drives** with its full capacity shown as free space (for example, "11.9 GB free of 11.9 GB").

## Extending the volume later

1. In the EC2 console, select the volume, then **Actions**, then **Modify volume**, and enter a larger size.
2. Wait until the modification state is **optimizing** or **completed**.
3. On the instance, open **Disk Management** and rescan if needed (**Action**, then **Rescan Disks**).
4. Right-click the volume and choose **Extend Volume**, then follow the wizard.

Notes:
- An EBS volume can be increased in size but **cannot be decreased**.
- Another modification to the same volume is not allowed for a period after the previous one (typically about six hours).
- Take a snapshot before modifying a volume that holds important data.

## Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| New disk not visible in Disk Management | Confirm the volume is attached and in the same Availability Zone; choose **Action**, then **Rescan Disks** |
| Disk shows Offline | Right-click the disk and choose **Online** |
| "Extend Volume" is greyed out | The unallocated space must be directly after the partition; rescan disks and confirm the EBS modification has completed |
| Volume cannot be attached in the console | The volume and instance are in different Availability Zones |

## Best practices

- Store application data on a separate data volume rather than the system drive (C:).
- Take regular **EBS snapshots** and test a restore.
- Enable **encryption** on volumes that hold sensitive data.
- Use **Delete on termination** deliberately: data volumes you want to keep should not be deleted automatically.
- Delete unused volumes and snapshots to avoid ongoing charges.
- Do not publish account numbers, passwords, or key files in screenshots or documents.

## Result

The additional EBS volume of approximately 12 GB was allocated, formatted, and appears in File Explorer as a usable drive.

## Screenshots

**1. Windows EC2 instance in the Running state**

![Windows EC2 instance running in the AWS console](WhatsApp%20Image%202026-11-25%20at%2011.58.13%20AM.jpeg)

The instance (`server 1`, type `t3.large`) is in the **Running** state in the EC2 console. The AWS account details are blurred for privacy.

**2. New volume allocated and visible in File Explorer**

![New volume shown in This PC after allocation](WhatsApp%20Image%202026-11-25%20at%2011.58.23%20AM.jpeg)

After allocation, the additional volume appears under **Devices and drives** as **New Volume (A:)**, with 11.9 GB free of 11.9 GB, alongside the system drive **Windows (C:)**.
