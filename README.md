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
