# Настройка и использование NFS. Vagrant стенд для NFS.
## Развертывание Vagrant стенда для NFS.

### Серверная часть

Скрипт `nfss_script.sh` используемый для развертывания серверной VM

```
#!/bin/bash
 sudo su
# Доустанавливаем необходимые утилиты NFS и ставим nfs-server в автозагрузку
 yum -y install nfs-utils
 systemctl enable nfs-server
 systemctl start nfs-server
# Создаём и настраиваем директорию для экспорта
 mkdir -p /srv/share/upload
 chown -R nfsnobody:nfsnobody /srv/share
 chmod 0777 /srv/share/upload
# Создаем файл /etc/exports.d/share.exports с необходимыми настройками для экспортирования ранее созданной директории
 echo '/srv/share 192.168.56.11/32(rw,sync,root_squash)' | tee /etc/exports.d/share.exports
# Экспортируем ранее созданную директорию
 exportfs -r
# Перезапускаем nfs-server
 systemctl restart nfs-server
# Ставим firewalld в автозагрузку
systemctl enable firewalld --now
# Производим необходимые разрешения в firewalld
 firewall-cmd --add-service="nfs3" \
 --add-service="rpc-bind" \
 --add-service="mountd" \
 --permanent
# Перечитываем и перезапускаем firewalld
 firewall-cmd --reload
 systemctl restart firewalld
```
### Клиентская часть

Скрипт `nfsс_script.sh` используемый для развертывания клиентской VM

```
#!/bin/bash
 sudo su
# Доустанавливаем необходимые утилиты NFS
 yum install -y nfs-utils
# Ставим firewalld в автозагрузку
 systemctl enable firewalld --now
#  Редактируем /etc/fstab для монтирования 
 echo "192.168.56.10:/srv/share/ /mnt nfs vers=3,proto=udp,noauto,x-systemd.automount 0 0" >> /etc/fstab
# Обновляем конфигурацию юнитов
 systemctl daemon-reload
# Перезапускаем remote-fs.target
 systemctl restart remote-fs.target
```
## Проверяем работоспособность

Заходим на сервер, заходим в каталог `/srv/share/upload`, создаём тестовый файл `touch check_file_otus`
```
[vagrant@nfsserver upload]$ touch check_file_otus
[vagrant@nfsserver upload]$ ll
total 0
-rw-rw-r--. 1 vagrant vagrant 0 Jan 25 13:22 check_file_otus
```

Заходим на клиент, заходим в каталог `/mnt/upload` ,проверяем наличие ранее созданного файла

```
[vagrant@nfsclient ~]$ cd /mnt/upload/
[vagrant@nfsclient upload]$ ll
total 0
-rw-rw-r--. 1 vagrant vagrant 0 Jan 25 13:22 check_file_otus
```
Создаём тестовый файл `touch client_file_otus`, проверяем, что файл успешно создан

```
[vagrant@nfsclient upload]$ touch client_file_otus
[vagrant@nfsclient upload]$ ll
total 0
-rw-rw-r--. 1 vagrant vagrant 0 Jan 25 13:22 check_file_otus
-rw-rw-r--. 1 vagrant vagrant 0 Jan 25 13:23 client_file_otus
```
Перезагружаем клиент

```
[vagrant@nfsclient upload]$ sudo reboot
Connection to 127.0.0.1 closed by remote host.
Connection to 127.0.0.1 closed.
```

Заходим на клиент, заходим в каталог `/mnt/upload`, проверяем наличие ранее созданных файлов

```
Last login: Wed Jan 25 13:21:06 2023 from 10.0.2.2
[vagrant@nfsclient ~]$ cd /mnt/upload/
[vagrant@nfsclient upload]$ ll
total 0
-rw-rw-r--. 1 vagrant vagrant 0 Jan 25 13:22 check_file_otus
-rw-rw-r--. 1 vagrant vagrant 0 Jan 25 13:23 client_file_otus
```
Перезагружаем сервер, заходим на сервер, проверяем наличие файлов в каталоге `/srv/share/upload/`

