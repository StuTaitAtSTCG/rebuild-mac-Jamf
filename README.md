# Easy Backup and Install macOS with Jamf

Provide a simple means to allow technicians to move user data, erase the Mac, reinstall the macOS and then restore the user data.

## Steps

- Create a new APFS volume to transfer accounts to
- Transfer copy of accounts to new volume
- Disable ignore macOS major update
- Download major macOS version via softwareupdate
- Begin installation of macOS with startosinstall and --eraseinstall + --preservecontainer
- Once the Macs has installed the OS and enrolled into MDM, run command to restore accounts
- After time delay delete volume used to backup accounts to


## Script - Main

```
#!/bin/bash

# Find out which disk the System volume is located on, the BackupHD volume will be created on the same disk
sysVol=$(diskutil ap listvolumegroups | grep -i -e '(Role)' | awk '/Role/ && /System/' | awk '{print $5}'  | awk -F"s[0-9]" '{print $1}')

# Create a BackupHD volume
if [ ! -d "/Volumes/BackupHD" ]; then
  echo "Creating BackupHD volume"
  /usr/sbin/diskutil ap addVolume "$sysVol" APFS BackupHD
else
  echo "BackupHD already exists"
fi

# Sync the User accounts to the BackupHD volume
echo "Syncing the User accounts to BackupHD volume"
rsync -au \
  --exclude '*/.Trash' \
  --exclude '*/Library/Application Support/CallHistoryTransactions' \
  --exclude '*/Library/Application Support/com.apple.sharedfilelist' \
  --exclude '*/Library/Application Support/com.apple.TCC' \
  --exclude '*/Library/Application Support/FileProvider' \
  --exclude '*/Library/Application Support/CallHistoryDB' \
  --exclude '*/Library/Autosave Information' \
  --exclude '*/Library/IdentityServices' \
  --exclude '*/Library/Messages' \
  --exclude '*/Library/Sharing' \
  --exclude '*/Library/Mail' \
  --exclude '*/Library/Safari' \
  --exclude '*/Library/Suggestions' \
  --exclude '*/Library/Containers/com.apple.Safari' \
  --exclude '*/Library/PersonalizationPortrait' \
  --exclude '*/Library/Metadata/CoreSpotlight' \
  --exclude '*/Library/Cookies' \
  --exclude '*/Library/Caches/CloudKit' \
  /Users/ /Volumes/BackupHD/Users

# Reset ignore settings in softwareupdate
echo "Reset ingored updates"
/usr/sbin/softwareupdate --reset-ignored

# Download macOS installer
echo "Download Catalina"
/usr/sbin/softwareupdate --fetch-full-installer --full-installer-version 10.15.7

# Erase and install macOS
echo "Begin installation"
/Applications/Install\ macOS\ Catalina.app/Contents/Resources/startosinstall --agreetolicense --eraseinstall --preservecontainer
```

## Script - Restore User Accounts

```
#!/bin/bash

# Sync the files from the backup drive to the user accounts.
if [ -d "/Volumes/BackupHD" ]; then
  echo "Syncing BackupHD accounts to Macintosh HD - Data"
  /usr/bin/rsync -au /Volumes/BackupHD/Users/ /Users
else
  echo "BackupHD does not exist"
fi
```

## Script - Delete BackupHD

```
#!/bin/bash

# Delete the BackupHD volume
if [ -d "/Volumes/BackupHD" ]; then
  echo "Deleting BackupHD volume"  
  /usr/sbin/diskutil ap deleteVolume 'BackupHD'
else
  echo "BackupHD does not exist"
fi

```

## Jamf Policies

### Backup and Rebuild macOS

- Self Service
- Ongoing
- Script - Main

### Restore User accounts

- Enrolment
- Script - Restore User Accounts

### Delete BackupHD

- Scope: Smart Group that shows enrolled Macs x many days old
- Recurring Check-in
- Once per computer
- Script - Delete BackupHD
