# TP 6 - Gestion des disques / Tâches d’administration (1)

## Exercice 1. Disques et partitions

**1. Dans l’interface de configuration de votre VM, créez un second disque dur, de 5 Go dynamiquement alloués ; puis démarrez la VM**

**2. Vérifiez que ce nouveau disque dur est bien détecté par le système**

```
serveur@serveur:~$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
loop0    7:0    0 54,7M  1 loop /snap/lxd/12181
loop1    7:1    0 89,1M  1 loop /snap/core/7917
loop2    7:2    0 54,6M  1 loop /snap/lxd/11985
loop3    7:3    0   89M  1 loop /snap/core/7713
sda      8:0    0   10G  0 disk
├─sda1   8:1    0    1M  0 part
└─sda2   8:2    0   10G  0 part /
sdb      8:16   0    5G  0 disk
sr0     11:0    1 1024M  0 rom
serveur@serveur:~$
```

Le disque est bien détecté sous le nom sdb.

**3. Partitionnez ce disque en utilisant fdisk : créez une première partition de 2 Go de type Linux (n°83), et une seconde partition de 3 Go en NTFS (n°7)**

```
Device Boot   Start      End Sectors Size Id Type
sdb1           2048  4196351 4194304   2G 83 Linux
sdb2        4196352 10485759 6289408   3G  7 HPFS/NTFS/exFAT
```

**4. A ce stade, les partitions ont été créées, mais elles n’ont pas été formatées avec leur système de fichiers. A l’aide de la commande mkfs, formatez vos deux partitions ( pensez à consulter le manuel !)**

```
serveur@serveur:/dev$ sudo mkfs.ext4 sdb1
mke2fs 1.44.6 (5-Mar-2019)
Creating filesystem with 524288 4k blocks and 131072 inodes
Filesystem UUID: 297479ef-95a0-46fe-954d-480e82e72ef5
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

serveur@serveur:/dev$ sudo mkfs.ntfs sdb2
Cluster size has been automatically set to 4096 bytes.
Initializing device with zeroes: 100% - Done.
Creating NTFS volume structures.
mkntfs completed successfully. Have a nice day.
```

**5. Pourquoi la commande df -T, qui affiche le type de système de fichier des partitions, ne fonctionne-telle pas sur notre disque ?**

Car le disque n'est pas connecté au système principale. Celui-ci n'est pas monté sur l'OS.

**6. Faites en sorte que les deux partitions créées soient montées automatiquement au démarrage de la machine, respectivement dans les points de montage /data et /win (vous pourrez vous passer des UUID en raison de l’impossibilité d’effectuer des copier-coller)**

```
UUID=146057e5-0fa9-47f8-94f0-eff90b97421f / ext4 defaults 0 0
/swap.img       none    swap    sw      0       0
UUID=297479ef-95a0-46fe-954d-480e82e72ef5 /data ext4 defaults 0 0
UUID=77957BE920BF54E1 /win ntfs defaults 0 0
```

**7. Utilisez la commande mount puis redémarrez votre VM pour valider la configuration**

```
serveur@serveur:/$ sudo mount /dev/sdb1
serveur@serveur:/$ sudo mount /dev/sdb2
```

*reboot*

```
serveur@serveur:~$ df -aTh
Filesystem     Type        Size  Used Avail Use% Mounted on
/dev/sdb1      ext4        2,0G  6,0M  1,8G   1% /data
/dev/sdb2      fuseblk     3,0G   16M  3,0G   1% /win
serveur@serveur:~$           
```
PS: une bonne partie des partitions à été supprimé pour éviter un copier/coller trop gros

**8. Montez votre clé USB dans la VM**

