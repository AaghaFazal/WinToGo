# Windows 10 External Drive Installation / Windows To Go

This README documents the manual process for preparing an external NTFS drive with a separate EFI partition and applying a Windows 10 image from a mounted ISO.

## Drive Letters Used

- `X:` = Mounted Windows 10 ISO / CD-ROM drive
- `W:` = External drive NTFS partition where Windows will be installed
- `S:` = External drive EFI partition

Run Command Prompt as Administrator.

---

## 1. Mount the Windows ISO

### File Explorer

Right-click the Windows 10 ISO and select **Mount**.

### PowerShell

Run PowerShell as Administrator:

```powershell
Mount-DiskImage -ImagePath "C:\path\to\Windows.iso"
```

To find the drive letter assigned to the mounted ISO:

```powershell
Get-DiskImage -ImagePath "C:\path\to\Windows.iso" | Get-Volume
```

### Command Prompt (`cmd.exe`)

`cmd.exe` does not have a native ISO-mount command. From an elevated Command Prompt, call PowerShell:

```cmd
powershell.exe -NoProfile -Command "Mount-DiskImage -ImagePath 'C:\path\to\Windows.iso'"
```

To find the assigned drive letter from `cmd.exe`:

```cmd
powershell.exe -NoProfile -Command "(Get-DiskImage -ImagePath 'C:\path\to\Windows.iso' | Get-Volume).DriveLetter"
```

Use the returned drive letter in place of `X:` below. You can also confirm it with:

```cmd
wmic logicaldisk get deviceid, volumename, description
```

In this example, the mounted ISO is:

```text
X:
```

Check the Windows installation image:

```cmd
dir X:\sources\install.*
```

You should find one of:

```text
X:\sources\install.wim
```

or:

```text
X:\sources\install.esd
```

---

## 2. Check Available Windows Editions

### If the ISO contains install.wim

```cmd
Dism /Get-WimInfo /WimFile:"X:\sources\install.wim"
```

### If the ISO contains install.esd

```cmd
Dism /Get-WimInfo /WimFile:"X:\sources\install.esd"
```

Note the `Index` of the Windows edition you want to install.

Example:

```text
Index : 1
Name : Windows 10 Home

Index : 6
Name : Windows 10 Pro
```

Use the correct index for your image.

---

# 3. Prepare the External Drive

WARNING:

The following commands can erase the selected disk.

Make absolutely sure that you select the external drive and NOT your normal Windows/system disk.

Open DiskPart:

```cmd
diskpart
```

List disks:

```cmd
list disk
```

Identify the external drive by its size.

Select the correct disk:

```cmd
select disk N
```

Replace `N` with the external disk number.

Verify it:

```cmd
list disk
```

If you are intentionally erasing the entire external drive:

```cmd
clean
```

Create a GPT partition layout:

```cmd
convert gpt
```

---

# 4. Create the EFI Partition

Create a 200 MB EFI System Partition:

```cmd
create partition efi size=200
```

Format it as FAT32:

```cmd
format fs=fat32 quick label="EFI"
```

Assign drive letter S:

```cmd
assign letter=S
```

---

# 5. Create the Windows NTFS Partition

Create the remaining space as a primary partition:

```cmd
create partition primary
```

Format it as NTFS:

```cmd
format fs=ntfs quick label="Windows To Go"
```

Assign drive letter W:

```cmd
assign letter=W
```

Exit DiskPart:

```cmd
exit
```

The resulting layout should look approximately like:

```text
External Drive
|
+-- EFI partition
|   FAT32
|   ~200 MB
|   S:
|
+-- Windows partition
    NTFS
    Remaining space
    W:
```

---

# 6. Apply the Windows Image

## Option A: Apply install.wim from the ISO

```cmd
Dism /Apply-Image /ImageFile:"X:\sources\install.wim" /ApplyDir:W:\ /Index:6 /CheckIntegrity
```

Replace `6` with the desired image index.

## Option B: Apply install.esd directly

