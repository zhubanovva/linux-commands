## Создание большого диска из нескольких маленьких дисков

#### Создание логического тома(lv) из физических дисков:

1. Размонтировать диски, которые войдут в lv-том
(выполнить только если диски до этого были mounted)
 ```
[root@localhost ~]# umount /data/01
```

2. Пересоздаем партицию
```
[root@localhost ~]# parted /dev/sds
  GNU Parted 3.1
  Using /dev/sds
  Welcome to GNU Parted! Type 'help' to view a list of commands.(parted) print
  Model: DELL PERC H730P Mini (scsi)
  Disk /dev/sds: 2000GB
  Sector size (logical/physical): 512B/4096B
  Partition Table: gpt
  Disk Flags:Number Start End Size File system Name Flags
  1 1049kB 2000GB 2000GB dt7T
  (parted)rm 1 <<== remove old partition table with all partitions
  (parted)mklabel gpt <<== create a new disklabel (partition table)
  (parted)mkpart primary xfs 1 2000GB <<== make new partition (file.system=XFS, size=sine 1MB till 2TB)
  (parted)quit <<== exit programm
 ```
 
 3. Помечаем новый диск для использования в lv-томе
 ```
 [root@localhost ~]# pvcreate /dev/sds1
 ```

4. Создаем по п.1-3 несколько дисков, общей емкостью = 7.3Т
(в данном примере это /dev/sds /dev/sdr /dev/sdq /dev/sdp по 1.8Т Х 4 =~ 7.3T)

5. Объединяем "нарезанные" диски(/dev/sds /dev/sdr /dev/sdq /dev/sdp) в логическую группу(vg) 
```
[root@localhost ~]# vgcreate vg_dt4 /dev/sds1 /dev/sdq1 /dev/sdp1 /dev/sdr1
```
где vg_dt4 = имя группы, по семантике {/dev/sds1 /dev/sdq1 /dev/sdp1 /dev/sdr1} = диски общей емкостью = 7.3Т)

6. Создаем логичкеский lv-том, что и будет для нас диском. Выделяем ему емкость из vg-группы
```
[root@localhost ~]# lvcreate -l 100%FREE -n lv_dt4 vg_dt4
``` 
где -l 100%FREE = выделить весь объем диску, -n lv_dt4= имя диска lv-том, vg_dt4 = имя vg-группы)

7. Создаем файловую систему для lv-диска 
```
[root@localhost ~]# mkfs.xfs /dev/vg_dt4/lv_dt4
```

8. Монтируем диск как метку /data/XX
```
[root@localhost ~]# mount /dev/vg_dt4/lv_dt4 /data/01
```

9. Для автоматического подключения диска при перегрузке сервера можно добавить запись /etc/fstab
```
[root@localhost ~]# vi /etc/fstab
 >> добавить значение:
 /dev/vg_dt4/lv_dt4 /data/01 xfs defaults 0 0
 ```
 
 #### Добавление еще одного диска в существующую группу
```
$ parted /dev/sdb
    (parted) rm 1
    (parted) mklabel gpt
    (parted) mkpart primary xfs 1 1200GB
    (parted) quit
$ umount /data/01
$ pvcreate /dev/sdb1
$ vgextend vg_dt1 /dev/sdb1
$ lvextend /dev/vg_dt1/lv_dt1 -L+1.09T -r
$ mount /dev/mapper/vg_dt1-lv_dt1 /data/01
```

#### Полезные команды
``` 
$ pvs
$ pvscan
$ pvdisplay
$ vgs
$ vgdisplay vg_dt1
$ pvs
$ lvs
$ df -hT
$ lsblk
```
