## ===== [ jinchi (router) ] =====
``` bash
#!/bin/bash

echo 1 > /proc/sys/net/ipv4/ip_forward

cat <<EOF > /etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp

auto eth1
iface eth1 inet static
    address 192.242.1.1
    netmask 255.255.255.0

auto eth2
iface eth2 inet static
    address 192.242.2.1
    netmask 255.255.255.0

auto eth3
iface eth3 inet static
    address 192.242.3.1
    netmask 255.255.255.0

auto eth4
iface eth4 inet static
    address 192.242.4.1
    netmask 255.255.255.0
EOF

dhclient eth0 2>/dev/null || true

ip addr flush dev eth1
ip addr add 192.242.1.1/24 dev eth1
ip link set eth1 up

ip addr flush dev eth2
ip addr add 192.242.2.1/24 dev eth2
ip link set eth2 up

ip addr flush dev eth3
ip addr add 192.242.3.1/24 dev eth3
ip link set eth3 up

ip addr flush dev eth4
ip addr add 192.242.4.1/24 dev eth4
ip link set eth4 up

iptables -t nat -F
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

iptables -A FORWARD -i eth0 -o eth1 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -o eth2 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth2 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -o eth3 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth3 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -o eth4 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth4 -o eth0 -j ACCEPT

export DEBIAN_FRONTEND=noninteractive
apt-get update ; apt-get install -y isc-dhcp-relay

pkill dhcrelay 2>/dev/null || true
/usr/sbin/dhcrelay -4 -i eth2 -i eth3 -i eth4 192.242.4.10 
``` 

## ======= [ SWITCH 1 ] =======

### ===== [ utahime, shoko, gojo ] =====
```bash
#!/bin/bash
ip addr flush dev eth0
ip link set eth0 up
# ip sesuai dengan tiap nodes
ip addr add 192.242.1.x/24 dev eth0 
ip route add default via 192.242.1.1

echo "nameserver 192.168.122.1" > /etc/resolv.conf
echo "x ready."
```

## ======= [ SWITCH 10 ] =======

### ==== [ tengen ] =====
```bash
#!/bin/bash
echo "nameserver 192.168.122.1" > /etc/resolv.conf

ip addr flush dev eth0
ip link set eth0 up
ip addr add 192.242.4.10/24 dev eth0
ip route add default via 192.242.4.1

export DEBIAN_FRONTEND=noninteractive
apt-get update ; apt-get install -y isc-dhcp-server

mkdir -p /var/lib/dhcp
touch /var/lib/dhcp/dhcpd.leases
sed -i 's/INTERFACESv4=.*/INTERFACESv4="eth0"/' /etc/default/isc-dhcp-server

cat <<'EOF' > /etc/dhcp/dhcpd.conf
authoritative;
default-lease-time 600;
max-lease-time 7200;
option domain-name-servers 192.242.1.2, 192.242.1.3;
option domain-search "K62.com";

subnet 192.242.4.0 netmask 255.255.255.0 {
}

class "pool1" { match hardware; }
subclass "pool1" 1:de:ad:be:ef:62:02;   # itadori
subclass "pool1" 1:de:ad:be:ef:62:03;   # fushiguro

class "pool2" { match hardware; }
subclass "pool2" 1:de:ad:be:ef:62:04;   # zenin
subclass "pool2" 1:de:ad:be:ef:62:05;   # okkotsu

class "short-lease" {
    match hardware;
    default-lease-time 1800;
    max-lease-time 1800;
}
subclass "short-lease" 1:de:ad:be:ef:62:06;   # ino
subclass "short-lease" 1:de:ad:be:ef:62:07;   # kusakabe

class "long-lease" {
    match hardware;
    default-lease-time 43200;
    max-lease-time 43200;
}
subclass "long-lease" 1:de:ad:be:ef:62:08;    # miwa

subnet 192.242.2.0 netmask 255.255.255.0 {
    option routers 192.242.2.1;
    option subnet-mask 255.255.255.0;
    option broadcast-address 192.242.2.255;

    pool {
        range 192.242.2.10 192.242.2.35;
        allow members of "pool1";
    }
    pool {
        range 192.242.2.70 192.242.2.95;
        allow members of "pool2";
    }
}

subnet 192.242.3.0 netmask 255.255.255.0 {
    option routers 192.242.3.1;
    option subnet-mask 255.255.255.0;
    option broadcast-address 192.242.3.255;

    pool {
        range 192.242.3.10 192.242.3.89;
        allow members of "short-lease";
    }
    pool {
        range 192.242.3.100 192.242.3.199;
        allow members of "long-lease";
    }

      host kugisaki {
        hardware ethernet de:ad:be:ef:62:01;
        fixed-address 192.242.3.99;
    }
}
EOF

dhcpd -t -4 -cf /etc/dhcp/dhcpd.conf || exit 1

pkill dhcpd 2>/dev/null || true
dhcpd -4 -pf /var/run/dhcpd.pid -cf /etc/dhcp/dhcpd.conf eth0
```

