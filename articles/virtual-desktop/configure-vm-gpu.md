---
title: Configure GPU for Windows Virtual Desktop Preview - Azure
description: How to enable GPU for Remote Desktop sessions and enable hardware encoding of RDP in Windows Virtual Desktop preview master.
services: virtual-desktop
author: denisgun

ms.service: virtual-desktop
ms.topic: how-to
ms.date: 04/07/2019
ms.author: denisgun
---

# Configure GPU for Windows Virtual Desktop Preview

Windows Virtual Desktop Preview uses Remote Desktop Protocol for transferring desktop and application graphics from a remote virtual machine (VM) to a client device. To better use network resources, RDP uses combination of different codecs, one of them is ITU-T H.264 codec (also known as MPEG-4 AVC (Advanced Video Coding)). This codec is widely available in both server and client hardware, including the GPU-enabled VMs on Azure Compute. This means resource-intensive encoding operation offloaded to the GPU and leave more CPU resources available to the user.
Windows Server 2016 and Windows 10 1511 support AVC/H.264 4:4:4 format, which doesn’t lose the chrominance during conversion and ensure the high-quality text in a remote session.

In this article you will learn how to:

> [!div class="checklist"]
> * Enable use of Graphics Processing Unit (GPU) in a Remote Desktop session
> * Enable hardware acceleration of Remote Desktop Protocol (RDP).

## Prerequisites
Prior to performing the steps outlined in this article you must have configured and verified:

- Windows Virtual Desktop tenant configured.
- Session host pool VMs configured and registered with the Windows Virtual Desktop service.
- One or multiple session hosts created on VMs with one of the [GPU-Optimized](/azure/virtual-machines/windows/sizes-gpu) VM sizes.

### Operating System

Use of GPU and hardware acceleration of RDP is supported on the following OSs:

| Distribution |
|---|
| Windows 10 (version 1511 or higher)|
| Windows Server 2016 |

### VM Sizes
Use of GPU and hardware acceleration of RDP is supported on the VM sizes listed in the following article:
[GPU-Optimized](/azure/virtual-machines/windows/sizes-gpu) VM sizes

### Driver version

Only NVIDIA GRID drivers listed in the following article are supported for Windows Virtual Desktop. Make sure that you use GRID driver redistributed by Microsoft:
[NVIDIA GRID drivers](/azure/virtual-machines/windows/n-series-driver-setup#nvidia-grid-drivers)

### App Groups
Only Desktop type of App Group is supported for GPU-enabled host pools. You can use default desktop app group (named "Desktop Application Group") which is automatically created whenever you create a host pool.
RemoteApp App Groups aren't supported for GPU-enabled host pools.

## Install Driver

To take advantage of the GPU capabilities of Azure N-series VMs in Windows Virtual Desktop Preview, NVIDIA GPU drivers must be installed. The [NVIDIA GPU Driver Extension](/azure/virtual-machines/extensions/hpccompute-gpu-windows) installs appropriate NVIDIA GRID drivers on an N-series VM. Install or manage the extension using the Azure portal or tools such as Azure PowerShell or Azure Resource Manager templates. See the [NVIDIA GPU Driver Extension documentation](/azure/virtual-machines/extensions/hpccompute-gpu-windows) for supported operating systems and deployment steps. For general information about VM extensions, see [Azure virtual machine extensions and features](/azure/virtual-machines/extensions/overview).

If you choose to install NVIDIA GPU drivers manually, see [N-series GPU driver setup for Windows](/azure/virtual-machines/windows/n-series-driver-setup) for supported operating systems, drivers, and installation and verification steps.

After GRID driver installation on a VM, a restart is required.

## Configure Group Policy Object (GPO)

After installing the GPU driver, additional step is required to configure a Group Policy Object. To perform this task, you can ether use  Local Group Policy or Active Directory Group Policy Management.

### Configuring Local Group Policy

To configure the Local Group Policy on the GPU-Optimized VM:

1. Connect to the desktop of the VM using the account that has local administrator privileges.
2. Open the Start menu and type "gpedit.msc" to open the Group Policy Editor
3. Navigate the tree: **Computer Configuration -> Administrative Templates -> Windows Components -> Remote Desktop Services -> Remote Desktop Session Host -> Remote Session Environment**
4. Select policy **"Use the hardware default graphics adapter for all Remote Desktop Services sessions"** and set this policy to **"Enabled"** to enable GPU rendering in the remote session.
5. Select policy **"Prioritize H.264/AVC 444 Graphics mode for Remote Desktop connections"** and set this policy to **"Enabled"** to force H.264/AVC 444 codec in the remote session.
6. Select policy **"Configure H.264/AVC hardware encoding for Remote Desktop connections"** and set this policy to **"Enabled"** to enable hardware encoding for AVC/H.264 in the remote session. 

>[!NOTE]
>In Windows Server 2016, you must select option **"Prefer AVC Hardware Encoding"** to **"Always attempt"**.

7. Open command prompt and type:

```batch
gpupdate.exe /force
```

8. Sign out from the Remote Desktop session

### Active Directory Group Policy

Active Directory Group Policies allow you to configure a policy once and apply it to multiple hosts automatically. However, the planning of Active Directory Group Policies is outside of the scope for this tutorial. For more information on managing GPO infrastructure, use the following article: [Group Policy](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-R2-and-2012/hh831791(v%3dws.11))

## Verify the Configuration

To verify driver installation, follow the steps outlined here: [Verify driver installation](https://docs.microsoft.com/en-us)/azure/virtual-machines/windows/n-series-driver-setup#verify-driver-installation)

To verify that a Remote Desktop session is using AVC444 mode and is using hardware encoding RDP:

### Using Event Viewer

1. Connect to the desktop of the VM using Windows Virtual Desktop client
2. Launch the Event Viewer and navigate to the following node: **Applications and Services Logs -> Microsoft -> Windows -> RemoteDesktopServices-RdpCoreTS - > Operational**
3. To determine if AVC 444 mode is used, look for **Event ID 162**, if **"AVC Available: 1 Initial Profile: 2048"** then AVC 444 is used.
4. To determine if Hardware Encoding is used, look for **Event ID 170**, if **"AVC hardware encoder enabled: 1"** then hardware is used.

### Using Powershell

1. Connect to the desktop of the VM using Windows Virtual Desktop client
2. Open Powershell window and run the following command:

```powershell
Get-WinEvent -LogName "Microsoft-Windows-RemoteDesktopServices-RdpCoreTS/Operational" -FilterXPath "*/System/EventID=170 and */EventData/Data[@Name='IsHardwareEncode']=1" -MaxEvents 1|Select-Object -Property TimeCreated,Message|fl
```

If commands show **"AVC hardware encoder enabled"** then GPU is used.

## Next steps

Now that you have a GPU-Optimized VM, repeat the configuration steps for each VM in a host pool.