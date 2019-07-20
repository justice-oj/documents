# Introduction

[Justice](https://www.justice.plus/) is an [online-judge system](https://en.wikipedia.org/wiki/Online_judge) which supports Java, C and CPP.

Now `Justice` is composed of three parts:

- [frontend](https://github.com/justice-oj/frontend): the website of `Justice` online judge with an additional admin control panel, powered by yii2;

- [dispatcher](https://github.com/justice-oj/dispatcher): dispatches submissions to a local worker(AKA `justice-sandbox`) and fetches the results, powered by spring-boot;

- [sandbox](https://github.com/justice-oj/sandbox): yet another sandbox written in Go, providing kernel-based namespace and cgroup isolation.

# Blogs about Justice
[Blogs](https://tech.liuchao.me/tag/justice-oj/)

# Deployment Guide

Here is a deployment guide for [Justice](https://www.justice.plus/) online judge.

![Architecture](/images/architecture.png)

The picture above describes the main judging process of `Justice`:

- user submits a piece of code, `justice-frontend` then updates MySQL and puts the judge submission to RabbitMQ;

- `justice-dispatcher` fetches the judge submission from RabbitMQ: if the submitted code is written in Java, the dispatcher will judge the code itself, or it will delegate the code to `justice-sandbox` for judging;

- `justice-sandbox` runs the submitted code in a jailed environment, and returns the judge result in a json-encoded string;

- `justice-dispatcher` then updates MySQL / Redis with the result decoded from `justice-sandbox`.

For now [Justice](https://www.justice.plus/) is tested on [Debian Buster](https://wiki.debian.org/DebianBuster) only, migrating the following instructions to any other dist may cause security issues.

## Components

### IP aliases

Replace the following ip addresses in your hosts file for justice dev environment.

```text
192.168.17.128 justice.mysql
192.168.17.128 justice.rabbitmq
192.168.17.128 justice.redis
```

### MySQL

- Install MySQL Server:
```bash
apt install mariadb-server
mysql_secure_installation
```

- Create database for our app:
```mysql
CREATE DATABASE www_justice_plus;
CREATE USER 'root'@'%' IDENTIFIED BY '${PASSWORD}';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
```

- Update `/etc/mysql/mysql.conf.d/mysqld.cnf`, and replace `${justice.mysql}` with your own IP address:
<pre>
+ bind-address = <b><i>${justice.mysql}</i></b>
</pre>

- Restart MySQL Server:
```bash
/etc/init.d/mysql restart
```

### RabbitMQ

- Install RabbitMQ:
```bash
apt install -y rabbitmq-server
```

- Enable `rabbitmq_management` plugin, and replace ${password} with your own password:
```bash
rabbitmq-plugins enable rabbitmq_management
rabbitmqctl add_user justice ${password}
rabbitmqctl set_user_tags justice administrator
rabbitmqctl set_permissions -p / justice ".*" ".*" ".*"
```

- Update `/etc/rabbitmq/rabbitmq-env.conf`, and replace `${justice.rabbitmq}` with your own IP address:
<pre>
NODENAME=justice
NODE_IP_ADDRESS=<b><i>${justice.rabbitmq}</i></b>
NODE_PORT=5672
</pre>

- Restart RabbitMQ:
```bash
/etc/init.d/rabbitmq-server restart
```

### Redis

- Install Redis
```bash
apt install -y redis-server
```

- Update `/etc/redis/redis.conf`, and replace `${justice.redis}` with your own IP address:
<pre>
bind <b><i>${justice.redis}</i></b>
</pre>

- Restart Redis:
```bash
/etc/init.d/redis-server restart
```

## Frontend

### PHP

- Install PHP 7.3:
```bash
apt install -y php7.3-xml php7.3-mbstring php7.3-zip php7.3-mysql php7.3 \
php7.3-opcache php7.3-json php7.3-xmlrpc php7.3-curl php7.3-bz2 php7.3-cgi \
php7.3-cli php7.3-fpm php7.3-gmp php7.3-common php7.3-bcmath php7.3-gd
```

- Composer
```bash
curl -o /usr/local/bin/composer https://getcomposer.org/composer.phar
chmod +x /usr/local/bin/composer
```

### Init project

```bash
mkdir -p /var/www
git clone https://github.com/justice-oj/frontend.git /var/www/justice.plus

cd /var/www/justice.plus
composer global require "fxp/composer-asset-plugin:^1.4.2"
composer install
```

#### **For Development environment**

- Init yii2 project
```bash
cd /var/www/justice.plus
php init --env=Development --overwrite=All
php yii migrate
php yii fixture "*"
php yii serve --docroot="www/web" localhost:8888
php yii serve --docroot="admin/web" localhost:9999
```

#### **For Production environment**

- Update config `justice.plus/environments/prod/common/config/main-local.php` for MySQL / Redis and RabbitMQ:
<pre>
[
    'components' => [
        'db' => [
            'class' => 'yii\db\Connection',
            'dsn' => 'mysql:host=<b><i>${justice.mysql}</i></b>;dbname=www_justice_plus',
            'username' => '<b><i>root</i></b>',
            'password' => '<b><i>${justice.mysql.password}</i></b>',
            'charset' => 'utf8',
        ],
        'redis' => [
            'class' => 'yii\redis\Connection',
            'hostname' => '<b><i>${justice.redis}</i></b>',
            'port' => 6379,
            'database' => 15,
        ],
        'rabbitMQ' => [
            'class' => \common\components\queue\BasicRabbitMQProducer::class,
            'host' => '<b><i>${justice.rabbitmq}</i></b>',
            'port' => 5672,
            'user' => 'justice',
            'password' => '<b><i>${justice.rabbitmq.password}</i></b>',
            'queueName' => 'justice',
            'exchangeName' => 'justice',
            'exchangeType' => 'topic',
        ],
    ],
];
</pre>

- Init yii2 project
```bash
cd /var/www/justice.plus
php init --env=Production --overwrite=All
php yii migrate
chown -R www-data:www-data /var/www/justice.plus
```

- Install Nginx
```bash
apt install -y nginx
```

- Edit `/etc/php/7.3/fpm/pool.d/www.conf` to change PHP-FPM from listening on unix socket to listening on TCP/IP port:
```ini
- listen = /run/php/php7.3-fpm.sock
+ listen = 127.0.0.1:9000
``` 

- Sample server block of `/etc/nginx/conf.d/justice.plus.conf`
```
server {
    listen 80;
    server_name admin.justice.plus;

    access_log logs/admin_justice_plus.access.log main;
    error_log logs/admin_justice_plus.error.log warn;

    root /var/www/justice.plus/admin/web;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php$is_args$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}

server {
    listen 80;
    server_name www.justice.plus;

    access_log logs/www_justice_plus.access.log main;
    error_log logs/www_justice_plus.error.log warn;

    root /var/www/justice.plus/www/web;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php$is_args$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

- Start nginx and PHP-fpm
```bash
/etc/init.d/nginx start
/etc/init.d/php7.3-fpm start
```

## Dispatcher

- Install openjdk-11
```bash
apt install openjdk-11-jdk
```

- Install Tomcat 9
```bash
cd /opt
wget http://apache.cs.utah.edu/tomcat/tomcat-9/v9.0.22/bin/apache-tomcat-9.0.22.tar.gz
tar zxf apache-tomcat-9.0.22.tar.gz
mv apache-tomcat-9.0.22 dispatcher
```

- Update `conf/catalina.properties`, ${ENV} must be in one of `local`, `test` or `prod`:
```ini
+ spring.profiles.active=${ENV}
```

- Update `application-*.properties` for each environment.

- Deploy .war and start Tomcat
```bash
mkdir -p /var/log/justice/code
./bin/startup.sh
```

## Sandbox

- Make sure that Golang version >= 1.12

- Init Project(compile binaries, enable [cgroup auto clean-up](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/sec-common_tunable_parameters), etc)
```bash
cd /opt
git clone https://github.com/justice-oj/sandbox.git
cd /opt/sandbox/
./build.sh
```

- C/C++ language support
```bash
apt install -y gcc g++
```
