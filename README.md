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