### ===== [ yaga ] =====
```bash
#!/bin/bash
echo "nameserver 192.168.122.1" > /etc/resolv.conf

ip addr flush dev eth0
ip link set eth0 up
ip addr add 192.242.4.11/24 dev eth0
ip route add default via 192.242.4.1

export DEBIAN_FRONTEND=noninteractive
apt-get update ; apt-get install -y mariadb-server

mkdir -p /var/log/mysql
chown mysql:mysql /var/log/mysql

cat <<EOF > /etc/mysql/mariadb.conf.d/99-master.cnf
[mysqld]
bind-address = 0.0.0.0
skip-name-resolve
server-id    = 1
log_bin      = /var/log/mysql/mysql-bin.log
EOF

service mariadb restart || service mysql restart
sleep 5

mysql <<EOF
CREATE DATABASE IF NOT EXISTS db_shibuya_K62;
USE db_shibuya_K62;
CREATE TABLE IF NOT EXISTS request_logs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    worker VARCHAR(50),
    endpoint VARCHAR(255),
    client_ip VARCHAR(45),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE USER IF NOT EXISTS 'shibuya'@'192.242.4.%' IDENTIFIED BY 'shibuya123';
GRANT ALL PRIVILEGES ON db_shibuya_K62.* TO 'shibuya'@'192.242.4.%';

CREATE USER IF NOT EXISTS 'repl'@'192.242.4.%' IDENTIFIED BY 'repl123';
GRANT REPLICATION SLAVE, REPLICATION CLIENT, RELOAD ON *.* TO 'repl'@'192.242.4.%';
GRANT SELECT, LOCK TABLES ON db_shibuya_K62.* TO 'repl'@'192.242.4.%';
FLUSH PRIVILEGES;
EOF

mysql -e "SHOW MASTER STATUS;"
```

### ==== [ inumaki ] =====
```bash
#!/bin/bash
echo "nameserver 192.168.122.1" > /etc/resolv.conf

ip addr flush dev eth0
ip link set eth0 up
ip addr add 192.242.4.12/24 dev eth0
ip route add default via 192.242.4.1

export DEBIAN_FRONTEND=noninteractive
apt-get update ; apt-get install -y mariadb-server

mkdir -p /var/log/mysql
chown mysql:mysql /var/log/mysql

cat <<EOF > /etc/mysql/mariadb.conf.d/99-slave.cnf
[mysqld]
bind-address = 0.0.0.0
skip-name-resolve
server-id    = 2
relay_log    = /var/log/mysql/relay-bin
EOF

service mariadb restart || service mysql restart
sleep 5

mysql -e "STOP SLAVE; RESET SLAVE ALL;
CHANGE MASTER TO MASTER_HOST='192.242.4.11', MASTER_USER='repl', MASTER_PASSWORD='repl123', MASTER_CONNECT_RETRY=10;"

mysqldump -h 192.242.4.11 -urepl -prepl123 --single-transaction --master-data=1 --databases db_shibuya_K62 | mysql
mysql -e "START SLAVE;"
sleep 2

mysql -e "SHOW SLAVE STATUS\G" | grep -E "Slave_IO_Running|Slave_SQL_Running|Last_.*Error"
```

