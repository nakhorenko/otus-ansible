Первым делом создаются 2 вирт машины. Для этого используем вагрант файл с конфигом.
<details>
  <summary>Конфиг</summary>  
	
```  
# -*- mode: ruby -*- 
# vi: set ft=ruby : 
Vagrant.configure(2) do |config| 
	   config.vm.box = "centos/7" 
	   config.vm.box_version = "2004.01" 
	   config.vm.provider "virtualbox" do |v| 
		  v.memory = 256 
		  v.cpus = 1 
	   end 
	   config.vm.define "nfss" do |nfss| 
	   nfss.vm.network "private_network", ip: "192.168.50.10",  virtualbox__intnet: "net1" 
	  	 nfss.vm.hostname = "nfss" 
	   end 
	   config.vm.define "nfsc" do |nfsc| 
		   nfsc.vm.network "private_network", ip: "192.168.50.11",  virtualbox__intnet: "net1" 
		   nfsc.vm.hostname = "nfsc" 
	   end 
	end 
	
```
</details>

Далее доустановим утилиты nfs

<details>
	<summary>Вывод</summary>
	
```
[root@nfss ~]# yum install nfs-utils
Loaded plugins: fastestmirror
Determining fastest mirrors
 * base: mirror.besthosting.ua
 * extras: mirror.besthosting.ua
 * updates: mirror.besthosting.ua
                                                                                                                                            
Updated:
  nfs-utils.x86_64 1:1.3.0-0.68.el7.2                                                                                                                                                                       

Complete!
[root@nfss ~]# 
```
</details>

Включаем firewall

```
[root@nfss ~]# systemctl status firewalld.service 
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
```
```
[root@nfss ~]# systemctl enable firewalld.service 
Created symlink from /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service to /usr/lib/systemd/system/firewalld.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/firewalld.service to /usr/lib/systemd/system/firewalld.service.
```
```
[root@nfss ~]# systemctl enable firewalld --now   
[root@nfss ~]# systemctl status firewalld.service 
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2023-03-02 13:01:29 UTC; 1s ago
```

Разрешаем в firewall доступ к сервисам NFS

```
[root@nfss ~]# firewall-cmd --add-service="nfs3"
success
[root@nfss ~]# firewall-cmd --add-service="rpc-bind"
success
[root@nfss ~]# firewall-cmd --add-service="mountd"  
success

[root@nfss ~]# firewall-cmd  --permanent firewall-cmd --reload
usage: see firewall-cmd man page
firewall-cmd: error: unrecognized arguments: firewall-cmd
[root@nfss ~]# firewall-cmd   --reload
success
```
И включаем NFS севрер
```
[root@nfss ~]# systemctl enable nfs --now 
Created symlink from /etc/systemd/system/multi-user.target.wants/nfs-server.service to /usr/lib/systemd/system/nfs-server.service.
```
Создаём share директорию, меняем владельца и права на неё
```
[root@nfss ~]# mkdir -p /srv/share/upload
[root@nfss ~]# chown -R nfsnobody:nfsnobody /srv/share
[root@nfss ~]# chmod 0777 /srv/share/upload 
[root@nfss ~]# ls -la /srv/share
total 0
drwxr-xr-x. 3 nfsnobody nfsnobody 20 Mar  2 13:10 .
drwxr-xr-x. 3 root      root      19 Mar  2 13:10 ..
drwxrwxrwx. 2 nfsnobody nfsnobody  6 Mar  2 13:10 upload
```
Далее создаём запись в /etc/exports, чтобы экспортировать директорию /srv/share, экспортируем её и проверяем всё ли получилось
```
[root@nfss exports.d]# cat << EOF > /etc/exports
> /srv/share 192.168.50.11/32(rw,sync,root_squash) 
> EOF
[root@nfss exports.d]# cat /etc/exports
/srv/share 192.168.50.11/32(rw,sync,root_squash) 
[root@nfss exports.d]# exportfs -r 
[root@nfss exports.d]# exportfs -s 
/srv/share  192.168.50.11/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
```
Помимо этого, чтобы не столкнуться с ошибками и невозможностью примонтировать на клиенте директорию /srv/share, меняем настройки firewall
```
[root@nfss exports.d]# firewall-cmd --permanent --add-port=111/tcp
success
[root@nfss exports.d]# firewall-cmd --permanent --add-port=20048/tcp
success
[root@nfss exports.d]# firewall-cmd --permanent --zone=public --add-service=nfs
success
[root@nfss exports.d]# firewall-cmd --permanent --zone=public --add-service=mountd
success
[root@nfss exports.d]# firewall-cmd --permanent --zone=public --add-service=rpc-bind
success
[root@nfss exports.d]# firewall-cmd --permanent --add-port=2049/udp
success
[root@nfss exports.d]#  firewall-cmd --reload        
success
```

Идём на сервер-клиент и доустанавливаем утилиты nfs
<details>
	<summary>Вывод</summary>
	
