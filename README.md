# less-3

Домашнее задание  
1) Уменьшить том под / до 8G
2) Выделить том под /home
3) Выделить том под /var - сделать в mirror
4) /home - сделать том для снапшотов
5) Прописать монтирование в fstab. 

Работа со снапшотами:
- сгенерить файлы в /home/
- снять снапшот
- удалит часть файлов
- восстановитсā со снапшота

----- Уменьшить том под / до 8G -----  

alex-linux@alexlinux:~/linux-vm/Less-3$ vagrant ssh  

Создание временно тома, раздела для переноса root  

[vagrant@lvm ~]$ lsblk  
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT  
sda                       8:0    0   40G  0 disk   
├─sda1                    8:1    0    1M  0 part   
├─sda2                    8:2    0    1G  0 part /boot  
└─sda3                    8:3    0   39G  0 part   
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /  
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]  
sdb                       8:16   0   10G  0 disk   
sdc                       8:32   0    2G  0 disk  
sdd                       8:48   0    1G  0 disk   
sde                       8:64   0    1G  0 disk   

[vagrant@lvm ~]$ df -hT смотрим сколько заминмает раздел root  

Filesystem                      Type      Size  Used Avail Use% Mounted on  
/dev/mapper/VolGroup00-LogVol00 xfs        38G  847M   37G   3% /  
devtmpfs                        devtmpfs  109M     0  109M   0% /dev  
tmpfs                           tmpfs     118M     0  118M   0% /dev/shm  
tmpfs                           tmpfs     118M  4.5M  114M   4% /run  
tmpfs                           tmpfs     118M     0  118M   0% /sys/fs/cgroup  
/dev/sda2                       xfs      1014M   63M  952M   7% /boot  
tmpfs                           tmpfs      24M     0   24M   0% /run/user/1000  

 Устанавливаем пакет xfsdump ля снятия копии с root тома  
[vagrant@lvm ~]$ sudo -i  
[root@lvm ~]# yum install xfsdump  
Installed:  
  xfsdump.x86_64 0:3.1.7-2.el7_9                                                                                                                                                                                     

Dependency Installed:  
  attr.x86_64 0:2.4.46-13.el7                                                                                                                                                                                       

Complete!  

Подготовим временнýй том для root раздела:   

[root@lvm ~]# pvcreate /dev/sdb  
  Physical volume "/dev/sdb" successfully created.  
[root@lvm ~]# vgcreate vg_root /dev/sdb  
  Volume group "vg_root" successfully created  
[root@lvm ~]# lvcreate -n lv_root -l +100%FREE /dev/vg_root  
  Logical volume "lv_root" created.  
[root@lvm ~]# lsblk  
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT  
sda                       8:0    0   40G  0 disk   
├─sda1                    8:1    0    1M  0 part   
├─sda2                    8:2    0    1G  0 part /boot  
└─sda3                    8:3    0   39G  0 part   
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /  
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]  
sdb                       8:16   0   10G  0 disk   
└─vg_root-lv_root       253:2    0   10G  0 lvm    
sdc                       8:32   0    2G  0 disk   
sdd                       8:48   0    1G  0 disk   
sde                       8:64   0    1G  0 disk  

Создадим на нем файловую систему и смонтируем его, чтобý перенести туда данные:
[root@lvm ~]# mkfs.xfs /dev/vg_root/lv_root  
meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks  
         =                       sectsz=512   attr=2, projid32bit=1  
         =                       crc=1        finobt=0, sparse=0  
data     =                       bsize=4096   blocks=2620416, imaxpct=25  
         =                       sunit=0      swidth=0 blks  
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1  
log      =internal log           bsize=4096   blocks=2560, version=2  
         =                       sectsz=512   sunit=0 blks, lazy-count=1  
realtime =none                   extsz=4096   blocks=0, rtextents=0  

[root@lvm ~]# mount /dev/vg_root/lv_root /mnt  

скопируем все данные с root раздела в /mnt:  
[root@lvm ~]# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt  
xfsdump: Dump Status: SUCCESS  
xfsrestore: restore complete: 15 seconds elapsed  
xfsrestore: Restore Status: SUCCESS  