**9. Créez un dossier partagé entre votre VM et votre système hôte (rem. il peut être nécessaire d’installer les Additions invité de VirtualBox**

## Exercice 2. Partitionnement LVM

Dans cet exercice, nous allons aborder le partitionnement LVM, beaucoup plus flexible pour manipuler les disques et les partitions.

**1. On va réutiliser le disque de 5 Gio de l’exercice précédent. Commencez par démonter les systèmes de fichiers montés dans /data et /win s’ils sont encore montés, et supprimez les lignes correspondantes du fichier /etc/fstab**

```
serveur@serveur:/dev$ sudo umount sdb1
serveur@serveur:/dev$ sudo umount sdb2
```

**2. Supprimez les deux partitions du disque, et créez une patition unique de type LVM**

 La création d’une partition LVM n’est pas indispensable, mais vivement recommandée quand
on utilise LVM sur un disque entier. En effet, elle permet d’indiquer à d’autres OS ou logiciels de
gestion de disques (qui ne reconnaissent pas forcément le format LVM) qu’il y a des données sur
ce disque.

 Attention à ne pas supprimer la partition système !

```
sdb1         2048 10485759 10483712   5G 8e Linux LVM
```

**3. A l’aide de la commande pvcreate, créez un volume physique LVM. Validez qu’il est bien créé, en utilisant la commande pvdisplay.**

 Toutes les commandes concernant les volumes physiques commencent par pv. Celles concernant
les groupes de volumes commencent par vg, et celles concernant les volumes logiques par lv.

```
  Unable to obtain global lock.
serveur@serveur:/dev$ sudo pvdisplay
  "/dev/sdb1" is a new physical volume of "<5,00 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/sdb1
  VG Name
  PV Size               <5,00 GiB
  Allocatable           NO
  PE Size               0
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               72W8ce-UrBn-NLeW-YBS6-eKzn-u2WB-j5VzwB
```
  
**4. A l’aide de la commande vgcreate, créez un groupe de volumes, qui pour l’instant ne contiendra que le volume physique créé à l’étape précédente. Vérifiez à l’aide de la commande vgdisplay.**

 Par convention, on nomme généralement les groupes de volumes vgxx (où xx représente l’indice du groupe de volume, en commençant par 00, puis 01...)

```
serveur@serveur:/dev$ sudo vgdisplay
  --- Volume group ---
  VG Name               VG00
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  1
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                0
  Open LV               0
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <5,00 GiB
  PE Size               4,00 MiB
  Total PE              1279
  Alloc PE / Size       0 / 0
  Free  PE / Size       1279 / <5,00 GiB
  VG UUID               RjsCjb-Qsnu-wTOj-LZgQ-2ALx-cOG2-PYeAVO
```

**5. Créez un volume logique appelé lvData occupant l’intégralité de l’espace disque disponible.**

 On peut renseigner la taille d’un volume logique soit de manière absolue avec l’option -L (par exemple -L 10G pour créer un volume de 10 Gio), soit de manière relative avec l’option -l : -l 60%VG pour utiliser 60% de l’espace total du groupe de volumes, ou encore -l 100%FREE pour utiliser la totalité de l’espace libre.

```
serveur@serveur:/dev$ sudo lvcreate -l 100%FREE -n lvData VG00
  Logical volume "lvData" created.
```

**6. Dans ce volume logique, créez une partition que vous formaterez en ext4, puis procédez comme dans l’exercice 1 pour qu’elle soit montée automatiquement, au démarrage de la machine, dans /data.**

 A ce stade, l’utilité de LVM peut paraître limitée. Il trouve tout son intérêt quand on veut par exemple agrandir une partition à l’aide d’un nouveau disque.

```
Device                        Boot Start      End  Sectors Size Id Type
/dev/mapper/VG00-lvData-part1       2048 10477567 10475520   5G 83 Linux
```

*reboot*

```
UUID=146057e5-0fa9-47f8-94f0-eff90b97421f / ext4 defaults 0 0
/swap.img       none    swap    sw      0       0
UID=72W8ce-UrBn-NLeW-YBS6-eKzn-u2WB-j5VzwB /data ext4 defaults 0 0
```

**7. Eteignez la VM pour ajouter un second disque (peu importe la taille pour cet exercice). Redémarrez la VM, vérifiez que le disque est bien présent. Puis, répétez les questions 2 et 3 sur ce nouveau disque.**

À partir de là, ma VM est morte ;(

J'ai essayé d'ajouter un second disque et elle n'as plus voulu démarrer correctement (même après avoir supprimer le disque). Le meilleur que je peu avoir c'est l'"emergency mode" qui ne me permet pas de faire grand chose... RIP la VM.

**8. Utilisez la commande vgextend <nom_vg> <nom_pv> pour ajouter le nouveau disque au groupe de volumes**

**9. Utilisez la commande lvresize (ou lvextend) pour agrandir le volume logique. Enfin, il ne faut pas oublier de redimensionner le système de fichiers à l’aide de la commande resize2fs.**

 Il est possible d’aller beaucoup plus loin avec LVM, par exemple en créant des volumes par
bandes (l’équivalent du RAID 0) ou du mirroring (RAID 1). Le but de cet exercice n’était que de
présenter les fonctionnalités de base.

## Exercice 3. Exécution de commandes en différé : at et cron

**1. Programmez une tâche qui affiche un rappel pour la réunion qui aura lieu dans 3 minutes. Vérifiez
entre temps que la tâche est bien programmée.**

**2. Est-ce que le message s’est affiché ? Si la réponse est non, essayez de trouver la cause du problème (par
exemple en vous aidant des logs, du manuel...)**

**3. Pour tester le fonctionnement de cron, commencez par programmer l’exécution d’une tâche simple,
l’affichage de “Il faut réviser pour l’examen !”, toutes les 3 minutes.**

**4. Programmez l’exécution d’une commande tous les jours, toute l’année, tous les quarts d’heure**

**5. Programmez l’exécution d’une commande toutes les cinq minutes à partir de 2 (2, 7, 12, etc.) à 18
heures les 1er et 15 du mois :**

**6. Programmez l’exécution d’une commande du lundi au vendredi à 17 heures**

**7. Modifiez votre crontab pour que les messages ne soient plus envoyés par mail, mais redirigés dans un
fichier de log situé dans votre dossier personnel**

**8. Videz votre crontab**
