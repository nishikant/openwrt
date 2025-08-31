# Extend OPENWRT root (/) filesystem automatically on Raspberry Pi 5

OpenWRT by default creates a root (/) partition of ext4 type of size 100MB. For
folks running OpenWRT on a Raspberry Pi, this means manually extending their
root (/) filesystem everytime they upgrade openwrt.

Actual disk extend activity is done only once on a fresh upgrade by creating
marker files. Logs can be found in /root/resize.log. Marker files are
/root/.resize-step1, /root/.resize-step2, /root/.resize-step3. DO NOT DELETE
THESE files.

Below setup will help configure openwrt such that after upgrade through
sysupgrade or attended-sysupgrade, the root partition will be extended to
consume all the free space, in my case 30GB.

## Copy the scripts to /etc/init.d
Copy resize-root, tune-root and resize2fs-root in /etc/init.d/ folder. Make sure
the are executable.

```
git clone git@github.com:nishikant/openwrt.git

rsync resize-root root@<openwrt_ip>:/etc/init.d/
rsync tune-root root@<openwrt_ip>:/etc/init.d/
rsync resize2fs-root root@<openwrt_ip>:/etc/init.d/
rsync sysupgrade.conf root@<openwrt_ip>:/etc/

```
These represent 3 distinct steps in extending the root partition. After each
step a reboot is required.

### Step 1
Increase the actual partition size using sfdisk.

### Step 2
Adjust inode size of the partition (using tune2fs) and run filesystem check to fix issues.
Failing to run this step corrupts the filesystem and openwrt crashes and you
will have to do a full install.

### Step 3
Extend ext4 filesystem using resize2fs.

## Enable resize-root service

Under System >> Startup, search for resize-root and enable the service. Make
sure tune-root and resize2fs-root services are disabled. These will be
automatically enabled by resize-root only once and later disabled.

## Update backup file list (sysupgrade.conf)

Openwrt sysupgrade or attended-sysupgrade backups certain files and restores
them after upgrade. This is controlled by /etc/sysupgrade.conf.

You can directly add full path of above files or use LUCI System >> Backup/Flash
Firmware >> Configuration and append below lines

```
/etc/init.d/resize-root
/etc/init.d/resize2fs-root
/etc/init.d/tune-root
/etc/rc.d/S99resize-root
```

/etc/rc.d/S99resize-root this file is critical to execute disk extension
automatically. It basically makes sure resize-root service is started on bootup.
