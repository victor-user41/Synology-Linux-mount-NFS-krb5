# Synology-Linux-mount-NFS-krb5
Synology Directory Server Kerberos Linux mount NFS krb5

# Kerberized NFS Setup Guide

Complete guide for setting up Kerberized NFS between Synology NAS and Linux client.

## Official Documentation
https://kb.synology.com/en-eu/DSM/tutorial/how_to_set_up_kerberized_NFS

## Environment Information

**Synology NAS:**
- IP: 192.168.1.153
- Web Interface: http://192.168.1.153:5000/
- Hostname: mynas
- Admin Username: ADMIN_USER_NAME

**Linux Client:**
- IP: 192.168.1.160
- Username: victor_user
- Hostname: victor-vb

## Prerequisites - Linux Client Information

Get your user ID:
```bash
id -u
```

Get your hostname:
```bash
hostname
```

## Part 1: Synology NAS Configuration

### 1. Create Synology User
Create a user account `YOU_USER_NAME` in Synology DSM.

### 2. Configure Directory Server
Navigate to Synology Directory Server and create a domain:
- Domain name: `MYHOMENAS.LOCAL`
- WorkGroup: `MYHOMENAS`

### 3. Create Directory Server User
Since Synology users are not automatically Directory Server users, create a matching user `YOU_USER_NAME` in Directory Server.

### 4. Configure Shared Folder
1. Create a shared folder in Control Panel > Shared Folder
2. Add NFS permissions for client IP 192.168.1.160
3. Create temporary folder for exporting keys: `/volume1/homes/ADMIN_USER_NAME/tempFolder`
4. Create the folder to be mounted: `nfsMount`

### 5. SSH into Synology and Configure Kerberos

Connect via SSH:
```bash
ssh ADMIN_USER_NAME@192.168.1.153
```

Switch to root:
```bash
sudo -i
```

Create computer account (may already exist from domain creation):
```bash
samba-tool computer create MYNAS$
```

Create SPNs for NFS service:
```bash
samba-tool spn add nfs/mynas.myhomenas.local MYNAS$
samba-tool spn add nfs/myhomenas.local MYNAS$
```

Verify SPNs:
```bash
samba-tool spn list 'MYNAS$' | grep nfs
```

Export keytabs for NFS service:
```bash
samba-tool domain exportkeytab /volume1/homes/ADMIN_USER_NAME/tempFolder/nfs.keytab --principal=nfs/mynas.myhomenas.local
samba-tool domain exportkeytab /volume1/homes/ADMIN_USER_NAME/tempFolder/nfs.keytab --principal=nfs/myhomenas.local
```

### 6. Configure Client Computer Account

Create computer account for client:
```bash
samba-tool computer create VICTOR-VB$
samba-tool user setpassword VICTOR-VB$ --random
```

Add SPNs for client:
```bash
samba-tool spn add host/victor-vb.myhomenas.local VICTOR-VB$
samba-tool spn add host/victor-vb VICTOR-VB$
samba-tool spn add nfs/victor-vb.myhomenas.local VICTOR-VB$
samba-tool spn add nfs/victor-vb VICTOR-VB$
```

Verify client SPNs:
```bash
samba-tool spn list 'VICTOR-VB$'
```

Export client keytabs:
```bash
samba-tool domain exportkeytab /volume1/homes/ADMIN_USER_NAME/tempFolder/krb5.keytab --principal='VICTOR-VB$@MYHOMENAS.LOCAL'
samba-tool domain exportkeytab /volume1/homes/ADMIN_USER_NAME/tempFolder/krb5.keytab --principal=victor@MYHOMENAS.LOCAL
```

Download the `krb5.keytab` file from the Synology to your Linux client.

## Part 2: Linux Client Configuration

### 1. Install Required Packages

```bash
sudo apt install krb5-user nfs-common
sudo systemctl start rpc-gssd.service
```

### 2. Install Keytab File

Copy keytab to system location:
```bash
sudo cp ~/Downloads/krb5.keytab /etc/krb5.keytab
sudo chown root:root /etc/krb5.keytab
sudo chmod 600 /etc/krb5.keytab
```

