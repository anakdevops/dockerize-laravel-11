# Membuat network bridge untuk menghubungkan kontainer
```
docker network create laravel_11_network
```
# Menjalankan kontainer MySQL
```
docker run -d \
  --name mysql_db \
  --network laravel_11_network \
  -e MYSQL_ROOT_PASSWORD=my-secret-pw \
  -e MYSQL_DATABASE=my_database \
  mysql:8.3
```
# Menjalankan kontainer phpMyAdmin
```
docker run -d \
  --name phpmyadmin \
  --network laravel_11_network \
  -e PMA_ARBITRARY=1 \
  -p 8080:80 \
  phpmyadmin/phpmyadmin:5.2.1
```

# Clone source code laravel
```
git clone -b 11.x https://github.com/laravel/laravel.git
cd laravel
```

# Create apache.conf
```
<VirtualHost *:80>
ServerName app.anakdevops.com
ServerAdmin ady@anakdevops.com
DocumentRoot /var/www/html/public
<Directory /var/www/html>
AllowOverride All
</Directory>
ErrorLog ${APACHE_LOG_DIR}/error.log
CustomLog ${APACHE_LOG_DIR}/access.log combined
RewriteEngine on
RewriteCond %{SERVER_NAME} =app.anakdevops.com
RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
</VirtualHost>

<IfModule mod_ssl.c>
<VirtualHost *:443>
ServerName app.anakdevops.com
ServerAdmin ady@anakdevops.com
DocumentRoot /var/www/html/public
<Directory /var/www/html>
AllowOverride All
</Directory>
ErrorLog ${APACHE_LOG_DIR}/error.log
CustomLog ${APACHE_LOG_DIR}/access.log combined

#########aaktifkan jika ada file ssl
# SSLCertificateFile var/www/html/apachehost/certificate.crt
# SSLCertificateKeyFile var/www/html/apachehost/private.key
# SSLCertificateChainFile var/www/html/apachehost/ca_bundle.crt
#########aaktifkan jika ada file ssl

</VirtualHost>
</IfModule>
```

# Create Dockerfile
```
FROM php:8.3-apache

COPY . /var/www/html
WORKDIR /var/www/html
COPY .env.example /var/www/html/.env

RUN a2enmod ssl && a2enmod rewrite
RUN apt-get update && apt-get install -y \
        nano \
        libpng-dev \
        zlib1g-dev \
        libxml2-dev \
        libzip-dev \
        libonig-dev \
        zip \
        curl \
        unzip \
    && docker-php-ext-configure gd \
    && docker-php-ext-install -j$(nproc) gd \
    && docker-php-ext-install pdo_mysql \
    && docker-php-ext-install mysqli \
    && docker-php-ext-install zip \
    && docker-php-source delete


COPY apache.conf /etc/apache2/sites-available/000-default.conf
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

RUN chown -R www-data:www-data /var/www/html \
    && a2enmod rewrite 

ENV COMPOSER_ALLOW_SUPERUSER=1

RUN composer install

EXPOSE 80
EXPOSE 443

COPY start.sh /usr/local/bin/startup-script.sh
RUN chmod +x /usr/local/bin/startup-script.sh
CMD ["/usr/local/bin/startup-script.sh"]
```

# Change database config .env.example
```
DB_HOST=mysql_db
DB_PORT=3306
DB_DATABASE=my_database
DB_USERNAME=root
DB_PASSWORD=my-secret-pw
```

# Create file start.sh
```
cp .env.example .env
php artisan config:cache
php artisan migrate
php artisan key:generate
apache2-foreground
```

# Change default database config/database.php to mysql
```
'default' => env('DB_CONNECTION', 'mysql'),
```
# Build image laravel
```
docker build . -t laravel11:v1
```

# Run image laravel
```
docker run -d \
  --name appslaravel11 \
  --network laravel_11_network \
  -p 8011:80 \
  laravel11:v1
```
