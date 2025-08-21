# BYOVD Read Write primitive

## Disclaimer
⚠️ This project is provided exclusively for educational purposes and is intended to be used only in authorized environments. You may only run or deploy this project on systems you own or have explicit, documented permission to test. Any unauthorized use of this project against systems without consent is strictly prohibited and may be illegal.

By using this project, you agree to use it responsibly and ethically. The author assumes no liability for misuse or any consequences arising from the use of this project.

This project involves reading from or writing to kernel memory. Such operations can cause system instability, crashes, or Blue Screen of Death (BSOD) if used incorrectly. Proceed with caution, and ensure you fully understand the potential impact before running the code.

Tested on:
- Windows 11 (24H2)
- Windows Server 2022 (21H2)

# General

To practice Bring Your Own Vulnerable Driver (BYOVD) techniques from the CETP course, I set out to develop a toolkit leveraging a kernel-level read/write primitive to bypass security mechanisms such as LSASS’s RunasPPL protection and to enumerate and remove EDR telemetry via kernel callback manipulation. For the vulnerable driver, I used the well-known [RTCore64.sys](https://www.loldrivers.io/drivers/e32bc3da-4db1-4858-a62c-6fbe4db6afbd/) from MSI Afterburner.

# Proof of Concept code examples

This `C` project includes multiple proof-of-concept (POC) code examples that perform the following exploitation concepts:
- Changing Process Protection Levels
	- Disable Runasppl LSASS protection
- Removing Kernel Callbacks
- Disabling ETW providers
- Change process token
	- Privilege escalation to system
	- Downgrade EDR's token
- Disable DSE in case VBS is disabled and load unsigned driver

Requirements:
- All  these attacks require local administrative privileges to load the vulnerable driver.

## Kernel Callback Remover
- What does it do
	- Calculates and prints offsets by downloading symbols from the internet as in EDRSandBlast project for all kernel callbacks and their structures.
	- Writes vulnerable RTCore64 driver to `C:\Windows\System32\Drivers\RTCore64.sys` and loads the driver.
	- Enumerate all loaded kernel drivers on the system
	- Enumerates all kernel callbacks and if `-d` selected also removes or unlinks them using the read and write IOCTL
		- Process Creation Kernel Callbacks - Removed through overwriting the callback address with `0x0`
		- Thread Creation Kernel Callbacks - Removed through overwriting the callback address with `0x0
		- Image Loading Kernel Callbacks - Removed through overwriting the callback address with `0x0
		- Registry Operations Kernel Callbacks - Removed through pointing the flink and blink to dwListHead (itself)
		- Object Creation Kernel Callbacks
			- Process - Removed through pointing the flink and blink to dwListHead (itself)
			- Thread - Removed through pointing the flink and blink to dwListHead (itself)
		- Minifilter Kernel Callbacks and their callbacknodes - Removed through unlinking the callbacknodes
	- Compare callback addresses with the loaded kernel drivers to check to which driver the callback belongs
	- Unloads the RTCore64 driver and removes the file from `C:\Windows\System32\Drivers\RTCore64.sys`
- Filenames can be configured in `config.h`

```powershell
PS C:\ > .\KernelCallbackRemover.exe
Usage: KernelCallbackRemover.exe -l / -d
Options:
  -l List Kernel Callbacks       - Lists all kernel callbacks through vulnerable driver
  -d Disable Kernel Callbacks    - Lists and remove all kernel callbacks through vulnerable driver
  -h Display this help message.
```

#### Example removing kernel callbacks
```powershell
PS C:\ > .\KernelCallbackRemover.exe -d
```

## Protection Changer
- What does it do
	- Calculates and prints offsets by enumerating windows version with `RtlGetVersion`, offsets are hardcoded.
		- Update offsets with [vergiliusproject](https://www.vergiliusproject.com/kernels/x64/windows-11/24h2/_EPROCESS) 
	- Writes vulnerable RTCore64 driver to `C:\Windows\System32\Drivers\RTCore64.sys` and loads the driver
	- Gets base address of `Ntoskrnl.exe` and calculates `PsInitialSystemProcess` offset
	- Changes the protection using the read and write IOCTL, writing the chosen protection value to the `_PS_PROTECTION Protection` struct in the `E_PROCESS`
	- Unloads the RTCore64 driver and removes the file from `C:\Windows\System32\Drivers\RTCore64.sys`
- Filenames and service names can be configured in `config.h`

```powershell
PS C:\ > .\ProtectionChanger.exe
Usage: ProtectionChanger.exe -p <PID> -v <NEW PROTECTION LEVEL>
Options:
  -p <pid>              Specify the process ID (PID) of the process to change the protection level.
  -v <protection_level> Specify the protection level value in hexadecimal (e.g., 0x00 for NO_PROTECTION).
  -h                    Display this help message.

Possible protection level values:
  0x72  PS_PROTECTED_SYSTEM               System protected process
  0x62  PS_PROTECTED_WINTCB               Windows TCB protected process
  0x52  PS_PROTECTED_WINDOWS              Windows protected process
  0x12  PS_PROTECTED_AUTHENTICODE         Authenticode protected process
  0x61  PS_PROTECTED_WINTCB_LIGHT         Windows TCB light protected process
  0x51  PS_PROTECTED_WINDOWS_LIGHT        Windows light protected process
  0x41  PS_PROTECTED_LSA_LIGHT            LSA light protected process
  0x31  PS_PROTECTED_ANTIMALWARE_LIGHT    Antimalware light protected process
  0x11  PS_PROTECTED_AUTHENTICODE_LIGHT   Authenticode light protected process
  0x00  NO_PROTECTION for no protection
```

#### Example lsass
- Example removing runasppl from LSASS process by setting the protection value to `0x00`

```powershell
PS C:\ > .\ProtectionChanger.exe -v 0x00 -p (Get-Process -Name lsass).id
```

## ETwTi Remover
- What does it do
	- Calculates and prints offsets by downloading symbols from the internet as in EDRSandBlast project
	- Writes vulnerable RTCore64 driver to `C:\Windows\System32\Drivers\RTCore64.sys` and loads the driver
	- Disables or enables ETwTi using the read and write IOCTL writing to the `ProviderEnableInfo` field
	- Unloads the RTCore64 driver and removes the file from `C:\Windows\System32\Drivers\RTCore64.sys`
- Filenames can be configured in `config.h`

```powershell
PS C:\ > .\ETwTiRemover.exe
Usage: ETwTiRemover.exe -e / -d
Options:
  -e Enable ETwTi     - set ProviderEnableInfo field within the GUID entry to 0x1
  -d Disable ETwTi    - set ProviderEnableInfo field within the GUID entry to 0x0
  -h Display this help message.
```

#### Example Disabling ETwTi

```powershell
PS C:\ > .\ETwTiRemover.exe -d
```

## Token Changer
- What does it do
	- Calculates and prints offsets by enumerating windows version with `RtlGetVersion`
		- Update offsets with [vergiliusproject](https://www.vergiliusproject.com/kernels/x64/windows-11/24h2/_EPROCESS) 
	- Writes vulnerable RTCore64 driver to `C:\Windows\System32\Drivers\RTCore64.sys` and loads the driver
	- Gets base address of `Ntoskrnl.exe` and calculates `PsInitialSystemProcess` offset
	- Changes the token value by reading the `_EX_FAST_REF Token` value and writing it into the other process using the read and write IOCTL
	- Unloads the RTCore64 driver and removes the file from `C:\Windows\System32\Drivers\RTCore64.sys

```
PS C:\ > .\TokenChanger.exe
Usage: TokenChanger.exe --tp <PID> --sp <PID>
Usage: TokenChanger.exe --edr
Usage: TokenChanger.exe --edr --sp <PID>
Options:
  --tp <pid>             Specify the target process ID (PID) to replace the token of
  --sp <pid>             Specify the source process ID (PID) to clone the token from
  --edr                  Specify to downgrade the token of all EDR processes
  --spawnsystem          Specify to spawn a new process and steal token from system
  -h                     Display this help message.
```

#### Example disabling EDR by downgrading token from explorer

```powershell
PS C:\ > .\TokenChanger.exe --edr --sp (Get-Process -Name explorer).id
```

#### Example spawning system shell

```powershell
PS C:\ > .\TokenChanger.exe --spawnsystem
```

## DSERemover
- What does it do
	- Checks if DSE protection is enabled or if test signing is enabled
	- Calculates and prints offsets by downloading symbols from the internet as in EDRSandBlast project
	- Writes vulnerable RTCore64 driver to `C:\Windows\System32\Drivers\RTCore64.sys` and loads the driver
	- Disables DSE using the read and write IOCTL, writing `0xe` (testsigning) into the kernel global variable `g_CiOptions`
	- Writes and loads a simple unsigned driver example code from my [FirstDriver](https://github.com/0xJs/FirstDriver) project to `C:\Windows\System32\Drivers\'
	- Enables DSE again using the read and write IOCTL, writing `0x6` (DSE Enabled mode) to `g_CiOptions`
	- Unloads the RTCore64 driver and removes the file from `C:\Windows\System32\Drivers\RTCore64.sys`
- Requirements;
	- Virtualized Based Security (VBS) to be disabled
- Filenames and service names can be configured in `config.h`

```powershell
PS C:\ > .\DSERemover.exe
```

# Cleanup
- The PE file should unload and remove the RTCORE driver. If it didn't then manually remove it or run the `cleanup.bat` script.
- Also removes the ROOTKIT service and driver from the DSERemover project incase its there

```cmd
sc stop RTCORE
sc delete RTCORE
sc stop ROOTKIT
sc delete ROOTKIT

del C:\Windows\System32\Drivers\RTCore64.sys
del C:\Windows\System32\Drivers\FirstDriver.sys
```

## Credits
I got inspired to expand upon the tools provided in the Evasion Lab (CETP from [Altered Security](https://www.alteredsecurity.com/evasionlab)), taught by [Saad Ahla](https://www.linkedin.com/in/saad-ahla/).

The [EDRSandblast)(https://github.com/wavestone-cdt/EDRSandblast) project for some of their code, espacially around downloading and parsing the Symbol files for offsets and the listing and removing of MiniFilters.
