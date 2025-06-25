# LDAP Centralized Authentication with NFS and Application Access

Implemented centralized authentication across all systems in the company network using LDAP and NFS. The setup allows users to log into any system using their LDAP credentials and automatically mounts their personal data from a shared NFS server, ensuring persistent access regardless of the login location. System environment variables were configured to store user-specific session data in their home directories, and commonly used applications were made accessible to LDAP users, providing a consistent and reliable working environment across all machines.
This project was completed as part of my internship at MediaAMP.

## Overview

- **LDAP**: Centralized authentication allowing users to log in to any system with LDAP credentials
- **NFS**: Shared home directories for accessible user data on any system
- **Session Management**: Configures environment variables to store application data in home directories
- **Application Access**: Ensures system applications (e.g., Postman) are available to LDAP users
- **Sudo Access**: Grants administrative privileges to LDAP users in designated groups

## Setup Instructions

### 1. Transfer Local User Data to LDAP User Directory

```bash
scp -r * localuser@nfs-server-ip:/srv/nfs/home/user1
```

> **Note**: Replace `localuser`, `nfs-server-ip`, and `user1` with appropriate credentials and server details.

### 2. Configure LDAP Client Systems

Install required packages:
```bash
sudo apt-get install libpam-ldap nscd
```

Edit `/etc/nsswitch.conf`:
```bash
sudo nano /etc/nsswitch.conf
```
Add:
```
passwd: compat ldap
group: compat ldap
shadow: compat ldap
```

Configure PAM for home directories:
```bash
sudo nano /etc/pam.d/common-session
```
Add:
```
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

Restart services:
```bash
sudo /etc/init.d/nscd restart
getent passwd user1  # Verify LDAP user lookup
sudo dpkg-reconfigure libpam-runtime
```

### 3. Optimize LDAP Login Speed with SSSD

Install SSSD:
```bash
sudo apt install sssd sssd-tools
```

Configure `/etc/sssd/sssd.conf`:
```ini
[sssd]
services = nss, pam
config_file_version = 2
domains = example.local

[domain/example.local]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldap://ldap-server-ip
ldap_search_base = dc=example,dc=local
ldap_default_bind_dn = cn=admin,dc=example,dc=local
ldap_default_authtok = ********
cache_credentials = true
enumerate = true
ldap_id_use_start_tls = false
ldap_tls_reqcert = allow
```

Set permissions and restart:
```bash
sudo chmod 600 /etc/sssd/sssd.conf
sudo chown root:root /etc/sssd/sssd.conf
sudo systemctl enable sssd
sudo systemctl restart sssd
sudo systemctl status sssd
```

### 4. Configure NFS Client for Shared Home Directories

Install and configure autofs:
```bash
sudo apt update
sudo apt install autofs
```

Edit `/etc/auto.master`:
```
+auto.master
/home/users /etc/auto.ldapusers --timeout=60
```

Configure NFS mounts in `/etc/auto.ldapusers`:
```
* -rw,sync,nosuid nfs-server-ip:/srv/nfs/home/&
```

Restart autofs:
```bash
sudo systemctl restart autofs
```

### 5. Grant Sudo Access to LDAP Users

Edit sudoers file:
```bash
sudo visudo
```
Add:
```
%ldapgroup ALL=(ALL:ALL) ALL
```

### 6. Test LDAP Login and NFS Mount

```bash
cd /home/users/user1
ls
```

### 7. Resolve Session Expiry Issues

Create XDG fix script:
```bash
sudo bash -c 'cat > /etc/profile.d/fix-xdg-env.sh <<EOF
#!/bin/bash
[ -z "\$XDG_CONFIG_HOME" ] || [[ "\$XDG_CONFIG_HOME" == /tmp/* ]] && export XDG_CONFIG_HOME="\$HOME/.config"
[ -z "\$XDG_CACHE_HOME" ] || [[ "\$XDG_CACHE_HOME" == /tmp/* ]] && export XDG_CACHE_HOME="\$HOME/.cache"
EOF'
sudo chmod +x /etc/profile.d/fix-xdg-env.sh
```

Create template for new users:
```bash
sudo bash -c 'echo -e "export XDG_CONFIG_HOME=\$HOME/.config\nexport XDG_CACHE_HOME=\$HOME/.cache" > /etc/skel/.xprofile'
```

Apply to existing users:
```bash
for user in $(getent passwd | grep /home/users | cut -d: -f1); do
    if [ -d /home/users/$user ]; then
        sudo cp /etc/skel/.xprofile /home/users/$user/
        group=$(id -gn $user 2>/dev/null || echo ldapgroup)
        sudo chown $user:$group /home/users/$user/.xprofile
    else
        echo "Home directory for $user does not exist, skipping."
    fi
done
```

Reboot and test:
```bash
sudo reboot
echo $XDG_CONFIG_HOME  # Should return: $HOME/.config
echo $XDG_CACHE_HOME   # Should return: $HOME/.cache
```

### 8. Install Postman for LDAP Users

Install Postman:
```bash
sudo apt update
sudo apt install curl -y
sudo snap remove postman
sudo mkdir -p /opt/postman
sudo curl -L https://dl.pstmn.io/download/latest/linux64 -o /tmp/postman.tar.gz
sudo tar -xzf /tmp/postman.tar.gz -C /opt/postman --strip-components=1
sudo ln -sf /opt/postman/Postman /usr/bin/postman
```

Create desktop launcher:
```bash
sudo tee /usr/share/applications/postman.desktop > /dev/null <<EOF
[Desktop Entry]
Encoding=UTF-8
Name=Postman
Exec=/opt/postman/Postman
Icon=/opt/postman/app/resources/app/assets/icon.png
Terminal=false
Type=Application
Categories=Development;Utility;
StartupNotify=true
EOF
```

## Troubleshooting

- **LDAP User Not Found**: Verify `nsswitch.conf` includes ldap and SSSD is active (`systemctl status sssd`)
- **NFS Mount Fails**: Check autofs logs (`journalctl -u autofs`) and NFS server connectivity
- **Postman Fails to Launch**: Confirm `$DISPLAY` is set and desktop launcher path is correct
- **XDG Variables Point to /tmp**: Ensure fix scripts and `.xprofile` are applied correctly
