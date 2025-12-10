# Linux Backup & Restore Automation System

## Overview
This project implements an automated backup and restore system for Linux using Bash scripting.  
Features include encrypted backups, retention policy, logging, cron automation, and a restore workflow.  
The goal is to create a secure, reliable, and fully auditable backup pipeline suitable for SysAdmin and SecOps environments.

---

## Features
- Automated backup creation using `tar`
- AES256-encrypted archives using GPG
- Backup retention policy (removal of files older than defined days)
- Detailed logging of each backup event
- Restore script with decryption and extraction
- Cron-ready design for scheduled backups

## Prerequisites
- Linux system (Debian/Ubuntu recommended)
- Root privileges (required for backing up system directories)
- `/backups` directory created with appropriate permissions

---

## Configuration

### Backup Source
The directory being backed up is defined in `backup.sh`:

```
SOURCE="/etc"
```

This can be changed to include additional directories.

### Backup Destination
All backups and logs are stored in:

```
DEST="/backups"
```

Ensure this directory exists:

```
sudo mkdir /backups
sudo chmod 700 /backups
```

### Encryption Passphrase
The GPG passphrase is stored in a secure file readable only by root:

```
sudo nano /root/backup_passphrase.txt
sudo chmod 600 /root/backup_passphrase.txt
```

---

## backup.sh (Automated Encrypted Backup Script)

The script performs:

1. Timestamped archive creation  
2. AES256 encryption  
3. Logging  
4. Retention policy enforcement  

```
#!/bin/bash

SOURCE="/etc"
DEST="/backups"
LOG="/backups/backup.log"
DATE=$(date +"%Y-%m-%d_%H-%M-%S")
ARCHIVE="backup_$DATE.tar.gz"

echo "Starting backup: $DATE" >> $LOG

# Create encrypted backup
tar -czf - $SOURCE | gpg --batch --yes --passphrase-file /root/backup_passphrase.txt -c > $DEST/$ARCHIVE.gpg

if [ $? -eq 0 ]; then
    echo "Backup successful: $ARCHIVE.gpg" >> $LOG
else
    echo "Backup FAILED: $ARCHIVE.gpg" >> $LOG
fi

# Log size
SIZE=$(du -h $DEST/$ARCHIVE.gpg | cut -f1)
echo "Backup size: $SIZE" >> $LOG

# Retention Policy
RETENTION_DAYS=7
find "$DEST" -type f -name "backup_*.gpg" -mtime +$RETENTION_DAYS -exec rm {} \;
echo "Old backups older than $RETENTION_DAYS days removed" >> $LOG

echo "-----------------------------" >> $LOG
```

Make executable:

```
chmod +x backup.sh
```

Run manually:

```
sudo ./backup.sh
```

---

## restore.sh (Decryption + Restore Script)

```
#!/bin/bash

BACKUP_FILE=$1
TARGET="/"

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: ./restore.sh <backup file>"
    exit 1
fi

if [ ! -f "$BACKUP_FILE" ]; then
    echo "Backup file not found."
    exit 1
fi

# Decrypt backup
gpg --batch --yes --passphrase-file /root/backup_passphrase.txt -o /tmp/decrypted_backup.tar.gz -d "$BACKUP_FILE"

# Extract archive
sudo tar -xzf /tmp/decrypted_backup.tar.gz -C $TARGET

rm /tmp/decrypted_backup.tar.gz
echo "Restore complete."
```

Make executable:

```
chmod +x restore.sh
```

---

## Cron Automation

Schedule daily backups at 2 AM:

```
sudo crontab -e
```

Add:

```
0 2 * * * /backups/backup.sh
```

Verify:

```
crontab -l
```

---

## Validation Steps
To verify the system works:

1. Run `sudo ./backup.sh`
2. Confirm encrypted file exists in `/backups`
3. Decrypt test file:

```
sudo gpg --batch --yes --passphrase-file /root/backup_passphrase.txt -d /backups/<file>.gpg > /tmp/test_backup.tar.gz
```

4. Extract test archive:

```
mkdir /tmp/restore_test
sudo tar -xzf /tmp/test_backup.tar.gz -C /tmp/restore_test
```

5. Check `/tmp/restore_test/etc`

---

## Screenshots
<img width="648" height="210" alt="image" src="https://github.com/user-attachments/assets/bd0e6918-04ab-4b77-a99c-fcba8b7e37d3" />
<img width="584" height="637" alt="image" src="https://github.com/user-attachments/assets/7df0f4fb-80e1-43bd-b5fb-e5380e4d53be" />
<img width="753" height="494" alt="image" src="https://github.com/user-attachments/assets/65ba4cda-0f0a-4989-b9b8-77fae6c24235" />
<img width="519" height="370" alt="image" src="https://github.com/user-attachments/assets/733df7f7-24ed-4d2c-89d0-434b51ebff9d" />

---

## Lessons Learned
- Implementing secure backups requires proper encryption, permissions, and validation.
- Retention policies prevent uncontrolled storage growth.
- Restore procedures must be tested to ensure backup integrity.
- Automating backups with cron ensures consistent system protection.







