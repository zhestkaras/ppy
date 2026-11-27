Acquire::http::Proxy "http://login:password@proxy.example.com:8080/";
Acquire::https::Proxy "http://login:password@proxy.example.com:8080/";

sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo iptables -A FORWARD -i enp0s9 -o enp0s3 -j ACCEPT
sudo iptables -A FORWARD -i enp0s3 -o enp0s9 -m state --state RELATED,ESTABLISHED -j ACCEPT


server {
    listen 80;
    server_name moodle;

    root /var/www/html;
    index index.php index.html;

    location = /quinque.local {
        return 301 /quinque.local/;
    }

    location /quinque.local/ {
        try_files $uri $uri/ /quinque.local/index.php?$query_string;
    }

    location ~ ^/quinque\.local/.+\.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }
}

sudo ln -s /etc/nginx/sites-available/moodle-new /etc/nginx/sites-enabled/moodle-new

sudo cp /var/www/html/moodle/config.php /var/www/html/moodle/config.php.backup

php -m | grep -Ei 'curl|dom|gd|mbstring|openssl|posix|session|simplexml|xmlreader|xmlwriter|zip|zlib|pdo_mysql'

php -m | grep -Ei 'fileinfo|intl|pdo_mysql|bz2|sodium'

wget https://download.nextcloud.com/server/releases/latest.zip

sudo chown -R www-data:www-data /var/www/nextcloud
sudo chmod -R 750 /var/www/nextcloud

CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'ncuser'@'localhost' IDENTIFIED BY 'NcCloud_2026_Strong!';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'ncuser'@'localhost';
FLUSH PRIVILEGES;

