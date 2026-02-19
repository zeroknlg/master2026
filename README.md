# master2026
b modul on master 2026

1)Настройте службу сетевого времени на базе сервиса chrony

2)Настройте веб-сервер nginx как обратный прокси-сервер

3)Настройте web-based аутентификацию

-------Chrony-----------

control chrony server
после команды выше посмотреть слушается ли 123 порт(Порт синхронизирования времени)

sed -i 's/pool pool.ntp.org iburst/pool pool.ntp.org iburst prefer minstratum 4/' /etc/chrony.conf | grep pool /etc/chrony.conf

sed -i 's/\#local stratum 10/local stratum 5/' /etc/chrony.conf | grep "local stratum" /etc/chrony.conf

systemctl restart chronyd

-------NGINX-----------

apt-get update && apt-get install nginx -y

cat << "EOF" > /etc/nginx/sites-available.d/r-proxy.conf

server {

listen 80;

server_name web.au-team.irpo;

location / {

proxy_pass http://172.16.1.10:8080;

proxy_set_header Host $host;

proxy_set_header X-Real-IP $remote_addr;

proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

proxy_set_header X-Forwarded-Proto $scheme;

auth_basic "Restricted Access";

auth_basic_user_file /etc/nginx/.htpasswd;

}

}

server {

listen 80;

server_name docker.au-team.irpo;

location / {

proxy_pass http://172.16.2.10:8080;

proxy_set_header Host $host;

proxy_set_header X-Real-IP $remote_addr;

proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

proxy_set_header X-Forwarded-Proto $scheme;

}

}

EOF

ln -s /etc/nginx/sites-available.d/r-proxy.conf /etc/nginx/sites-enabled.d/

nginx -t

systemctl enable --now nginx

systemctl status nginx

-------HTTP Basic Auth-----------

apt-get install apache2-htpasswd -y

htpasswd -c /etc/nginx/.htpasswd WEB

cat /etc/nginx/.htpasswd

добавить в файл, в сайт web.au-team.irpo после блока proxy строки

auth_basic "Restricted Access";

auth_basic_user_file /etc/nginx/.htpasswd;

nginx -t

systemctl restart nginx
1)Сконфигурируйте статическую трансляцию портов - 8080 и 2026

==========HQ-RTR====

nft add chain nat prerouting { type nat hook prerouting priority dstnat \; }

nft add rule nat prerouting iif "enp7s1" tcp dport 2026 dnat to 192.168.1.10

nft add rule nat prerouting iif "enp7s1" tcp dport 8080 dnat to 192.168.1.10:80

nft list ruleset

nft list ruleset > /etc/nftables/nftables.nft

systemctl restart nftables

nft list ruleset

==========BR-RTR====

nft add chain nat prerouting { type nat hook prerouting priority dstnat \; }

nft add rule nat prerouting iif "enp7s1" tcp dport { 8080, 2026 } dnat to 192.168.3.10

nft list ruleset

nft list ruleset > /etc/nftables/nftables.nft

systemctl restart nftables

nft list ruleset

sed -i 's/pool pool.ntp.org iburst/server 172.16.1.1 iburst/' /etc/chrony.conf && systemctl restart chronyd && chronyc sources
chronyc sources - вводится эта команда при проверке синхронизации и работы нашего сервиса
BR-SRV

1)Настройте контроллер домена Samba DC на сервере BR-SRV

2)Сконфигурируйте ansible на сервере BR-SRV

3)Разверните веб приложение в docker на сервере BR-SRV

==========Domain==================

---------SambaDC------------

echo "nameserver 192.168.1.10" >> /etc/net/ifaces/enp7s1/resolv.conf; systemctl restart network; cat /etc/resolv.conf

ping ya.ru -c 2

apt-get update && apt-get install -y task-samba-dc

rm -f /etc/samba/smb.conf

rm -rf {/var/lib/samba, /var/cache/samba}

mkdir -p /var/lib/samba/sysvol

samba-tool domain provision

mv /etc/krb5.conf /etc/krb5.conf.back

cp /var/lib/samba/private/krb5.conf /etc/krb5.conf

systemctl enable --now samba

systemctl status samba

samba-tool domain info 127.0.0.1

---------DNS------------

samba-tool dns add br-srv.au-team.irpo au-team.irpo hq-srv A 192.168.1.10 -U Administrator

samba-tool dns add br-srv.au-team.irpo au-team.irpo hq-rtr A 192.168.1.1 -U Administrator

samba-tool dns add br-srv.au-team.irpo au-team.irpo br-rtr A 192.168.3.1 -U Administrator

samba-tool dns add br-srv.au-team.irpo au-team.irpo web.au-team.irpo A 172.16.1.1 -U Administrator

samba-tool dns add br-srv.au-team.irpo au-team.irpo docker.au-team.irpo A 172.16.2.1 -U Administrator

samba-tool dns query br-srv.au-team.irpo au-team.irpo @ ALL -U administrator

