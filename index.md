# **TryHackMe: "Extracted" — Full CTF Write-Up**

### **Network Forensics, KeePass Exfiltration & Memory Forensics**

---

## **Overview**

This TryHackMe lab covers a realistic DFIR (Digital Forensics & Incident Response) scenario involving:

* Identifying malicious network activity in a packet capture  
* Extracting and reverse-engineering obfuscated exfiltrated data  
* Recovering credentials from a compromised KeePass database using CVE-2023-32784

**Room Link:** https://tryhackme.com/room/extractedroom

---

## **Files & Artifacts Collected**

During the investigation the following files were created and recovered:

| File | Description |
| ----- | ----- |
| `traffic.pcapng` | Original network capture |
| `1337_hex.txt` | Raw hex of TCP stream (port 1337\) — memory dump |
| `1337_hex_bytes.txt` | Intermediate decoded bytes |
| `1338_hex.txt` | Raw hex of TCP stream (port 1338\) — KeePass DB |
| `1338_hex_bytes.txt` | Intermediate decoded bytes |
| `recovered_dump.dmp` | Decoded KeePass memory dump |
| `recovered_database.kdbx` | Decoded KeePass database |
| `xor_decode.py` | Custom decoder script |
| `dns_logs.json` | DNS logs artifact |
| `ssh_logs.json` | SSH logs artifact |

---

## **Step 1 — Initial Traffic Analysis in Wireshark**

Opened `traffic.pcapng` in Wireshark. Two IPs immediately stood out:

* **Victim host:** `10.10.45.95`  
* **Suspicious source:** `10.10.94.106`

Using **Analyze \> Conversations** revealed TCP-dominated traffic with HTTP and ARP. The first suspicious indicators were:

* **Line 4** — `GET` request for a `.ps1` (PowerShell) file  
* **Line 8** — `200 OK` response

The URL in the request:

http://10.10.94.106:1339/xxxmmdcclxxxiv.ps1

A PowerShell script being downloaded over HTTP is immediately suspicious and was extracted via **File \> Export Objects \> HTTP**.

---

## **Step 2 — Malicious PowerShell Script Analysis**

The downloaded script was heavily obfuscated. Despite the obfuscation, its behavior was clear:

1. Downloads living-off-the-land tools (ProcDump / Sysinternals)  
2. Checks if **KeePass** is running  
3. If yes, **dumps the KeePass process memory**  
4. Obfuscates the dump using **XOR \+ Base64**  
5. Exfiltrates the dump to a C2 server  
6. Separately exfiltrates the **KeePass `.kdbx` database file**

### **C2 Server Details**

The `$sERveRIP` variable was hex-encoded. Decoding revealed:

printf "%d.%d.%d.%d\\n" 0xa0 0xa5 0xe 0x6a  
\# Result: 160.165.14.106

### **XOR Keys Identified**

| Target | Port | XOR Key |
| ----- | ----- | ----- |
| KeePass memory dump | 1337 | `0x41` |
| KeePass database (.kdbx) | 1338 | `0x42` |

---

## **Step 3 — Extracting Exfiltrated Data with TShark**

Filtered Wireshark to:

tcp.port \== 1337 || tcp.port \== 1338

### **Port 1337 — Memory Dump**

The TCP stream was extremely large. Used **TShark** to extract it efficiently:

tshark \-r traffic.pcapng \-q \-z follow,tcp,raw,1 | tail \-n \+7 | head \-n \-1 \> 1337\_hex.txt

### **Port 1338 — KeePass Database**

A much smaller stream (Stream 2), containing Base64-like strings:

tshark \-r traffic.pcapng \-q \-z follow,tcp,raw,2 | tail \-n \+7 | head \-n \-1 \> 1338\_hex.txt

---

## **Step 4 — Deobfuscating the Data**

Created `xor_decode.py` to reverse the encoding pipeline: **Hex → Base64 Decode → XOR Decrypt**

### **Decode the Memory Dump (XOR key 0x41):**

python3 xor\_decode.py 1337\_hex.txt recovered\_dump.dmp \--xor 0x41

Verified with:

file recovered\_dump.dmp  
\# Output: Windows minidump (MDMP signature detected)

### **Decode the KeePass Database (XOR key 0x42):**

python3 xor\_decode.py 1338\_hex.txt recovered\_database.kdbx \--xor 0x42

Verified KeePass signature `\x03\xd9\xa2\x9a` detected successfully.

---

## **Step 5 — Extract Partial Password from Memory Dump**