## ======= [ SWITCH 9 ] =======

### ===== [ geto.sh ] =====
```bash
#!/bin/bash
echo "nameserver 192.168.122.1" > /etc/resolv.conf

ip addr flush dev eth0
ip link set eth0 up
ip addr add 192.242.4.20/24 dev eth0
ip route add default via 192.242.4.1

export DEBIAN_FRONTEND=noninteractive
apt-get update ; apt-get install -y nginx curl

cat <<'EOF' > /etc/nginx/sites-enabled/default
upstream api_cluster {
    server 192.242.4.23 weight=3 max_fails=2 fail_timeout=5s;   # nanami
    server 192.242.4.24 weight=1 max_fails=2 fail_timeout=5s;   # todo
    server 192.242.4.25 backup;                                 # kirara
}

server {
    listen 80 default_server;
    server_name jujutsu.K62.com www.jujutsu.K62.com;

    set_real_ip_from 192.242.1.4;
    real_ip_header X-Forwarded-For;

    location / {
        proxy_pass http://api_cluster;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_connect_timeout 2s;
        proxy_next_upstream error timeout http_502 http_503 http_504;
    }
}
EOF

nginx -t && service nginx restart
```

### ===== [ maki ] =====
```bash
#!/bin/bash
echo "nameserver 192.168.122.1" > /etc/resolv.conf

ip addr flush dev eth0
ip link set eth0 up
ip addr add 192.242.4.21/24 dev eth0
ip route add default via 192.242.4.1

export DEBIAN_FRONTEND=noninteractive
apt-get update ; apt-get install -y nginx php-fpm

PHP=$(ls /etc/php | sort -V | tail -n1)
mkdir -p /run/php /var/www/frontend

cat <<'EOF' > /var/www/frontend/index.php
<?php echo "Hostname: maki"; ?>
EOF

cat <<EOF > /etc/nginx/sites-enabled/default
server {
    listen 80 default_server;
    server_name jujutsu.K62.com www.jujutsu.K62.com;
    root /var/www/frontend;
    index index.php;

    allow 192.242.1.4;
    allow 127.0.0.1;
    deny all;

    location / {
        try_files \$uri \$uri/ /index.php?\$query_string;
    }
    location ~ \.php\$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php${PHP}-fpm.sock;
    }
}
EOF

nginx -t && service php${PHP}-fpm restart && service nginx restart
```

### ===== [ panda ] =====
```bash
#!/bin/bash
echo "nameserver 192.168.122.1" > /etc/resolv.conf

ip addr flush dev eth0
ip link set eth0 up
ip addr add 192.242.4.22/24 dev eth0
ip route add default via 192.242.4.1

export DEBIAN_FRONTEND=noninteractive
apt-get update ; apt-get install -y nginx php-fpm

PHP=$(ls /etc/php | sort -V | tail -n1)
mkdir -p /run/php /var/www/frontend

cat <<'EOF' > /var/www/frontend/index.php
<?php echo "Hostname: panda"; ?>
EOF

cat <<EOF > /etc/nginx/sites-enabled/default
server {
    listen 80 default_server;
    server_name jujutsu.K62.com www.jujutsu.K62.com;
    root /var/www/frontend;
    index index.php;

    allow 192.242.1.4;
    allow 127.0.0.1;
    deny all;

    location / {
        try_files \$uri \$uri/ /index.php?\$query_string;
    }
    location ~ \.php\$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php${PHP}-fpm.sock;
    }
}
EOF

nginx -t && service php${PHP}-fpm restart && service nginx restart
```

