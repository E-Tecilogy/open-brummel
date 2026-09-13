# Stadlbauer Brummel: Technical Specification

This document contains the technical data and communication notes observed so far. All information is provided **without warranty**. Unconfirmed points are explicitly marked as hypotheses.

## USB Device

```text
VID                 0x0c45
PID                 0x9040
Manufacturer        Microdia (USB descriptor)
Product             Storage 9040 (Linux device identification)
Serial number       000000000001
USB version         1.10
Device class        0x00, not classified in the device descriptor
MaxPacketSize0      64 bytes
Configuration       1
Configuration value 1
Attributes          Self Powered
MaxPower            500 mA
Interface           0
Interface class     0x08 Mass Storage
Subclass            0x06 SCSI Transparent
Protocol            0x50 Bulk-Only Transport
Bulk IN             Endpoint 0x81, 64 bytes
Bulk OUT            Endpoint 0x02, 64 bytes
```

The bus and device numbers change depending on the USB connection and are not fixed device properties. Under Linux, the device was identified as `/dev/sr1` with the label `Brummel`.

## Visible Medium

The visible medium is handled by the Linux `usb-storage` driver as a read-only CD-ROM:

```text
Filesystem           ISO-9660 / Joliet
Block size           2048 bytes
ISO image size       163840 bytes in the reference image
Reported device size approximately 31.6 MiB
Volume ID            BRUMMEL
ISO system ID        APPLE INC., TYPE: 0002
ISO UUID             2013-01-12-16-06-29-00
SCSI device          sr1
```

The root directory contains exactly these files:

```text
AUTORUN.INF                         136 bytes
BrummelBibliothek installieren.url  73 bytes
BrummelBibliothek starten.exe       123440 bytes
```

The stories are not present as files on this visible ISO.

## Windows Reference Application

The local reference is an Adobe AIR application:

```text
AIR namespace       http://ns.adobe.com/air/application/3.1
Application ID      at.stadlbauer.BrummelApp
Version             1.0.1
Name                BrummelBibliothek
SWF                 BrummelApp.swf
Profile             extendedDesktop
ANE                 at.stadlbauer.SNC715
ANE platform        Windows-x86
Native ANE DLL      SNCExt.dll
Initializer         SNCExtInitialize
Finalizer           SNCExtFinalize
```

Other files in the Windows package:

```text
FWDll.dll            90112 bytes, PE32 Windows DLL
SNC715FAT.dll        77824 bytes, PE32 Windows DLL
BrummelApp.swf       2811682 bytes, compressed Flash/SWF
brummel_raw.img      163840 bytes, ISO-9660 image
```

## Observed Communication Layers

### 1. USB / Mass Storage Layer

The Linux standard path communicates through USB Mass Storage, SCSI Transparent, and Bulk-Only Transport. `0x81` is the Bulk-IN data channel and `0x02` is the Bulk-OUT channel. In Mass Storage, command blocks are sent over OUT and responses or data are received over IN.

The `READ(10)` opcode is `0x28`. The fact that a command is intended to read does not guarantee that a guessed packet is safe for this device. A malformed command block can cause timeouts, USB resets, or device state changes. The safe test therefore uses only the kernel-provided volume in read-only mode.

### 2. Flash / Transport Layer

The following names were observed in `FWDll.dll`:

```text
NORFW_CheckDevice
NORFW_Initial
NORFW_AutoRead
NORFW_AutoWrite
NORFW_BulkIn
NORFW_BulkOut
NORFW_BulkNone
NORFW_ReadRAMData
NORFW_SetVirtualFAT
NORFW_GetID
NORFW_GetBatteryValue
NORFW_FlashReset
NORFW_SectorErase
NORFW_SerialFlashChipErase
NORFW_ExtendedCmd_0..9
NORFW_SyncPictureSize
NORFW_WritePictureData
NORFW_SynchronizeDate
NORFW_SynchronizeTime
```

These names suggest a host/flash abstraction with automatic reading, RAM reading, and Bulk-IN/OUT operations. They do **not** establish verified function signatures or byte formats.

### 3. Virtual FAT / File Layer

The following names were observed in `SNC715FAT.dll`:

```text
SNC715_FAT_InitProgramStates
SNC715_FAT_CheckDevice
SNC715_FAT_OpenFAT
SNC715_FAT_CloseFAT
SNC715_FAT_GetFileInfo
SNC715_FAT_FreeFileInfo
SNC715_FAT_ReadFileData
SNC715_FAT_WriteFileData
SNC715_FAT_SetFileInfo
SNC715_FAT_GetFlashInfo
SNC715_FAT_GetHardwareParameter
SNC715_FAT_SetHardwareParameter
SNC715_FAT_NoticeDeviceEOF
SNC715_FAT_GetBatteryValue
SNC715_FAT_Sync
SNC715_FAT_Defragment
SNC715_FAT_ExtendedCmd_0..9
```

The most likely architecture is:

```text
BrummelBibliothek / AIR SWF
    -> SNC715_FAT.dll: virtual files and FAT management
    -> FWDll.dll / SNCExt.dll: flash and USB transport
    -> SNC715 controller in the bear
    -> internal NOR flash containing the stories
```

The FAT geometry, filename encoding, flash addresses, parameters, status codes, authentication, audio format, mouth-data format, and exact initialization order have not been verified.

## Communication Status

The device identity, USB descriptors, visible ISO, and function names from the reference DLLs are confirmed observations. A complete protocol for reading the internal story flash has not been confirmed. The original application and its DLLs are not part of a new implementation.

**All information is provided without warranty.**
