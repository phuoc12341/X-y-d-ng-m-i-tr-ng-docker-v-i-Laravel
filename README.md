# Xây dựng môi trường Docker với Laravel

Bài này mình sẽ chia sẻ với các bạn build 1 môi trường Dockẻ chạy Laravel gồm có php-fpm, nginx, mysql.

# Bước 1:
Đầu tiên ta kéo framework Laravel về và bạn phải cấp quyền cho thư mục này là user đang sử dụng (ko phải root) nếu không khi chạy Docker sẽ cần quyền root và mình sẽ phải thêm sudo vào rất lằng nhằng nhé
```
cd ~
git clone https://github.com/laravel/laravel.git laravel-app
```
```
cd ~/laravel-app
sudo chown -R $USER:$USER ~/laravel-app
```

# Bước 2: Tạo 1 file Docker compose

À quên, bước này là trong máy của các bạn đã phải cài Docker và Docker Compose rồi nhé. Nếu chưa các bạn lên trang chủ Docker download nhé
https://docs.docker.com/engine/install/ubuntu/
https://docs.docker.com/compose/install/

Các bạn tạo cho mình 1 file docker-compose.yml, trong này chúng ta định nghĩa ra 3 service: php, nginx, mysql:
```
nano ~/laravel-app/docker-compose.yml
```
```
version: '3'
services:

  #PHP Service
  php:
    build:
      context: .envs/dev
      dockerfile: php/Dockerfile
      args:
        user: $USER
        uid: $GR
    image: phuoc-php
    container_name: app_phuoc
    volumes:
       - ./:/var/www/app
    restart: on-failure
    working_dir: /var/www/app
    networks:
      - app-network

  #Nginx Service
  nginx:
    image: nginx:alpine
    container_name: webserver_phuoc
    volumes:
      - ./:/var/www/app
      - ./.envs/dev/nginx/conf.d/:/etc/nginx/conf.d/
    restart: on-failure
    tty: true
    ports:
      - "88:80"
      - "443:443"
    networks:
      - app-network

  #MySQL Service
  mysql:
    image: mysql:5.7
    container_name: db_phuoc
    volumes:
      - db_data:/var/lib/mysql
      - ./.envs/dev/mysql/my.cnf:/etc/mysql/my.cnf
    restart: on-failure
    ports:
      - "1306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: ms_root
      MYSQL_DATABASE: laravel
      MYSQL_USER: phuoc
      MYSQL_PASSWORD: 1
    networks:
      - app-network

#Docker Networks
networks:
  app-network:
    driver: bridge

#Volumes
volumes:
  db_data:
```

2 service nginx và mysql chúng ta sẽ dùng các image có sẵn trên docker-hub để build container, còn riêng service php chúng ta sẽ tự build image bằng Dockerfile nhé

Giải thích các Service:
php: Trong service này sẽ chứa ứng dụng Laravel, working_dir là /var/www/app

nginx: Service này sẽ kéo image nginx:alpine từ Docker về và expose 2 port 80 và 443

mysql: Service này sẽ kéo image mysql:5.7 từ Dockerhub về. Có 1 vài định nghĩa biến môi trường ở đây để cấu hình cho mysql, chúng ta tự điền nhé.


# Bước 3: Ràng buộc data

Trong docker-compose file chúng ta định nghĩa 1 volumn được gọi là da_data trong mysql service để ràng buộc MYSQL database
```
mysql:
  ...
    volumes:
      - dbdata:/var/lib/mysql
    networks:
      - app-network
  ...
```

Volumn dbdata sẽ persit nội dung của thư mục /var/lib/mysql bên trong container. Như vậy khi chúng ta stop hoặc restart mysql service mà không bị mất dự liệu

Như vậy ở bên dưới ta sẽ định nghĩa volumn dbdata như sau:

```
...
#Volumes
volumes:
  dbdata:
    driver: local
```
Như vậy, volumn dbdata này sẽ được sử dụng trong nhiều service


Trong service nginx chúng ta sẽ bind mount 1 thư mục .envs/dev/nginx/conf.d/ của host vào /etc/nginx/conf.d/ trong container để định nghĩa cấu hình cho nginx.