### ===== [ nanami ] =====
```bash
#!/bin/bash
echo "nameserver 192.168.122.1" > /etc/resolv.conf

ip addr flush dev eth0
ip link set eth0 up
ip addr add 192.242.4.23/24 dev eth0
ip route add default via 192.242.4.1

export DEBIAN_FRONTEND=noninteractive
apt-get update ; apt-get install -y nginx php-fpm php-mysql

PHP=$(ls /etc/php | sort -V | tail -n1)
mkdir -p /run/php /var/www/api

cat <<'EOF' > /var/www/api/index.php
<?php
header('Content-Type: application/json');
$ip = $_SERVER['HTTP_X_REAL_IP'] ?? '';
if (!filter_var($ip, FILTER_VALIDATE_IP)) $ip = $_SERVER['REMOTE_ADDR'];

mysqli_report(MYSQLI_REPORT_OFF);
$db = @new mysqli('192.242.4.11', 'shibuya', 'shibuya123', 'db_shibuya_K62');
if ($db->connect_errno) {
    http_response_code(500);
    echo json_encode(['status' => 'error', 'worker' => 'nanami', 'message' => 'db connect failed']);
    exit;
}
$st = $db->prepare('INSERT INTO request_logs (worker, endpoint, client_ip) VALUES (?, ?, ?)');
$w = 'nanami'; $ep = $_SERVER['REQUEST_URI'];
$st->bind_param('sss', $w, $ep, $ip);
$st->execute();
echo json_encode(['status' => 'ok', 'worker' => $w, 'client_ip' => $ip]);
EOF

cat <<EOF > /etc/nginx/sites-enabled/default
server {
    listen 80 default_server;
    server_name jujutsu.K62.com www.jujutsu.K62.com;
    root /var/www/api;

    allow 192.242.4.20;
    allow 127.0.0.1;
    deny all;

    location / {
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME \$document_root/index.php;
        fastcgi_pass unix:/run/php/php${PHP}-fpm.sock;
    }
}
EOF

nginx -t && service php${PHP}-fpm restart && service nginx restart
```

### ===== [ todo ] =====
```bash
#!/bin/bash
echo "nameserver 192.168.122.1" > /etc/resolv.conf

ip addr flush dev eth0
ip link set eth0 up
ip addr add 192.242.4.24/24 dev eth0
ip route add default via 192.242.4.1

export DEBIAN_FRONTEND=noninteractive
apt-get update ; apt-get install -y nginx php-fpm php-mysql

PHP=$(ls /etc/php | sort -V | tail -n1)
mkdir -p /run/php /var/www/api

cat <<'EOF' > /var/www/api/index.php
<?php
header('Content-Type: application/json');
$ip = $_SERVER['HTTP_X_REAL_IP'] ?? '';
if (!filter_var($ip, FILTER_VALIDATE_IP)) $ip = $_SERVER['REMOTE_ADDR'];

mysqli_report(MYSQLI_REPORT_OFF);
$db = @new mysqli('192.242.4.11', 'shibuya', 'shibuya123', 'db_shibuya_K62');
if ($db->connect_errno) {
    http_response_code(500);
    echo json_encode(['status' => 'error', 'worker' => 'todo', 'message' => 'db connect failed']);
    exit;
}
$st = $db->prepare('INSERT INTO request_logs (worker, endpoint, client_ip) VALUES (?, ?, ?)');
$w = 'todo'; $ep = $_SERVER['REQUEST_URI'];
$st->bind_param('sss', $w, $ep, $ip);
$st->execute();
echo json_encode(['status' => 'ok', 'worker' => $w, 'client_ip' => $ip]);
EOF

cat <<EOF > /etc/nginx/sites-enabled/default
server {
    listen 80 default_server;
    server_name jujutsu.K62.com www.jujutsu.K62.com;
    root /var/www/api;

    allow 192.242.4.20;
    allow 127.0.0.1;
    deny all;

    location / {
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME \$document_root/index.php;
        fastcgi_pass unix:/run/php/php${PHP}-fpm.sock;
    }
}
EOF

nginx -t && service php${PHP}-fpm restart && service nginx restart
```

