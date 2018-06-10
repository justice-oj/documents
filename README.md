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

For now [Justice](https://www.justice.plus/) is tested on [Ubuntu 18.04 LTS](http://releases.ubuntu.com/18.04/) only, migrating the following instructions to any other dist may cause security issues.

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
apt install -y mysql-server
mysql_secure_installation
```

- Create database for our app:
```mysql
CREATE DATABASE www_justice_plus;
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '${justice.mysql}';
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
add-apt-repository ppa:chris-lea/redis-server
apt update -y
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

### Sentry

```bash
apt install -y make docker.io
```

- Follow [this instruction](https://docs.sentry.io/server/installation/docker/) to install Sentry.

## Frontend

### PHP

- Install PHP 7.2:
```bash
add-apt-repository -y ppa:ondrej/php
apt update -y
apt install -y php7.2-xml php7.2-mbstring php7.2-zip php7.2-mysql php7.2 php7.2-opcache php7.2-json php7.2-xmlrpc php7.2-curl php7.2-bz2 php7.2-cgi php7.2-cli php7.2-fpm php7.2-gmp php7.2-common php7.2-bcmath php7.2-gd
```

- Composer
```bash
curl -o /usr/local/bin/composer https://getcomposer.org/composer.phar && chmod +x /usr/local/bin/composer
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

- Update config `justice.plus/environments/dev/common/config/params-local.php` for sentry:
<pre>
[
    'sentryDSN' => '<b><i>${sentry_url}</i></b>',
];
</pre>

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

- Update config `justice.plus/environments/prod/common/config/params-local.php` for sentry:
<pre>
[
    'sentryDSN' => '<b><i>${sentry_url}</i></b>',
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

- Edit `/etc/php/7.2/fpm/pool.d/www.conf` to change PHP-FPM from listening on unix socket to listening on TCP/IP port:
```ini
- listen = /run/php/php7.1-fpm.sock
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
/etc/init.d/php7.2-fpm start
```

## Dispatcher

- Install JDK 8
```bash
cd /opt
wget --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u172-b11/a58eab1ec242421181065cdc37240b08/jdk-8u172-linux-x64.tar.gz
tar zxf jdk-8u172-linux-x64.tar.gz
ln -s jdk1.8.0_172 jdk
```

- Install Tomcat 8.5
```bash
cd /opt
wget http://www-us.apache.org/dist/tomcat/tomcat-8/v8.5.31/bin/apache-tomcat-8.5.31.tar.gz
tar zxf apache-tomcat-8.5.31.tar.gz
mv apache-tomcat-8.5.31 tomcat-justice-dispatcher
```

- Update `conf/catalina.properties`, ${ENV} must be in one of `local`, `test` or `prod`:
```ini
+ sentry.dsn=${DSN}
+ spring.profiles.active=${ENV}
```

- Update `application-*.properties` for each environment.

- Deploy .war and start Tomcat
```bash
mkdir -p /var/log/justice/code
./bin/startup.sh
```

## Sandbox

- Install Golang
```bash
apt install -y golang-go
```

- Init Project(compile binaries, enable [cgroup auto clean-up](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/sec-common_tunable_parameters), etc)
```bash
go get github.com/justice-oj/sandbox
cd ${GOPATH}/src/github.com/justice-oj/sandbox/
./build.sh
```

- C/C++ language support
```bash
apt install -y gcc g++
```
