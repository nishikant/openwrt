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

### Execution Order
1. /etc/init.d/resize-root
2. /etc/init.d/tune-root
3. /etc/init.d/resize2fs-root
4. /etc/init.d/custom-package-install

### Step 1: resize-root
Increase the actual partition size using sfdisk. /etc/init.d/resize-root does
this and enables next step.

### Step 2: tune-root
Adjust inode size of the partition (using tune2fs) and run filesystem check to fix issues.
Failing to run this step corrupts the filesystem and openwrt crashes and you
will have to do a full install. /etc/init.d/tune-root does this and enables next
step.

### Step 3: resize2fs-root
Extend ext4 filesystem using resize2fs. /etc/init.d/resize2fs-root does this and
enables custom-pacakge-install.

### Step 4: custom-package-install
The official online OpenWrt Firmware Selector enforces an arbitrary partition
constraint (around 100MB for x86 builds) when generating pre-compiled images. To
ensure all the custom packages are installed after upgrade, this script is
added. This will install packages I use to customize my setup like adguardhome,
minidlna, etc.


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
/etc/init.d/custom-package-install
```

/etc/rc.d/S99resize-root this file is critical to execute disk extension
automatically. It basically makes sure resize-root service is started on bootup.