```
[vagrant@nfsserver upload]$ sudo reboot
Connection to 127.0.0.1 closed by remote host.
Connection to 127.0.0.1 closed.
```
```
Last login: Wed Jan 25 13:21:30 2023 from 10.0.2.2
[vagrant@nfsserver ~]$ cd /srv/share/upload/  
[vagrant@nfsserver upload]$ ll
total 0
-rw-rw-r--. 1 vagrant vagrant 0 Jan 25 13:22 check_file_otus
-rw-rw-r--. 1 vagrant vagrant 0 Jan 25 13:23 client_file_otus

```

Проверяем статус сервера NFS

```
[vagrant@nfsserver upload]$ systemctl status nfs
● nfs-server.service - NFS server and services
   Loaded: loaded (/usr/lib/systemd/system/nfs-server.service; enabled; vendor preset: disabled)
  Drop-In: /run/systemd/generator/nfs-server.service.d
           └─order-with-mounts.conf
   Active: active (exited) since Wed 2023-01-25 14:26:36 UTC; 1min 50s ago
  Process: 828 ExecStartPost=/bin/sh -c if systemctl -q is-active gssproxy; then systemctl reload gssproxy ; fi (code=exited, status=0/SUCCESS)
  Process: 801 ExecStart=/usr/sbin/rpc.nfsd $RPCNFSDARGS (code=exited, status=0/SUCCESS)
  Process: 798 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
 Main PID: 801 (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/nfs-server.service
```
Проверяем статус firewall 

```
[vagrant@nfsserver upload]$ systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2023-01-25 14:26:32 UTC; 2min 31s ago
     Docs: man:firewalld(1)
 Main PID: 402 (firewalld)
   CGroup: /system.slice/firewalld.service
           └─402 /usr/bin/python2 -Es /usr/sbin/firewalld --nofork --nopid
```
Проверяем экспорты 

```
[vagrant@nfsserver upload]$ sudo exportfs -s
/srv/share  192.168.56.11/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
```

Проверяем работу RPC 

```
[vagrant@nfsserver upload]$ showmount -a 192.168.56.10
All mount points on 192.168.56.10:
192.168.56.11:/srv/share
```

Перезагружаем клиент, заходим на клиент

```
[vagrant@nfsclient upload]$ sudo reboot
Connection to 127.0.0.1 closed by remote host.
Connection to 127.0.0.1 closed.
```

Проверяем работу RPC 

```
Last login: Wed Jan 25 14:25:11 2023 from 10.0.2.2
[vagrant@nfsclient ~]$ showmount -a 192.168.56.10
All mount points on 192.168.56.10:
```

Заходим в каталог `/mnt/upload`, проверяем статус монтирования 

```
[vagrant@nfsclient ~]$ cd /mnt/upload/
[vagrant@nfsclient upload]$ mount | grep mnt
systemd-1 on /mnt type autofs (rw,relatime,fd=33,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=11197)
192.168.56.10:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,proto=udp,timeo=11,retrans=3,sec=sys,mountaddr=192.168.56.10,mountvers=3,mountport=20048,mountproto=udp,local_lock=none,addr=192.168.56.10)
```

Проверяем наличие ранее созданных файлов

```
vagrant@nfsclient upload]$ ll
total 0
-rw-rw-r--. 1 vagrant vagrant 0 Jan 25 13:22 check_file_otus
-rw-rw-r--. 1 vagrant vagrant 0 Jan 25 13:23 client_file_otus
```

Создаём тестовый файл `touch final_check_otus` ,проверяем, что файл успешно создан

```
[vagrant@nfsclient upload]$ touch final_check_otus
[vagrant@nfsclient upload]$ ll
total 0
-rw-rw-r--. 1 vagrant vagrant 0 Jan 25 13:22 check_file_otus
-rw-rw-r--. 1 vagrant vagrant 0 Jan 25 13:23 client_file_otus
-rw-rw-r--. 1 vagrant vagrant 0 Jan 25 14:35 final_check_otus
```
```
[vagrant@nfsserver upload]$ ll
total 0
-rw-rw-r--. 1 vagrant vagrant 0 Jan 25 13:22 check_file_otus
-rw-rw-r--. 1 vagrant vagrant 0 Jan 25 13:23 client_file_otus
-rw-rw-r--. 1 vagrant vagrant 0 Jan 25 14:35 final_check_otus
```