**Note**: By default, service usb protection `epsdaemon` doesn't run because `libidn11` is missing. Debian-based distro can download and install this library from `http://ftp.debian.org/debian/pool/main/libi/libidn/`

# OS command injection in set USB password
## Description and impact
Escan GUI has an option that set password to authenticate USB device. Set password dialog calls program `checkpassdesktop` without any data sanitization, causing OS command injection via password dialog.

## Root-cause
In method `ESetUsbPasswordDialog::okButtonokButto`, function `system` crafts system command that calls `checkkpassdesktop` without any data sanitization. Any low privilege user that can starts Escan GUI and open dialog `Set USB Password` can run arbitrary command as user's privilege.

![image](https://github.com/user-attachments/assets/92130f9d-3d93-4b89-b62d-fe5590fc37e7)

## Steps to Reprocedure
1. Open `Device Control` option from Escan Antivirus GUI

![image](https://github.com/user-attachments/assets/2a104958-b0d6-4f03-9867-3851b702a8ec)

2. Set `Use Other Password` with value `hehe; id > pwned #`. Program shows pop up `Invalid password`

![image](https://github.com/user-attachments/assets/468e2830-9b35-4a6a-b9b2-45f49a7af1f6)

3. Command executed successfully

![image](https://github.com/user-attachments/assets/6caa9a95-8c02-4a1a-b3ee-7c5bf10014e2)

# OS command injection in ask USB password.
## Description and impact
USB protection of Escan Antivirus prompts password dialog if `Ask Password` option is enabled and an USB device is plugged in. This dialog execute function `system` to call program `checkpassdesktop` without any data sanitization. Low privilege user can provide crafted data in input dialog, execute arbitrary command as root.

## Root-cause

![image](https://github.com/user-attachments/assets/6b58b738-244b-4f36-8cd3-4df21324dfad)

## Proof of concept
TODO: condition: must enable ask password option
1. Plug USB and wait for password prompt

![image](https://github.com/user-attachments/assets/1272ab51-0d1c-46e4-806c-2bb8e062797e)

2. Enter password ```ha `id > /executed` he```

![image](https://github.com/user-attachments/assets/f25fc0b1-91cb-48fb-ba71-f8f0d9f8dbc5)

3. Command executed successfully

![image](https://github.com/user-attachments/assets/fda49076-49c8-46d1-95e9-f1f93d5417e2)

# OS command injection in autoscan USB
## Description and impact
Escan Antivirus has `AutoScan` USB option by default. When an USB device is plugged in, service `epsdaemon` executes function `system` that calls program `autoscanusb.sh` with mount point of USB as parameter. Value mount point is not santized, allowing crafted USB name executes OS command as `root` as soon as USB is plugged in.

## Root-cause

Service `epsdaemon` uses function `sprintf` to craft command without any data validation.

![image](https://github.com/user-attachments/assets/9082474d-f7f2-4730-a8da-27f588749840)

Full command is used in `system` causing OS command injection.

![image](https://github.com/user-attachments/assets/21896fd7-b235-4d57-b7fc-7cbe11b1c4e6)

**Important notes**:
- The `Auto Scan USB` thread is unstable because of the service itself. The problem is from poorly developed product rather than unstable exploit.
- Device name has limit length to 12 characters and a lot special characters are not allowed. OS Command Injection is limited.

## Steps-to-reprocedure
1. Create USB with name `id` and plug in

![image](https://github.com/user-attachments/assets/159e8734-4ee9-45b6-98b9-3d887becdfca)

2. Escan starts scan command automatically

![image](https://github.com/user-attachments/assets/77672a6b-3ec1-41d1-9751-45f11243dbe1)

![image](https://github.com/user-attachments/assets/04fed841-fb32-47d0-9d11-5cab893acf39)

# Stack-based buffer overflow in Serial Number of passPrompt
## Description and impact
Program `passPrompt` takes Serial Number of USB device, save to a stack-based variable without any length check. This cause program crashes and register **Base Pointer** is manipulated.

## Root-cause

Program uses function `sprintf` to craft a string.

![image](https://github.com/user-attachments/assets/721895e9-7865-4380-9416-99531342ec6f)

## Steps to reprocedure
1. Start `passPrompt` with `gdb`
2. Run command `run $(python3 -c "print('a' * 11496)") hehe`
3. Program crashes, some registers like `RBP` is overwritten

![image](https://github.com/user-attachments/assets/646b8914-bec8-41e4-8370-eeff68bead4c)

# Stack-based buffer overflow in Manufacturer Name of passPrompt
## Description and impact
Program `passPrompt` takes Manufacturer Name of USB device, save to a stack-based variable without any length check. This cause program crashes and register **Base Pointer** is manipulated.

## Root-cause
Program uses `strcpy` to copy value to a stack-based variable.

![image](https://github.com/user-attachments/assets/61f7c30e-e99f-49d7-9c1c-f88e76038b58)

## Steps to reprocedure
1. Start `passPromp` with `gdb`
2. Run command `run hehe $(python3 -c "print('a' * 10496)"`
3. Program crashes. Registers like `RBP` is overwritten

![image](https://github.com/user-attachments/assets/cdf55a31-a8f1-41af-baf3-650ec93c9fe3)

# Stack-based buffer overflow in VirusPopUp
## Description and impact
Program `VirusPopUp` shows a popup of infected binary to current user that's using graphic interface. Program takes information of infected file from command line parameter, copy to a stack using function `strcpy`. This causes stack-based overflow that overwrites value of **Base Pointer**

## Root-cause

Variable that saves data from parameter is saved on stack.

![image](https://github.com/user-attachments/assets/0ec391da-8b53-47d8-b64d-52282f9a6cdb)

Program uses `strcpy` to copy value which cause buffer-overflow.

![image](https://github.com/user-attachments/assets/94effc66-c511-4a5f-8251-0e55556e7c2a)

## Steps to reprocedure

Run `VirusPopUp` with long data in GDB. Program crashes and **Base Pointer** is overwritten.

![image](https://github.com/user-attachments/assets/88d6fd08-1e17-4b4f-b855-6079cef7c0f9)

# Stack-based Buffer overflow in USB password
## Description and Impact

Program uses `sprintf` to craft system command without checking variable's length. This cause buffer overflow, making service crashes when user provide very long password.

## Root-cause
Program uses `sprintf` to craft system command without checking variable's length.

![image](https://github.com/user-attachments/assets/42dba52b-b4e1-44f6-9a8c-3dff673344a5)

# Heap-Based buffer-Overflow in ReadConfiguration.
## Description and Impact
Escan Antivirus has method `ReadConfiguration` that saves value into object without checking data length. Any option that is parsed as string, such as `BasesPath`, `TempPath`, `LicenseKey`, cause multiple programs of Escan AV crash. This problem affect `escan`, `escangui` and serveral different binaries depends on settings of programs.

## Steps to reprocedure

1. Modify config file `/opt/MicroWorld/etc/mwav.conf` as root, modify value `BasePath`

![image](https://github.com/user-attachments/assets/e0c2782c-3bc2-4734-be3e-50209915f97a)

2. Run program `escan`. Program crashes

![image](https://github.com/user-attachments/assets/c4ae8928-1213-4d2d-8ad1-5bd0dce5a88f)