sed -i 's/nameserver 192.168.1.10/nameserver 127.0.0.1/' /etc/net/ifaces/enp7s1/resolv.conf; systemctl restart network; cat /etc/resolv.conf

---------Users----------

samba-tool group add hq

for i in {1..5}; do samba-tool user add hquser$i P@ssw0rd; done

# for i in {1..5}; do samba-tool user setexpiry hquser1$i --noexpiry - может понадобится (но это не точно)

for i in {1..5}; do samba-tool group addmembers hq hquser$i; done

samba-tool group listmembers hq

===========HQ-CLI

вводим HQ-CLI в домен

на HQ-RTR меняем параметры DNS на dnsmasq

sed -i 's/192.168.1.10/192.168.3.10/' /etc/dnsmasq.conf; systemctl restart dnsmasq

cat /etc/dnsmasq.conf

на HQ-CLI перезагружаем сеть, проверяем DNS

вводим в домен

control libnss-role

roleadd hq wheel

echo "WHEEL_USERS ALL=(ALL:ALL) /usr/bin/cat, /usr/bin/grep, /usr/bin/id" >> /etc/sudoers

tail /etc/sudoers

заходим доменным пользователем, выполняем sudo id

==========Ansible==================

apt-get install ansible sshpass -y

vim /etc/ansible/ansible.cfg

[defaults]

host_key_checking = False

interpreter_python=auto_silent

cat << EOF >/etc/ansible/hosts

HQ-SRV ansible_user=user ansible_password=resu ansible_port=2026

HQ-RTR ansible_user=net_admin ansible_password=P@ssw0rd

BR-RTR ansible_user=net_admin ansible_password=P@ssw0rd

HQ-CLI ansible_user=user ansible_password=resu

EOF

проверить состояние службы SSH на хостах

ansible all -m ping

==========Docker==================

apt-get install docker-engine docker-compose-v2 -y

systemctl enable --now docker.service

mount -o loop /dev/sr0 /mnt/ -v

ls -l /mnt/docker/

cat /mnt/docker/readme.txt

docker load < /mnt/docker/site_latest.tar

docker load < /mnt/docker/mariadb_latest.tar

docker image ls

<<<<docker-compose-file>>>>

docker compose config

docker compose up -d

docker ps

ss -ltnp4 | grep 8080

переходим на HQ-CLI, заходим по docker.au-team.irpo и по 192.168.3.10:8080

создаем запись

docker rm -f $(docker ps -qa)

снова запускаем docker compose

---NTP

sed -i 's/pool pool.ntp.org iburst/server 172.16.1.1 iburst/' /etc/chrony.conf && systemctl restart chronyd && chronyc sources



1) Файловое хранилище

2) Сервер сетевой файловой системы (nfs)

3) Веб приложение

sed -i 's/pool pool.ntp.org iburst/server 172.16.1.1 iburst/' /etc/chrony.conf && systemctl restart chronyd && chronyc sources

==========RAID==================

lsblk

parted /dev/sdb

mklabel msdos

mkpart primary 1MiB 100%

set 1 raid on

print

select /dev/sdc

mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb1 /dev/sdc1

mdadm --detail --scan >> /etc/mdadm.conf

mkfs.ext4 /dev/md0

mkdir /raid

cp /etc/fstab /etc/fstab.back

echo "/dev/md0 /raid ext4 defaults 0 0 " >> /etc/fstab

mount -av

df -T

==========NFS==================

apt-get update && apt-get install nfs-server nfs-utils -y

mkdir /raid/nfs

chmod 777 /raid/nfs

cp /etc/exports /etc/exports.back

echo "/raid/nfs 192.168.2.0/27(rw,no_subtree_check,no_root_squash)" >> /etc/exports

systemctl enable --now nfs-server

# exportfs -vra

===========HQ-CLI

mkdir /mnt/nfs

chmod -R 777 /mnt/nfs

showmount -e hq-srv

cp /etc/fstab /etc/fstab.back

echo "192.168.1.10:/raid/nfs /mnt/nfs nfs rw,soft,_netdev 0 0 " >> /etc/fstab

mount -av

df -T

создать файл, посмотреть на второй стороне

==========WEB App==================

mount -o loop /dev/sr0 /mnt/ -v

apt-get install lamp-server -y

cp /mnt/web/index.php /var/www/html

cp /mnt/web/logo.png /var/www/html

systemctl enable --now mariadb

mariadb -e "CREATE DATABASE webdb;"

mariadb -e "

CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';

GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost';

"

mariadb webdb < /mnt/web/dump.sql

mariadb -e "USE webdb; SHOW TABLES;"

vim /var/www/html/index.php

$servername = "localhost";

$username = "web";

$password = "P@ssw0rd";

$dbname = "webdb";

systemctl enable --now httpd2.service

===========HQ-CLI===============

переходим на HQ-CLI в браузере

web.au-team.irpo, аутентификация WEB/P@ssw0rd

apt-get update && apt-get install yandex-browser-stable -y

echo "server 172.16.1.1 iburst" >> /etc/chrony.conf && systemctl restart chronyd