```cmd
Dism /Apply-Image /ImageFile:"X:\sources\install.esd" /ApplyDir:W:\ /Index:6 /CheckIntegrity
```

Replace `6` with the desired index.

Wait for DISM to finish completely.

---

# 7. Create the UEFI Boot Entry

After Windows has been applied to W:, create the boot files on the EFI partition:

```cmd
bcdboot W:\Windows /s S: /f UEFI
```

Expected result:

```text
Boot files successfully created.
```

This command automatically creates the required EFI boot directory structure.

You do NOT need to manually create:

```text
S:\EFI\Microsoft\Boot
S:\EFI\Boot
```

`bcdboot` creates the required files.

---

# 8. Verify the Windows Installation

Check that Windows exists:

```cmd
dir W:\Windows
```

Check the EFI boot directory:

```cmd
dir S:\EFI\Boot
```

Check the Microsoft boot directory:

```cmd
dir S:\EFI\Microsoft\Boot
```

You should see Windows boot files and the BCD store.

---

# 9. Remove the EFI Drive Letter

After confirming that the boot files were created, you can remove the `S:` drive letter.

Open DiskPart:

```cmd
diskpart
```

List volumes:

```cmd
list volume
```

Find the small FAT32 EFI volume.

Select it:

```cmd
select volume N
```

Replace `N` with the EFI volume number.

Remove the drive letter:

```cmd
remove letter=S
```

Exit:

```cmd
exit
```

This does NOT delete the EFI partition.

It only removes the `S:` drive letter.

---

# 10. Final Partition Layout

The final external drive should contain:

```text
External Drive
|
+-- EFI System Partition
|   FAT32
|   ~200 MB
|   No drive letter
|
+-- Windows To Go
    NTFS
    Windows installation
    W:
```

The EFI partition contains the UEFI boot files.

The NTFS partition contains the Windows installation.

---

# Important Notes

1. Do not use the ISO file itself with `/Get-WimInfo`.

Incorrect:

```cmd
Dism /Get-WimInfo /WimFile:"C:\path\Windows.iso"
```

Correct:

```cmd
Dism /Get-WimInfo /WimFile:"X:\sources\install.wim"
```

or:

```cmd
Dism /Get-WimInfo /WimFile:"X:\sources\install.esd"
```

2. `install.wim` and `install.esd` are image files contained inside the ISO.

3. The EFI partition should be FAT32 for standard UEFI booting.

4. `create partition efi` automatically creates an EFI System Partition. Manually setting the EFI GPT type GUID is normally unnecessary.

5. `bcdboot` creates the required EFI boot files. Do not manually create the EFI directory structure unless troubleshooting.

6. Be extremely careful with:

```cmd
clean
```

The `clean` command removes the partition information from the selected disk.

Always verify:

```cmd
list disk
```

before running it.

---

# Quick Command Reference

## Check image

```cmd
Dism /Get-WimInfo /WimFile:"X:\sources\install.wim"
```

or:

```cmd
Dism /Get-WimInfo /WimFile:"X:\sources\install.esd"
```

## GPT / EFI / NTFS setup

```cmd
diskpart
list disk
select disk N
clean
convert gpt
create partition efi size=200
format fs=fat32 quick label="EFI"
assign letter=S
create partition primary
format fs=ntfs quick label="Windows To Go"
assign letter=W
exit
```

## Apply Windows

```cmd
Dism /Apply-Image /ImageFile:"X:\sources\install.wim" /ApplyDir:W:\ /Index:6 /CheckIntegrity
```

## Create UEFI boot files

```cmd
bcdboot W:\Windows /s S: /f UEFI
```

## Remove EFI drive letter

```cmd
diskpart
list volume
select volume N
remove letter=S
exit
```

---

# Example Drive Configuration

```text
X:  Windows 10 ISO
S:  EFI partition (FAT32, ~200 MB)
W:  Windows partition (NTFS, remaining space)
```

This procedure creates a GPT external drive with a dedicated EFI System Partition and a Windows installation on the NTFS partition.