```
[root@nfsc ~]# yum install nfs-utils -y
Running transaction
  Updating   : 1:nfs-utils-1.3.0-0.68.el7.2.x86_64                                                                                                                                          1/2 
  Cleanup    : 1:nfs-utils-1.3.0-0.66.el7.x86_64                                                                                                                                            2/2 
  Verifying  : 1:nfs-utils-1.3.0-0.68.el7.2.x86_64                                                                                                                                          1/2 
  Verifying  : 1:nfs-utils-1.3.0-0.66.el7.x86_64                                                                                                                                            2/2 
Updated:
  nfs-utils.x86_64 1:1.3.0-0.68.el7.2 
Complete!
```
</details>

Включаем firewall
```
[root@nfsc ~]# systemctl enable firewalld --now
[root@nfsc ~]# systemctl status firewalld.service 
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2023-03-02 13:28:26 UTC; 21s ago
```
Вносим в /etc/fstab запись
```
[root@nfsc ~]# cat << EOF >> /etc/fstab 
192.168.50.10:/srv/share/ /mnt nfs vers=3,proto=udp,noauto,x-systemd.automount 0 0
EOF
```
И релоад демонов
```
[root@nfsc ~]# systemctl daemon-reload 
[root@nfsc ~]# systemctl restart remote-fs.target
[root@nfsc ~]# systemctl status remote-fs.target
● remote-fs.target - Remote File Systems
   Loaded: loaded (/usr/lib/systemd/system/remote-fs.target; enabled; vendor preset: enabled)
   Active: active since Thu 2023-03-02 13:35:36 UTC; 4s ago
     Docs: man:systemd.special(7)

Mar 02 13:35:36 nfsc systemd[1]: Reached target Remote File Systems.
```
```
[root@nfsc mnt]# mount | grep mnt
systemd-1 on /mnt type autofs (rw,relatime,fd=46,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=43333)
192.168.50.10:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,proto=udp,timeo=11,retrans=3,sec=sys,mountaddr=192.168.50.10,mountvers=3,mountport=20048,mountproto=udp,local_lock=none,addr=192.168.50.10)
```
Чекаем как работает передача файлов между сервером и клиентом
server:
```
[root@nfss upload]# touch server_check_file1
[root@nfss upload]# ll
total 0
-rw-r--r--. 1 root root 0 Mar  2 13:51 server_check_file1
```
client:
```
[root@nfsc upload]# ll
total 0
-rw-r--r--. 1 root root 0 Mar  2 13:51 server_check_file1
```
client:
```
[root@nfsc upload]# touch client_check_file1
[root@nfsc upload]# ll
total 0
-rw-r--r--. 1 nfsnobody nfsnobody 0 Mar  2 13:52 client_check_file1
-rw-r--r--. 1 root      root      0 Mar  2 13:51 server_check_file1
```
server:
```
[root@nfsc upload]# touch client_check_file1
[root@nfsc upload]# ll
total 0
-rw-r--r--. 1 nfsnobody nfsnobody 0 Mar  2 13:52 client_check_file1
-rw-r--r--. 1 root      root      0 Mar  2 13:51 server_check_file1
```
Всё работает и файлы отражаются на обоих сторонах

Проверить, что ничего не слетело и продолжает работать

client
```
[root@nfsc upload]# reboot 
Connection to 127.0.0.1 closed by remote host.

[root@nfsc ~]# mount | grep mnt
systemd-1 on /mnt type autofs (rw,relatime,fd=24,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=10866)
192.168.50.10:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,proto=udp,timeo=11,retrans=3,sec=sys,mountaddr=192.168.50.10,mountvers=3,mountport=20048,mountproto=udp,local_lock=none,addr=192.168.50.10)

[root@nfsc ~]# ls -la /mnt/upload/
total 0
drwxrwxrwx. 2 nfsnobody nfsnobody 58 Mar  2 13:52 .
drwxr-xr-x. 3 nfsnobody nfsnobody 20 Mar  2 13:10 ..
-rw-r--r--. 1 nfsnobody nfsnobody  0 Mar  2 13:52 client_check_file1
-rw-r--r--. 1 root      root       0 Mar  2 13:51 server_check_file1
```
server
```
[root@nfss ~]# exportfs -s                   
/srv/share  192.168.50.11/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)

[root@nfss ~]# ls -la /srv/share/upload/
total 0
drwxrwxrwx. 2 nfsnobody nfsnobody 58 Mar  2 13:52 .
drwxr-xr-x. 3 nfsnobody nfsnobody 20 Mar  2 13:10 ..
-rw-r--r--. 1 nfsnobody nfsnobody  0 Mar  2 13:52 client_check_file1
-rw-r--r--. 1 root      root       0 Mar  2 13:51 server_check_file1

[root@nfss ~]# cd /srv/share/
[root@nfss share]# ll
total 0
drwxrwxrwx. 2 nfsnobody nfsnobody 58 Mar  2 13:52 upload
```