### ===== [ kirara ] =====
```bash
#!/bin/bash
echo "nameserver 192.168.122.1" > /etc/resolv.conf

ip addr flush dev eth0
ip link set eth0 up
ip addr add 192.242.4.25/24 dev eth0
ip route add default via 192.242.4.1

export DEBIAN_FRONTEND=noninteractive
apt-get update ; apt-get install -y nginx php-fpm php-mysql

PHP=$(ls /etc/php | sort -V | tail -n1)
mkdir -p /run/php /var/www/api

cat <<'EOF' > /var/www/api/index.php
<?php
header('Content-Type: application/json');
$ip = $_SERVER['HTTP_X_REAL_IP'] ?? '';
if (!filter_var($ip, FILTER_VALIDATE_IP)) $ip = $_SERVER['REMOTE_ADDR'];

mysqli_report(MYSQLI_REPORT_OFF);
$db = @new mysqli('192.242.4.11', 'shibuya', 'shibuya123', 'db_shibuya_K62');
if ($db->connect_errno) {
    http_response_code(500);
    echo json_encode(['status' => 'error', 'worker' => 'kirara', 'message' => 'db connect failed']);
    exit;
}
$st = $db->prepare('INSERT INTO request_logs (worker, endpoint, client_ip) VALUES (?, ?, ?)');
$w = 'kirara'; $ep = $_SERVER['REQUEST_URI'];
$st->bind_param('sss', $w, $ep, $ip);
$st->execute();
echo json_encode(['status' => 'ok', 'worker' => $w, 'client_ip' => $ip]);
EOF

cat <<EOF > /etc/nginx/sites-enabled/default
server {
    listen 80 default_server;
    server_name jujutsu.K62.com www.jujutsu.K62.com;
    root /var/www/api;

    allow 192.242.4.20;
    allow 127.0.0.1;
    deny all;

    location / {
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME \$document_root/index.php;
        fastcgi_pass unix:/run/php/php${PHP}-fpm.sock;
    }
}
EOF

nginx -t && service php${PHP}-fpm restart && service nginx restart
```

## ======= [ SWITCH 5, 6, 7, 8 ] =======
isi script sama, beda di addr belakang (...62:0x) dan switch.
untuk itadori, fushiguro, zenin, okkotsu, ino, Kusakabe, kugisaki, dan miwa

### ===== [ SWITCH 7 dan 8 ] =====
```bash
#!/bin/bash

ip link set eth0 down
# beda di ..62:0x
ip link set eth0 address de:ad:be:ef:62:0x
ip link set eth0 up
ip addr flush dev eth0

if ! command -v dhclient >/dev/null; then
    ip addr add 192.242.3.94/24 dev eth0
    ip route add default via 192.242.3.1
    echo "nameserver 192.168.122.1" > /etc/resolv.conf
    export DEBIAN_FRONTEND=noninteractive
    apt-get update ; apt-get install -y isc-dhcp-client
    ip addr flush dev eth0
fi

pkill dhclient 2>/dev/null || true
dhclient -v eth0
ip -br addr show eth0
ip route
```

### ===== [ SWITCH 5 dan 6 ] =====
```bash
#!/bin/bash

ip link set eth0 down
# beda di ..62:0x
ip link set eth0 address de:ad:be:ef:62:0x
ip link set eth0 up
ip addr flush dev eth0

if ! command -v dhclient >/dev/null; then
    ip addr add 192.242.3.94/24 dev eth0
    ip route add default via 192.242.2.1
    echo "nameserver 192.168.122.1" > /etc/resolv.conf
    export DEBIAN_FRONTEND=noninteractive
    apt-get update ; apt-get install -y isc-dhcp-client
    ip addr flush dev eth0
fi

pkill dhclient 2>/dev/null || true
dhclient -v eth0
ip -br addr show eth0
ip route
```