Set ACL permissions (required for service to start):
```bash
sudo setfacl -m u:victor_user:r /etc/krb5.keytab
```

Verify keytab:
```bash
sudo klist -k /etc/krb5.keytab
```

### 3. Configure ID Mapping

Edit `/etc/idmapd.conf`:
```ini
[General]
Domain = myhomenas.local
```

### 4. Configure Kerberos

Edit `/etc/krb5.conf`:
```ini
[libdefaults]
    default_realm = MYHOMENAS.LOCAL
    dns_lookup_realm = false
    dns_lookup_kdc = false
    rdns = false

[realms]
    MYHOMENAS.LOCAL = {
        kdc = mynas.myhomenas.local
        kdc = 192.168.1.153
        admin_server = mynas.myhomenas.local
        admin_server = 192.168.1.153
    }

[domain_realm]
    .myhomenas.local = MYHOMENAS.LOCAL
    myhomenas.local = MYHOMENAS.LOCAL
```

### 5. Configure DNS Resolution

Edit `/etc/systemd/resolved.conf`:
```ini
[Resolve]
DNS=192.168.1.153
Domains=mynas.myhomenas.local
DNSStubListener=no
```

Restart DNS service:
```bash
sudo systemctl restart systemd-resolved
```

## Network Configuration Files

### Hosts File Configuration

Edit `/etc/hosts` and add:
```
192.168.1.153    mynas.myhomenas.local mynas myhomenas.local
```

**Note:** This configuration allows hostname resolution without requiring DNS queries. The entry maps the Synology NAS IP address to its fully qualified domain name (FQDN) and short names.


### 6. Create Kerberos Ticket Service

Create `/etc/systemd/system/nfs-kerberos.service`:
```ini
[Unit]
Description=Get Kerberos ticket for NFS (User victor)
Wants=network-online.target
After=network-online.target
Before=remote-fs-pre.target

[Service]
Type=oneshot
User=victor_user
Environment=KRB5CCNAME=FILE:/tmp/krb5cc_1000
ExecStart=/bin/sh -c "/usr/bin/kinit -kt /etc/krb5.keytab victor@MYHOMENAS.LOCAL && /usr/bin/kvno nfs/mynas.myhomenas.local"
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable and start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable nfs-kerberos.service
sudo systemctl start nfs-kerberos.service
sudo systemctl status nfs-kerberos.service
```

### 7. Create Kerberos Ticket Refresh Timer

Create `/etc/systemd/system/nfs-kerberos.timer`:
```ini
[Unit]
Description=Refresh Kerberos ticket every 4 hours

[Timer]
OnBootSec=1min
OnUnitActiveSec=4h
Unit=nfs-kerberos.service

[Install]
WantedBy=timers.target
```

Enable the timer:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now nfs-kerberos.timer
```

### 8. Mount NFS Share

Manual mount:
```bash
sudo mount -t nfs4 -o sec=krb5,vers=4.0,rw myhomenas.local:/volume1/nfsMount /home/victor_user/nfsMount
```

For automatic mounting at boot, add to `/etc/fstab`:
```
myhomenas.local:/volume1/nfsMount /home/victor_user/nfsMount nfs4 sec=krb5,vers=4.0,rw,auto,_netdev 0 0
```

## Summary

This setup provides:
- Secure Kerberized NFS authentication
- Automatic ticket renewal every 4 hours
- Persistent mount configuration
- Proper domain integration between Synology and Linux client

## Troubleshooting

If you encounter issues:

1. Check Kerberos ticket status:
```bash
klist
```

2. Verify keytab contents:
```bash
sudo klist -k /etc/krb5.keytab
```

3. Check NFS service status:
```bash
sudo systemctl status rpc-gssd.service
sudo systemctl status nfs-kerberos.service
```

4. Test Kerberos authentication:
```bash
kinit -kt /etc/krb5.keytab victor@MYHOMENAS.LOCAL
```

5. Check mount status:
```bash
mount | grep nfs
```

## License

This guide is provided as-is for educational purposes.

## Contributing

Feel free to submit issues or pull requests if you find any errors or have suggestions for improvements.
