# Stadlbauer Brummel: Technische Spezifikation

Dieses Dokument enthält die bisher beobachteten technischen Daten und Hinweise zur Kommunikation. Alle Angaben erfolgen **ohne Gewähr**. Unbestätigte Punkte sind ausdrücklich als Hypothesen gekennzeichnet.

## USB-Gerät

```text
VID                 0x0c45
PID                 0x9040
Hersteller          Microdia (USB-Descriptor)
Produkt             Storage 9040 (Linux-Geräteerkennung)
Seriennummer        000000000001
USB-Version         1.10
Geräteklasse        0x00, im Device Descriptor nicht klassifiziert
MaxPacketSize0      64 Byte
Konfiguration       1
Konfigurationswert  1
Attribute           Self Powered
MaxPower            500 mA
Interface           0
Interfaceklasse     0x08 Mass Storage
Subclass            0x06 SCSI Transparent
Protokoll           0x50 Bulk-Only Transport
Bulk IN             Endpoint 0x81, 64 Byte
Bulk OUT            Endpoint 0x02, 64 Byte
```

Bus- und Gerätenummer ändern sich je nach USB-Anschluss und sind keine festen Geräteeigenschaften. Unter Linux wurde das Gerät als `/dev/sr1` mit dem Label `Brummel` erkannt.

## Sichtbares Medium

Das sichtbare Medium wird vom Linux-Treiber `usb-storage` als schreibgeschütztes CD-ROM behandelt:

```text
Dateisystem          ISO-9660 / Joliet
Blockgröße           2048 Byte
ISO-Imagegröße       163840 Byte im Referenzabbild
Gemeldete Größe      ungefähr 31,6 MiB beim Gerät
Volume-ID            BRUMMEL
ISO-System-ID        APPLE INC., TYPE: 0002
ISO-UUID             2013-01-12-16-06-29-00
SCSI-Gerät           sr1
```

Das Root-Verzeichnis enthält genau diese Dateien:

```text
AUTORUN.INF                         136 Byte
BrummelBibliothek installieren.url  73 Byte
BrummelBibliothek starten.exe       123440 Byte
```

Die Geschichten sind auf diesem sichtbaren ISO nicht als Dateien vorhanden.

## Windows-Referenzanwendung

Die lokale Referenz ist eine Adobe-AIR-Anwendung:

```text
AIR-Namespace       http://ns.adobe.com/air/application/3.1
Application-ID      at.stadlbauer.BrummelApp
Version             1.0.1
Name                BrummelBibliothek
SWF                 BrummelApp.swf
Profil              extendedDesktop
ANE                 at.stadlbauer.SNC715
ANE-Plattform       Windows-x86
Native ANE-DLL      SNCExt.dll
Initializer         SNCExtInitialize
Finalizer           SNCExtFinalize
```

Weitere Dateien im Windows-Paket:

```text
FWDll.dll            90112 Byte, PE32-Windows-DLL
SNC715FAT.dll        77824 Byte, PE32-Windows-DLL
BrummelApp.swf       2811682 Byte, komprimiertes Flash/SWF
brummel_raw.img      163840 Byte, ISO-9660-Abbild
```

## Beobachtete Kommunikationsschichten

### 1. USB-/Mass-Storage-Schicht

Der Linux-Standardpfad kommuniziert über USB Mass Storage, SCSI Transparent und Bulk-Only Transport. `0x81` ist der Bulk-IN-Datenkanal und `0x02` der Bulk-OUT-Kanal. Bei Mass Storage werden Kommandoblöcke über OUT gesendet und Antworten oder Daten über IN empfangen.

Der Opcode für `READ(10)` ist `0x28`. Dass ein Kommando lesen soll, garantiert nicht, dass ein geratenes Paket für dieses Gerät sicher ist. Ein fehlerhafter Kommandoblock kann Timeouts, USB-Resets oder Zustandsänderungen auslösen. Der sichere Test verwendet deshalb nur das vom Kernel bereitgestellte Volume im Read-only-Modus.

### 2. Flash-/Transport-Schicht

In `FWDll.dll` wurden folgende Namen beobachtet:

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

Diese Namen deuten auf eine Host-/Flash-Abstraktion mit automatischem Lesen, RAM-Lesen und Bulk-IN/OUT-Operationen hin. Sie liefern **keine** verifizierten Funktionssignaturen oder Byteformate.

### 3. Virtuelle FAT-/Dateischicht

In `SNC715FAT.dll` wurden folgende Namen beobachtet:

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

Die wahrscheinlichste Architektur ist:

```text
BrummelBibliothek / AIR-SWF
    -> SNC715_FAT.dll: virtuelle Dateien und FAT-Verwaltung
    -> FWDll.dll / SNCExt.dll: Flash- und USB-Transport
    -> SNC715-Controller im Bären
    -> interner NOR-Flash mit Geschichten
```

Nicht verifiziert sind FAT-Geometrie, Dateinamenkodierung, Flash-Adressen, Parameter, Statuscodes, Authentisierung, Audioformat, Munddatenformat und die genaue Reihenfolge der Initialisierung.

## Kommunikationsstatus

Bestätigt sind Gerätekennung, USB-Deskriptoren, sichtbares ISO und die Funktionsnamen der Referenz-DLLs. Nicht bestätigt ist ein vollständiges Protokoll zum Lesen des internen Geschichten-Flashs. Die originale Anwendung und ihre DLLs sind nicht Bestandteil einer neuen Implementierung.

**Alle Angaben erfolgen ohne Gewähr.**