server {
    listen 80;
    listen [::]:80;

    server_name quinque.local;

    root /var/www/nextcloud;
    index index.php index.html /index.php$request_uri;

    client_max_body_size 512M;
    fastcgi_buffers 64 4K;

    location / {
        try_files $uri $uri/ /index.php$request_uri;
    }

    location ~ \.php(?:$|/) {
        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        set $path_info $fastcgi_path_info;

        try_files $fastcgi_script_name =404;

        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $path_info;

        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}



work@debian:/tmp$ sudo cat /etc/nginx/sites-enabled/moodle
server {
    listen 80;
    server_name quinque.local;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name quinque.local;

    ssl_certificate /etc/nginx/ssl/bundle.pem;
    ssl_certificate_key /etc/nginx/ssl/private.key;

    root /var/www/html/moodle;
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ [^/]\.php(/|$) {
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;

        fastcgi_param HTTPS on;
        fastcgi_param HTTP_X_FORWARDED_PROTO https;
     }

    location ^~ /phpmyadmin/ {
        alias /usr/share/phpmyadmin/;
        index index.php;
        location ~ ^/phpmyadmin/(.+.php)$ {
            alias /usr/share/phpmyadmin/$1;
            fastcgi_pass unix:/run/php/php8.4-fpm.sock;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $request_filename;
            fastcgi_param HTTPS on;
            fastcgi_param HTTP_X_FORWARDED_PROTO https;
        }
     }
}
work@debian:/tmp$ sudo cat /etc/nginx/sites-enabled/moodle-new
server {
    listen 80;
    server_name moodle;

    root /var/www/html;
    index index.php index.html;

    location = /quinque.local {
        return 301 /quinque.local/;
    }

    location /quinque.local/ {
        try_files $uri $uri/ /quinque.local/index.php?$query_string;
    }

    location ~ ^/quinque\.local/.+\.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }
}
work@debian:/tmp$ sudo cat /etc/nginx/sites-enabled/nextcloud
server {
    listen 80;
    listen [::]:80;

    server_name quinque.local;

    root /var/www/nextcloud;
    index index.php index.html /index.php$request_uri;

    client_max_body_size 512M;
    fastcgi_buffers 64 4K;

    location / {
        try_files $uri $uri/ /index.php$request_uri;
    }

    location ~ \.php(?:$|/) {
        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        set $path_info $fastcgi_path_info;

        try_files $fastcgi_script_name =404;

        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $path_info;

        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
work@debian:/tmp$ ls -l /etc/nginx/sites-enabled/
total 0
lrwxrwxrwx 1 root root 33 Nov 16 21:11 moodle -> /etc/nginx/sites-available/moodle
lrwxrwxrwx 1 root root 37 Nov 25 23:16 moodle-new -> /etc/nginx/sites-available/moodle-new
lrwxrwxrwx 1 root root 36 Nov 26 19:55 nextcloud -> /etc/nginx/sites-available/nextcloud
lrwxrwxrwx 1 root root 37 Nov 16 21:12 phpmyadmin -> /etc/nginx/sites-available/phpmyadmin
work@debian:/tmp$


sudo ln -s /etc/nginx/sites-available/nextcloud /etc/nginx/sites-enabled/

sudo grep -RniE 'quinque\.local|return 30[12]|rewrite' /etc/nginx/sites-enabled/






























work@debian:/tmp$ sudo cat /etc/nginx/sites-enabled/moodle
[sudo] password for work:
server {
    listen 80;
    server_name quinque.local;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name quinque.local;

    ssl_certificate /etc/nginx/ssl/bundle.pem;
    ssl_certificate_key /etc/nginx/ssl/private.key;

    root /var/www/html/moodle;
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ [^/]\.php(/|$) {
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;

        fastcgi_param HTTPS on;
        fastcgi_param HTTP_X_FORWARDED_PROTO https;
     }

    location ^~ /phpmyadmin/ {
        alias /usr/share/phpmyadmin/;
        index index.php;
        location ~ ^/phpmyadmin/(.+.php)$ {
            alias /usr/share/phpmyadmin/$1;
            fastcgi_pass unix:/run/php/php8.4-fpm.sock;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $request_filename;
            fastcgi_param HTTPS on;
            fastcgi_param HTTP_X_FORWARDED_PROTO https;
        }
     }
}


work@debian:/tmp$ sudo cat /etc/nginx/sites-enabled/moodle-new
server {
    listen 80;
    server_name moodle;

    root /var/www/html;
    index index.php index.html;

    location = /quinque.local {
        return 301 /quinque.local/;
    }

    location /quinque.local/ {
        try_files $uri $uri/ /quinque.local/index.php?$query_string;
    }

    location ~ ^/quinque\.local/.+\.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }
}








work@debian:/tmp$ sudo cat /etc/nginx/sites-enabled/nextcloud
server {
    listen 80;
    listen [::]:80;

    server_name quinque.local;

    root /var/www/nextcloud;
    index index.php index.html /index.php$request_uri;

    client_max_body_size 512M;
    fastcgi_buffers 64 4K;

    location / {
        try_files $uri $uri/ /index.php$request_uri;
    }

    location ~ \.php(?:$|/) {
        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        set $path_info $fastcgi_path_info;

        try_files $fastcgi_script_name =404;

        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $path_info;

        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}


work@debian:/tmp$ curl -s http://quinque.local | grep -i nextcloud
                        Nextcloud               </title>
                                                Nextcloud                                      <        /h1>
                                <a href="https://nextcloud.com" target="_blank" rel="noreferrer         noopener">Nextcloud</a> – a safe home for all your data                 </p>





work@debian:/tmp$ ls -ld /var/www/nextcloud/data
drwxr-xr-x 4 www-data www-data 4096 Nov 26 21:07 /var/www/nextcloud/data
work@debian:/tmp$ sudo tail -n 30 /var/log/nginx/error.log
2026/11/26 21:08:55 [error] 1565#1565: *45 upstream timed out (110: Connection timed out) while reading              response header from upstream, client: 10.0.2.2, server: quinque.local, request: "GET /ocs/v2.php/apps/a             ppstore/api/v1/apps HTTP/1.1", upstream: "fastcgi://unix:/run/php/php8.4-fpm.sock", host: "quinque.local             "
work@debian:/tmp$ sudo tail -n 30 /var/log/nginx/error.log
2026/11/26 21:08:55 [error] 1565#1565: *45 upstream timed out (110: Connection timed out) while reading              response header from upstream, client: 10.0.2.2, server: quinque.local, request: "GET /ocs/v2.php/apps/a             ppstore/api/v1/apps HTTP/1.1", upstream: "fastcgi://unix:/run/php/php8.4-fpm.sock", host: "quinque.local             "
work@debian:/tmp$ sudo journalctl -u php8.4-fpm -n 20 --no0-pager
journalctl: unrecognized option '--no0-pager'
work@debian:/tmp$ sudo journalctl -u php8.4-fpm -n 20 --no-pager
Nov 24 21:42:21 debian systemd[1]: Started php8.4-fpm.service - The PHP 8.4 FastCGI Process Manager.
Nov 24 21:49:06 debian systemd[1]: Stopping php8.4-fpm.service - The PHP 8.4 FastCGI Process Manager...
Nov 24 21:49:07 debian systemd[1]: php8.4-fpm.service: Deactivated successfully.
Nov 24 21:49:07 debian systemd[1]: Stopped php8.4-fpm.service - The PHP 8.4 FastCGI Process Manager.
-- Boot fc72af1d7c9b4845b02cea326a7e3232 --
Nov 24 21:49:27 debian systemd[1]: Starting php8.4-fpm.service - The PHP 8.4 FastCGI Process Manager...
Nov 24 21:49:27 debian systemd[1]: Started php8.4-fpm.service - The PHP 8.4 FastCGI Process Manager.
Nov 24 21:50:07 debian systemd[1]: Stopping php8.4-fpm.service - The PHP 8.4 FastCGI Process Manager...
Nov 24 21:50:07 debian systemd[1]: php8.4-fpm.service: Deactivated successfully.
Nov 24 21:50:07 debian systemd[1]: Stopped php8.4-fpm.service - The PHP 8.4 FastCGI Process Manager.
-- Boot b6aeb9dd2768499aa4d43fac407c2563 --
Nov 24 21:50:26 debian systemd[1]: Starting php8.4-fpm.service - The PHP 8.4 FastCGI Process Manager...
Nov 24 21:50:26 debian systemd[1]: Started php8.4-fpm.service - The PHP 8.4 FastCGI Process Manager.
Nov 24 21:57:20 debian systemd[1]: Stopping php8.4-fpm.service - The PHP 8.4 FastCGI Process Manager...
Nov 24 21:57:20 debian systemd[1]: php8.4-fpm.service: Deactivated successfully.
Nov 24 21:57:20 debian systemd[1]: Stopped php8.4-fpm.service - The PHP 8.4 FastCGI Process Manager.
-- Boot 48e0b43703c8436eb27efdc62846919e --
Nov 24 21:57:38 debian systemd[1]: Starting php8.4-fpm.service - The PHP 8.4 FastCGI Process Manager...
Nov 24 21:57:38 debian systemd[1]: Started php8.4-fpm.service - The PHP 8.4 FastCGI Process Manager.
-- Boot a5f21ffe5eb54497baa08048969a66f6 --
Nov 25 22:22:19 debian systemd[1]: Starting php8.4-fpm.service - The PHP 8.4 FastCGI Process Manager...
Nov 25 22:22:19 debian systemd[1]: Started php8.4-fpm.service - The PHP 8.4 FastCGI Process Manager.
-- Boot 4f8b213426f34afd9176b8b04131bbce --
Nov 26 18:03:54 debian systemd[1]: Starting php8.4-fpm.service - The PHP 8.4 FastCGI Process Manager...
Nov 26 18:03:54 debian systemd[1]: Started php8.4-fpm.service - The PHP 8.4 FastCGI Process Manager.


sudo grep -nE 'fastcgi_read_timeout|fastcgi_connect_timeout' /etc/nginx/sites-available/nextcloud