переконфигурируем grub для того, чтобы при старте перейти в новый root       
[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done  
[root@lvm ~]# chroot /mnt/  
[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg  
Generating grub configuration file ...  
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64  
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img  
done

[root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done  
*** Creating image file ***  
*** Creating image file done ***  
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***  

Ну и для того, чтобы при загрузке был смонтирован нужнý root нужно в файле  
/boot/grub2/grub.cfg заменитþ rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root  
[root@lvm boot]# cd /boot/grub2/  
[root@lvm grub2]# ls   
device.map  fonts  grub.cfg  grubenv  i386-pc  locale 
[root@lvm grub2]# vi grub.cfg  

Перезагрузим сервер с новым root  
[vagrant@lvm ~]$ sudo reboot  
Connection to 127.0.0.1 closed by remote host.  
Connection to 127.0.0.1 closed.  
alex-linux@alexlinux:~/linux-vm/Less-3$ vagrant ssh  
Last login: Sun Mar  5 06:47:11 2023 from 10.0.2.2  
[vagrant@lvm ~]$ lsblk  
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol00 253:2    0 37.5G  0 lvm  
sdb                       8:16   0   10G  0 disk 
└─vg_root-lv_root       253:0    0   10G  0 lvm  /
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk 

Изменим размер старого VG и вернем на него рут  
Удалим старый LV и сделаем его размер 8Гб  
[vagrant@lvm ~]$ sudo -i  
[root@lvm ~]# lvremove /dev/VolGroup00/LogVol00  
Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y  
  Logical volume "LogVol00" successfully removed  
[root@lvm ~]# lsblk  
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT  
sda                       8:0    0   40G  0 disk   
├─sda1                    8:1    0    1M  0 part  
├─sda2                    8:2    0    1G  0 part /boot 
└─sda3                    8:3    0   39G  0 part   
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]  
sdb                       8:16   0   10G  0 disk   
└─vg_root-lv_root       253:0    0   10G  0 lvm  /  
sdc                       8:32   0    2G  0 disk   
sdd                       8:48   0    1G  0 disk   
sde                       8:64   0    1G  0 disk   
[root@lvm ~]# lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00  
WARNING: xfs signature detected on /dev/VolGroup00/LogVol00 at offset 0. Wipe it? [y/n]: y  
  Wiping xfs signature on /dev/VolGroup00/LogVol00.  
  Logical volume "LogVol00" created.  
[root@lvm ~]# lsblk  
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT   
sda                       8:0    0   40G  0 disk   
├─sda1                    8:1    0    1M  0 part   
├─sda2                    8:2    0    1G  0 part /boot  
└─sda3                    8:3    0   39G  0 part   
  ├─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]  
  └─VolGroup00-LogVol00 253:2    0    8G  0 lvm    
sdb                       8:16   0   10G  0 disk   
└─vg_root-lv_root       253:0    0   10G  0 lvm  /  
sdc                       8:32   0    2G  0 disk   
sdd                       8:48   0    1G  0 disk   
sde                       8:64   0    1G  0 disk  
[root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVol00  
meta-data=/dev/VolGroup00/LogVol00 isize=512    agcount=4, agsize=524288 blks  
         =                       sectsz=512   attr=2, projid32bit=1  
         =                       crc=1        finobt=0, sparse=0  
data     =                       bsize=4096   blocks=2097152, imaxpct=25  
         =                       sunit=0      swidth=0 blks  
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1  
log      =internal log           bsize=4096   blocks=2560, version=2  
         =                       sectsz=512   sunit=0 blks, lazy-count=1  
realtime =none                   extsz=4096   blocks=0, rtextents=0  
[root@lvm ~]# mount /dev/VolGroup00/LogVol00 /mnt  
[root@lvm ~]# xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt  
xfsrestore: Restore Status: SUCCESS  
[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done  
[root@lvm ~]# chroot /mnt/  
[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg  
Generating grub configuration file ...   
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64  
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img  
done  
[root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done  
*** Creating image file ***  
*** Creating image file done ***  
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***  


Не перезагружая и не выходя из chroot переносим /var в зеркало  

------ Выделить том под /var -----  

Создаем зеркало  
[root@lvm boot]# pvcreate /dev/sdc /dev/sdd  
  Physical volume "/dev/sdc" successfully created.  
  Physical volume "/dev/sdd" successfully created.  
[root@lvm boot]# vgcreate vg_var /dev/sdc /dev/sdd  
  Volume group "vg_var" successfully created  
[root@lvm boot]# lvcreate -L 950M -m1 -n lv_var vg_var  
  Rounding up size to full physical extent 952.00 MiB  
  Logical volume "lv_var" created.  
[root@lvm boot]# lsblk  
NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT  
sda                        8:0    0   40G  0 disk   
├─sda1                     8:1    0    1M  0 part   
├─sda2                     8:2    0    1G  0 part /boot  
└─sda3                     8:3    0   39G  0 part   
  ├─VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]  
  └─VolGroup00-LogVol00  253:2    0    8G  0 lvm  /  
sdb                        8:16   0   10G  0 disk   
└─vg_root-lv_root        253:0    0   10G  0 lvm    
sdc                        8:32   0    2G  0 disk   
├─vg_var-lv_var_rmeta_0  253:3    0    4M  0 lvm    
│ └─vg_var-lv_var        253:7    0  952M  0 lvm    
└─vg_var-lv_var_rimage_0 253:4    0  952M  0 lvm    
  └─vg_var-lv_var        253:7    0  952M  0 lvm    
sdd                        8:48   0    1G  0 disk   
├─vg_var-lv_var_rmeta_1  253:5    0    4M  0 lvm    
│ └─vg_var-lv_var        253:7    0  952M  0 lvm    
└─vg_var-lv_var_rimage_1 253:6    0  952M  0 lvm    
  └─vg_var-lv_var        253:7    0  952M  0 lvm    
sde                        8:64   0    1G  0 disk   
 Создаем файловую систему  
[root@lvm boot]# mkfs.ext4 /dev/vg_var/lv_var  
mke2fs 1.42.9 (28-Dec-2013)  
Filesystem label=  
OS type: Linux  
Block size=4096 (log=2)  
Fragment size=4096 (log=2)  
Stride=0 blocks, Stripe width=0 blocks  
60928 inodes, 243712 blocks  
12185 blocks (5.00%) reserved for the super user  
First data block=0  
Maximum filesystem blocks=249561088  
8 block groups  
32768 blocks per group, 32768 fragments per group  
7616 inodes per group  
Superblock backups stored on blocks:   
	32768, 98304, 163840, 229376  

Allocating group tables: done                              
Writing inode tables: done                              
Creating journal (4096 blocks): done  
Writing superblocks and filesystem accounting information: done  
 монтируем   
[root@lvm boot]# mount /dev/vg_var/lv_var /mnt
[root@lvm boot]# cp -aR /var/* /mnt/ # rsync -avHPSAX /var/ /mnt/  

сохняем содержимое старого var  
[root@lvm boot]# mkdir /tmp/oldvar && mv /var/* /tmp/oldvar  

отмонтируем временный католог и монтируем действующй var  
[root@lvm boot]# umount /mnt  
[root@lvm boot]# mount /dev/vg_var/lv_var /var  

 Правим fstab длā автоматического монтированиā /var:  

[root@lvm boot]# echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab  

перезагружаем сервер в новый уменьшенный рут  
[root@lvm boot]# exit  
exit  
[root@lvm ~]# sudo reboot  
alex-linux@alexlinux:~/linux-vm/Less-3$ vagrant ssh  
Last login: Sun Mar  5 07:15:49 2023 from 10.0.2.2  
 
 Удаляем временный рут том, который создавали для переноса  
[vagrant@lvm ~]$ sudo -i  
[root@lvm ~]# lvremove /dev/vg_root/lv_root  
Do you really want to remove active logical volume vg_root/lv_root? [y/n]: y  
  Logical volume "lv_root" successfully removed  
[root@lvm ~]# vgremove /dev/vg_root  
  Volume group "vg_root" successfully removed  
[root@lvm ~]# pvremove /dev/sdb  
  Labels on physical volume "/dev/sdb" successfully wiped.  
[root@lvm ~]# lsblk  
NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT  
sda                        8:0    0   40G  0 disk   
├─sda1                     8:1    0    1M  0 part   
├─sda2                     8:2    0    1G  0 part /boot  
└─sda3                     8:3    0   39G  0 part   
  ├─VolGroup00-LogVol00  253:0    0    8G  0 lvm  /  
  └─VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]  
sdb                        8:16   0   10G  0 disk   
sdc                        8:32   0    2G  0 disk   
├─vg_var-lv_var_rmeta_0  253:3    0    4M  0 lvm    
│ └─vg_var-lv_var        253:7    0  952M  0 lvm  /var  
└─vg_var-lv_var_rimage_0 253:4    0  952M  0 lvm    
  └─vg_var-lv_var        253:7    0  952M  0 lvm  /var  
sdd                        8:48   0    1G  0 disk   
├─vg_var-lv_var_rmeta_1  253:5    0    4M  0 lvm    
│ └─vg_var-lv_var        253:7    0  952M  0 lvm  /var  
└─vg_var-lv_var_rimage_1 253:6    0  952M  0 lvm    
  └─vg_var-lv_var        253:7    0  952M  0 lvm  /var  
sde                        8:64   0    1G  0 disk     

--------Выделяем том под /home-------   


[root@lvm ~]# lvcreate -n LogVol_Home -L 2G /dev/VolGroup00  
  Logical volume "LogVol_Home" created.  
[root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVol_Home  
meta-data=/dev/VolGroup00/LogVol_Home isize=512    agcount=4, agsize=131072 blks  
         =                       sectsz=512   attr=2, projid32bit=1  
         =                       crc=1        finobt=0, sparse=0  
data     =                       bsize=4096   blocks=524288, imaxpct=25  
         =                       sunit=0      swidth=0 blks  
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1  
log      =internal log           bsize=4096   blocks=2560, version=2  
         =                       sectsz=512   sunit=0 blks, lazy-count=1  
realtime =none                   extsz=4096   blocks=0, rtextents=0  
[root@lvm ~]# mount /dev/VolGroup00/LogVol_Home /mnt/  
[root@lvm ~]# cp -aR /home/* /mnt/  
[root@lvm ~]# rm -rf /home/*  
[root@lvm ~]# umount /mnt   
[root@lvm ~]# mount /dev/VolGroup00/LogVol_Home /home/  
[root@lvm ~]# echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab  
[root@lvm ~]# lsblk

Сгенерируем файлы в /home/:  
[root@lvm ~]# touch /home/file{1..20}  

 --------- Работа со снапшотами:------------    

Снимаем снапшот  
[root@lvm ~]# lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home  
  Rounding up size to full physical extent 128.00 MiB  
  Logical volume "home_snap" created.  

  Удалим часть файлов  
[root@lvm ~]# rm -f /home/file{11..20}  

Восстановим из снапшота  
[root@lvm ~]# umount /home    
[root@lvm ~]# lvconvert --merge /dev/VolGroup00/home_snap    
  Merging of volume VolGroup00/home_snap started.    
  VolGroup00/LogVol_Home: Merged: 100.00%    
[root@lvm ~]# mount /home  
[root@lvm ~]# lsblk  
NAME                       MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT  
sda                          8:0    0   40G  0 disk   
├─sda1                       8:1    0    1M  0 part   
├─sda2                       8:2    0    1G  0 part /boot  
└─sda3                       8:3    0   39G  0 part   
  ├─VolGroup00-LogVol00    253:0    0    8G  0 lvm  /  
  ├─VolGroup00-LogVol01    253:1    0  1.5G  0 lvm  [SWAP]  
  └─VolGroup00-LogVol_Home 253:2    0    2G  0 lvm  /home  
sdb                          8:16   0   10G  0 disk   
sdc                          8:32   0    2G  0 disk   
├─vg_var-lv_var_rmeta_0    253:3    0    4M  0 lvm    
│ └─vg_var-lv_var          253:7    0  952M  0 lvm  /var  
└─vg_var-lv_var_rimage_0   253:4    0  952M  0 lvm    
  └─vg_var-lv_var          253:7    0  952M  0 lvm  /var  
sdd                          8:48   0    1G  0 disk   
├─vg_var-lv_var_rmeta_1    253:5    0    4M  0 lvm    
│ └─vg_var-lv_var          253:7    0  952M  0 lvm  /var  
└─vg_var-lv_var_rimage_1   253:6    0  952M  0 lvm    
  └─vg_var-lv_var          253:7    0  952M  0 lvm  /var  
sde                          8:64   0    1G  0 disk   

Задание выполнено!  