Using **CVE-2023-32784** and the tool [keepass-dump-masterkey](https://github.com/matro7sh/keepass-dump-masterkey):

cd \~/keepass-dump-masterkey  
python3 poc.py /home/kali/Downloads/recovered\_dump.dmp

Output:

2026-05-11 03:33:41,251 \[.\] \[main\] Opened /home/kali/Downloads/recovered\_dump.dmp  
Possible password: ●NoWaYIcanF0rGetThis123  
Possible password: ●RoWaYIcanF0rGetThis123  
Possible password: ●AoWaYIcanF0rGetThis123  
Possible password: ●,oWaYIcanF0rGetThis123  
Possible password: ●?oWaYIcanF0rGetThis123  
Possible password: ●8oWaYIcanF0rGetThis123

The `●` represents the **missing first character** — a known limitation of CVE-2023-32784. The most consistent candidate was `NoWaYIcanF0rGetThis123`.

---

## **Step 6 — Generate Single-Character Wordlist**

Created `gen_wordlist.py` and generated all possible single-character prefixes:

python3 gen\_wordlist.py \-m 1 \-o prefixes.txt  
\# \[+\] Generated 92 prefixes

---

## **Step 7 — Brute-Force the Missing Character**

Created `KP-brute-force.sh` with:

* `KNOWN_SUFFIX="NoWaYIcanF0rGetThis123"`  
* `DATABASE="/home/kali/Downloads/recovered_database.kdbx"`

chmod \+x KP-brute-force.sh  
./KP-brute-force.sh

Output:

\[\*\] Starting brute force...  
\[\*\] Trying: 92 prefixes  
\[+\] Password found: ?NoWaYIcanF0rGetThis123

**Full Master Password: `?NoWaYIcanF0rGetThis123`**

---

## **Step 8 — Unlock the KeePass Database**

Listed all entries in the database:

echo "?NoWaYIcanF0rGetThis123" | keepassxc-cli ls /home/kali/Downloads/recovered\_database.kdbx

Output:

Sample Entry  
Sample Entry \#2  
You win\!  
General/  
Windows/  
Network/  
Internet/  
eMail/  
Homebanking/

---

## **Step 9 — Retrieve the Flag**

Opened KeePassXC GUI (due to TTY issues in VMware with CLI):

keepassxc /home/kali/Downloads/recovered\_database.kdbx

Entered the master password `?NoWaYIcanF0rGetThis123` and opened the **"You win\!"** entry.

The Notes field of the "You win\!" entry revealed the flag:

THM{B3tt3r\_...}

The KeePassXC database opened successfully showing 3 entries: Sample Entry, Sample Entry \#2, and "You win\!" — the flag was visible in the Notes column of the "You win\!" entry.

---

## **Full Attack Chain Summary**

traffic.pcapng  
    └── HTTP GET → malicious .ps1 PowerShell script  
            └── Script targets KeePass process  
                    ├── Dumps process memory → XOR(0x41) \+ Base64 → Port 1337  
                    └── Exfiltrates .kdbx DB → XOR(0x42) \+ Base64 → Port 1338  
                            │  
                            ▼  
                    xor\_decode.py reverses encoding  
                            │  
                    recovered\_dump.dmp \+ recovered\_database.kdbx  
                            │  
                    CVE-2023-32784 extracts partial password  
                            │  
                    Brute-force missing first character  
                            │  
                    ?NoWaYIcanF0rGetThis123  
                            │  
                    KeePassXC → "You win\!" → FLAG

---

## **Tools Used**

| Tool | Purpose |
| ----- | ----- |
| Wireshark | Network traffic analysis |
| TShark | Efficient stream extraction |
| Python3 (xor\_decode.py) | Reverse XOR+Base64 encoding |
| keepass-dump-masterkey | CVE-2023-32784 exploitation |
| gen\_wordlist.py | Single-char wordlist generation |
| KP-brute-force.sh | Brute-force missing character |
| keepassxc-cli / KeePassXC | Database unlocking |

---

## **Key Takeaways**

This lab demonstrates how multiple weaknesses chain together into full credential compromise:

1. **No egress filtering** — PowerShell script downloaded freely over HTTP  
2. **Outdated KeePass** — CVE-2023-32784 affects versions prior to 2.54  
3. **Weak cryptographic practice** — Single-byte XOR is trivially reversible  
4. **No endpoint detection** — ProcDump execution went undetected  
5. **Sensitive data unprotected at rest** — KeePass DB exfiltrated successfully

---

*Write-up based on TryHackMe "Extracted" room — completed May 2026*