Như vậy chúng ta sẽ tạo ra 1 file cấu hình chio nginx như sau .envs/dev/nginx/conf.d/app.conf:
```
server {
    listen 80;
    index index.php index.html;

    error_log /var/www/app/.envs/dev/nginx/logs/error.log;
    access_log /var/www/app/.envs/dev/nginx/logs/access.log;

    root /var/www/app/public;
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
    location / {
        try_files $uri $uri/ /index.php?$query_string;
        gzip_static on;
    }
}
```

Cấu hình trên chúng ta đã config chạy cho file .php nhé, nó sẽ redirect sang service app:9000 của service php mà chúng ta sẽ định nghĩa. Đoạn cấu hình trên rất cơ bản các bạn chưa rõ có thể tự tìm hiểu thêm nhé

Ok, như vậy giờ còn service php, chúng ta sẽ tạo 1 file Dockerfile trong .envs/dev/php/Dockerfile như sau:

```
FROM php:7.4-fpm

# Arguments defined in docker-compose.yml
ARG user
ARG uid

# Install dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    zip \
    unzip \
    sudo

# Clear cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Install extensions
RUN docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd

# Install composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Set working directory
WORKDIR /var/www/app

# Create system user to run Composer and Artisan Commands
RUN useradd -G www-data,root -u $uid -d /home/$user $user
RUN mkdir -p /home/$user/.composer && \
    chown -R $user:$user /home/$user

COPY php/*.sh /scripts/

RUN chmod a+x /scripts/*.sh

# Make php-fpm user can sudo without password
RUN sudo echo "phuoc ALL=(ALL:ALL) NOPASSWD: ALL" > /etc/sudoers.d/phuoc

USER $user

EXPOSE 9000

ENTRYPOINT ["/scripts/entrypoint.sh"]

CMD ["/scripts/command.sh"]
```
Giải thích sơ qua về Dockerfile:
FROM php:7.4-fpm: Dockerfile này được build dựa trên image php:7.4-fpm trên Dockerhub
ARG user, ARG uid: khai báo 2 argument được truyền vào từ docker-compose file, ở đay mình muốn owner trong container này là user hiện tại của hệ thống

Tiếp theo các đoạn phaias dưới là install các dependency và extension cho container, rồi cài composer

WORKDIR /var/www/app: thiết lập pwd của hệ thống

RUN useradd -G www-data,root -u $uid -d /home/$user $user: Tạo ra 1 user và gán quyền cho user này là user hiện tại của hệ thống

COPY php/*.sh /scripts/: copy thư mục bên ngoài vào hệ thống

USER $user: Set user hiện tại cho container 

EXPOSE 9000: Mở port 9000

ENTRYPOINT ["/scripts/entrypoint.sh"]: Định nghĩa các shell command chúng ta muốn chạy

CMD ["/scripts/command.sh"]: Khai báo các shell command sẽ được chạy mặc định

Ở trong Dockerfile này ta đã cài composer và expose port 9000

2 câu lệnh ENTRYPOINT và CMD sẽ chạy shell trong 2 file command.sh và entrypoint.sh. Như vậy ta sẽ tạo 2 file này nhé.

file command.sh
```
#!/bin/bash

sudo php-fpm
```
file entrypoint.sh
```
exec "$@"
```

Ok như vậy chúng ta cơ bản đã hoàn thành xong


Bây giờ chúng ta sẽ build image service php như sau:
```
docker-compose build
```
Sau khi build xong chúng ta sẽ chạy file docker-compose để build môi trường như sau:
```
docker-compose up
```
chúng ta nhớ là copy file .env từ .env.example ra nhé

sau đó vào container php tạo key cho ứng dungj:
```
docker-compose exec app php artisan key:generate
```
OK như vậy là xong, chúng ta vào http://127.0.0.1:88 để kiểm tra kết quả nhé, cũng khá nhanh phải không nào. Tuy nhiên đây cx là môi trường khá đơn giản thôi, nên nếu muốn có môi trường đầy đủ hơn và tiện dụng hown chúng ta sẽ phải viết shell script để tối ưu hơn nhé...

Tài liệu tham khảo:
https://www.digitalocean.com/community/tutorials/how-to-set-up-laravel-nginx-and-mysql-with-docker-compose






