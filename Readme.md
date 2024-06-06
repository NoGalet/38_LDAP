Vagrant-стенд c LDAP на базе FreeIPA

Цель домашнего задания
Научиться настраивать LDAP-сервер и подключать к нему LDAP-клиентов

Описание домашнего задания
1) Установить FreeIPA
2) Написать Ansible-playbook для конфигурации клиента

Дополнительное задание
3)* Настроить аутентификацию по SSH-ключам
4)** Firewall должен быть включен на сервере и на клиенте

Выполнение:
1. Подключаемся к FreeIPA-серверу:
stas@myserv:~/LAB/38_LDAP$ vagrant ssh ipa.otus.lan
2. Перейдём в root-пользователя:
[vagrant@ipa ~]$ sudo -i
3. Установим часовой пояс: 
[root@ipa ~]# timedatectl set-timezone Europe/Moscow 
4. Установим утилиту chrony:
cd /etc/yum.repos.d/
sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
yum update -y
[root@ipa yum.repos.d]# yum install -y chrony
Failed to set locale, defaulting to C.UTF-8
Last metadata expiration check: 0:06:25 ago on Thu Jun  6 14:39:59 2024.
Package chrony-4.1-1.el8.x86_64 is already installed.
Dependencies resolved.
Nothing to do.
Complete!
5. Запустим chrony и добавим его в автозагрузку: 
[root@ipa /]# systemctl enable chronyd --now
6. Выключим Firewall: 
[root@ipa /]# systemctl stop firewalld
7. Отключаем автозапуск Firewalld: 
[root@ipa /]# systemctl disable firewalld
8. Остановим Selinux:
[root@ipa /]# setenforce 0
9. Поменяем в файле /etc/selinux/config, параметр Selinux на disabled
[root@ipa /]# nano /etc/selinux/config
10. добавим запись 192.168.57.10 ipa.otus.lan ipa в файл /etc/hosts:
[root@ipa /]# nano /etc/hosts
11. Установим модуль DL1: 
[root@ipa /]# yum install -y @idm:DL1
Complete!
12. Установим FreeIPA-сервер:
[root@ipa /]# yum install -y ipa-server
Complete!
13. Запустим скрипт установки: 
[root@ipa /]# ipa-server-install
The log file for this installation can be found in /var/log/ipaserver-install.log
Do you want to configure integrated DNS (BIND)? [no]: no
Server host name [ipa.otus.lan]: <Enter>
Please confirm the domain name [otus.lan]: <Enter>
Please provide a realm name [OTUS.LAN]: <Enter>
Directory Manager password: <8 символов>
Password (confirm): <Дублируем указанный пароль>
IPA admin password: <8 символов>
Password (confirm): <Дублируем указанный пароль>
NetBIOS domain name [OTUS]: <Enter>
Do you want to configure integrated DNS (BIND)? [no]: no
The IPA Master Server will be configured with:
Hostname:       ipa.otus.lan
IP address(es): 192.168.57.10
Domain name:    otus.lan
Realm name:     OTUS.LAN

The CA will be configured with:
Subject DN:   CN=Certificate Authority,O=OTUS.LAN
Subject base: O=OTUS.LAN
Chaining:     self-signed

Continue to configure the system with these values? [no]:yes

The ipa-server-install command was successful
14. Проверим, что сервер Kerberos может выдать нам билет:
[root@ipa /]# kinit admin
Password for admin@OTUS.LAN: 
[root@ipa /]# klist
Ticket cache: KCM:0
Default principal: admin@OTUS.LAN

Valid starting     Expires            Service principal
06/06/24 15:40:00  06/07/24 14:47:08  krbtgt/OTUS.LAN@OTUS.LAN
15. На нашей хостовой машине пропишем строку 192.168.57.10 ipa.otus.lan в файле Hosts:
stas@myserv:~/LAB/SALT/min$ sudo nano /etc/hosts
16. ткроем c нашей хост-машины веб-страницу:
https://ipa.otus.lan/ipa/ui/
17. В имени пользователя укажем admin, в пароле укажем наш IPA admin password и нажмём войти

- Ansible playbook
18. В каталоге создадим каталог Ansible:
stas@myserv:~/LAB/38_LDAP$ mkdir ansible
19. В каталоге ansible создадим файл hosts со следующими параметрами:
[clients]
client1.otus.lan ansible_host=192.168.57.11 ansible_user=vagrant ansible_ssh_private_key_file=./.vagrant/machines/client1.otus.lan/virtualbox/private_key
client2.otus.lan ansible_host=192.168.57.12 ansible_user=vagrant ansible_ssh_private_key_file=./.vagrant/machines/client2.otus.lan/virtualbox/private_key
stas@myserv:~/LAB/38_LDAP$ nano ./ansible/hosts
20. Далее создадим файл provision.yml:
21. Создадим пользователя otus-user
[root@ipa ~]# ipa user-add otus2-user --first=Otus --last=User --password
Password: 
Enter Password again to verify: 
-----------------------
Added user "otus2-user"
-----------------------
  User login: otus2-user
  First name: Otus
  Last name: User
  Full name: Otus User
  Display name: Otus User
  Initials: OU
  Home directory: /home/otus2-user
  GECOS: Otus User
  Login shell: /bin/sh
  Principal name: otus2-user@OTUS.LAN
  Principal alias: otus2-user@OTUS.LAN
  User password expiration: 20240606150435Z
  Email address: otus2-user@otus.lan
  UID: 846400005
  GID: 846400005
  Password: True
  Member of groups: ipausers
  Kerberos keys available: True
22. На хосте client1 выполним команду kinit otus2-user:
tas@myserv:~/LAB/38_LDAP$ vagrant ssh client1.otus.lan
Last login: Thu Jun  6 17:15:12 2024 from 10.0.2.2
[vagrant@client1 ~]$ kinit otus2-user
Password for otus2-user@OTUS.LAN: 
Password expired.  You must change it now.
Enter new password: 
Enter it again: 
[vagrant@client1 ~]$ klist
Ticket cache: KCM:1000
Default principal: otus2-user@OTUS.LAN

Valid starting     Expires            Service principal
06/06/24 18:07:19  06/07/24 17:12:57  krbtgt/OTUS.LAN@OTUS.LAN
