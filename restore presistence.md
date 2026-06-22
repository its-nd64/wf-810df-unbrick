in the bootchain,i noticed this file:  
```
cat /lib/preinit/80_mount_root
#!/bin/sh
# Copyright (C) 2006 OpenWrt.org
# Copyright (C) 2010 Vertical Communications

do_mount_root() {
        # T5350-955: Mount root dir with tmpfs
        mount_root ram
        boot_run_hook preinit_mount_root
        [ -f /sysupgrade.tgz ] && {
                echo "- config restore -"
                cd /
                tar xzf /sysupgrade.tgz
        }
}

[ "$INITRAMFS" = "1" ] || boot_hook_add preinit_main do_mount_root
```
notice the ticket number and mount_root ram?  
thats why persist is dead, its intentional by aei!!!!  
they instead store it in an archive in rootfs_data iirc, there are many configs there but things like passwd dont stick  
after LOTS of trial and errors and arguments with claude, i found out how to pack squashfs to ubi, itb, etc etc 2 days of work is a lot so idr much
to backup rootfs ubi:
```
nand read 0x44000000 0x05D00000 0x04000000
tftpput 0x44000000 0x4000000 rootfs_u.ubi
```
to restore:
```
tftpboot 0x44000000 rootfs_u.ubi
nand erase 0x05D00000 ${filesize}
nand write 0x44000000 0x05D00000 ${filesize}
```
ik size 0x04000000 is excessive but works  
wait wtf???? i flashed the patched ubi succesfully, it mounted fine but persistence is still dead???  
```
/dev/ubi0_4 on /overlay type ubifs (rw,noatime,assert=read-only,ubi=0,vol=4)
overlayfs:/overlay on / type overlay (rw,noatime,lowerdir=/,upperdir=/overlay/upper,workdir=/overlay/work)
```

i failed, overlay mounted clean, in real rootfs_data not some /tmp/ junk but somehow persist is still dead  
i also intentionally delete aei_data thing which make it unable to be used as "overlay"(only keep configs) but it still dont work  
it stored my ash_history correctly(checked by dumping the partition in u boot and read it)  
guess thats it, i wasted way too much time on this, time to shift my focus on another big project i have in mind  


