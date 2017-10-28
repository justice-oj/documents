# Deployment Guide

This is a deployment guide for [Justice](https://www.justice.plus/) online judge.

![Architecture](/images/architecture.png)

The picture above describes the main judging process of `Justice`:

- user submits a piece of code, `justice-frontend` then updates MySQL and puts the judge submission to RabbitMQ;

- `justice-dispatcher` fetches the judge submission from RabbitMQ, delegates it to `justice-sandbox`;

- `justice-sandbox` runs the submitted code in a jailed environment, and returns the judge result in a json-encoded string;

- `justice-dispatcher` then updates MySQL with the result decoded from `justice-sandbox`.

For now [Justice](https://www.justice.plus/) is tested on [Ubuntu 16.04 LTS](http://releases.ubuntu.com/16.04/) only, migrating the following instructions to any other dist may cause security issues.

## Components

### MySQL

- Follow [this instruction](https://www.percona.com/doc/percona-server/LATEST/installation/apt_repo.html#installing-percona-server-on-debian-and-ubuntu) to install Percona Server.

- Create database for our app:
```
> CREATE DATABASE www_justice_plus;
```

- Update `/etc/mysql/percona-server.conf.d/mysqld.cnf`:
<pre>
+ bind-address = <b><i>192.168.216.128</i></b>
</pre>

- Restart Percona Server:
```
# /etc/init.d/mysql restart
```

### RabbitMQ

- Install RabbitMQ:
```bash
# apt install -y rabbitmq-server
```

- Enable `rabbitmq_management` plugin, and replace ${PASSWORD} with your own password:
```bash
# rabbitmq-plugins enable rabbitmq_management
# rabbitmqctl add_user justice ${PASSWORD}
# rabbitmqctl set_user_tags justice administrator
# rabbitmqctl set_permissions -p / justice ".*" ".*" ".*"
```

- Update `/etc/rabbitmq/rabbitmq-env.conf`:
<pre>
NODENAME=justice
NODE_IP_ADDRESS=<b><i>192.168.216.128</i></b>
NODE_PORT=5672
</pre>

- Restart RabbitMQ:
```bash
# /etc/init.d/rabbitmq-server restart
```

### Redis

- Install Redis
```bash
# apt install -y software-properties-common
# add-apt-repository ppa:chris-lea/redis-server
# apt update -y
# apt install -y redis-server
```

- Update `/etc/redis/redis.conf`:
<pre>
bind <b><i>192.168.216.128</i></b>
</pre>

- Restart Redis:
```bash
# /etc/init.d/redis-server restart
```

### Sentry

```bash
# apt install -y make docker.io
```

- Follow [this instruction](https://docs.sentry.io/server/installation/docker/) to install Sentry.

## Frontend

### PHP

- PHP 7.1:
```bash
# apt install -y python-software-properties
# add-apt-repository -y ppa:ondrej/php
# apt update -y
# apt install -y php7.1-xml php7.1-mbstring php7.1-zip php7.1-mysql php7.1 php7.1-opcache php7.1-json php7.1-xmlrpc php7.1-curl php7.1-bz2 php7.1-cgi php7.1-cli php7.1-fpm php7.1-gmp php7.1-common php7.1-bcmath php7.1-mcrypt php7.1-gd
```

- Composer
```
# curl -o /usr/local/bin/composer https://getcomposer.org/composer.phar && chmod +x /usr/local/bin/composer
```

### Init project

```bash
# mkdir -p /var/www
# git clone https://github.com/liupangzi/justice-frontend.git /var/www/justice.plus

# cd /var/www/justice.plus
# composer global require "fxp/composer-asset-plugin:^1.2.0"
# composer install
```

#### **For Development environment**

- Update config `justice.plus/environments/dev/common/config/main-local.php` for MySQL / Redis and RabbitMQ:
<pre>
[
    'components' => [
        'db' => [
            'class' => 'yii\db\Connection',
            'dsn' => 'mysql:host=<b><i>192.168.216.128</i></b>;dbname=www_justice_plus',
            'username' => '<b><i>root</i></b>',
            'password' => '<b><i>xaiTIVP7kB$oHuJecEooq#YsziVvVAzW</i></b>',
            'charset' => 'utf8',
        ],
        'redis' => [
            'class' => 'yii\redis\Connection',
            'hostname' => '<b><i>192.168.216.128</i></b>',
            'port' => 6379,
            'database' => 15,
        ],
        'rabbitMQ' => [
            'class' => \common\components\queue\BasicRabbitMQProducer::class,
            'host' => '<b><i>192.168.216.128</i></b>',
            'port' => 5672,
            'user' => 'justice',
            'password' => '<b><i>justice</i></b>',
            'queueName' => 'justice',
            'exchangeName' => 'justice',
            'exchangeType' => 'topic',
        ],
    ],
];
</pre>

- Update config `justice.plus/environments/dev/common/config/params-local.php` for sentry:
<pre>
[
    'sentryDSN' => '<b><i>http://f3ed86b6e6bd4c6a8c7f1ebf77d65dff:b616b38dc3cd438fbc4ffd789bed18d4@192.168.216.128:12000/2</i></b>',
];
</pre>

- Init yii2 project
```bash
# cd /var/www/justice.plus
# php init --env=Development --overwrite=All
# php yii migrate
# php yii fixture "*"
# chown -R www-data:www-data /var/www/justice.plus
# php yii serve --docroot="www/web" localhost:8888
# php yii serve --docroot="admin/web" localhost:9999
```

#### **For Production environment**

- Update config `justice.plus/environments/prod/common/config/main-local.php` for MySQL / Redis and RabbitMQ:
<pre>
[
    'components' => [
        'db' => [
            'class' => 'yii\db\Connection',
            'dsn' => 'mysql:host=<b><i>127.0.0.1</i></b>;dbname=www_justice_plus',
            'username' => '<b><i>root</i></b>',
            'password' => '<b><i>xaiTIVP7kB$oHuJecEooq#YsziVvVAzW</i></b>',
            'charset' => 'utf8',
        ],
        'redis' => [
            'class' => 'yii\redis\Connection',
            'hostname' => '<b><i>127.0.0.1</i></b>',
            'port' => 6379,
            'database' => 15,
        ],
        'rabbitMQ' => [
            'class' => \common\components\queue\BasicRabbitMQProducer::class,
            'host' => '<b><i>127.0.0.1</i></b>',
            'port' => 5672,
            'user' => 'justice',
            'password' => '<b><i>justice</i></b>',
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
    'sentryDSN' => '<b><i>http://c310ce509417482a98ecf42aaa2fce3d:056872ff2da64d93b121fa615899406e@127.0.0.1:12000/2</i></b>',
];
</pre>

- Init yii2 project
```bash
# cd /var/www/justice.plus
# php init --env=Production --overwrite=All
# php yii migrate
# chown -R www-data:www-data /var/www/justice.plus
```

- Install Openresty
```bash
# wget -qO - https://openresty.org/package/pubkey.gpg | apt-key add -
# apt -y install software-properties-common
# add-apt-repository -y "deb http://openresty.org/package/ubuntu $(lsb_release -sc) main"
# apt update -y
# apt install -y openresty
```

- Edit `/etc/php/7.1/fpm/pool.d/www.conf` to change PHP-FPM from listening on unix socket to listening on TCP/IP port:
```bash
- listen = /run/php/php7.1-fpm.sock
+ listen = 127.0.0.1:9000
``` 

- Sample server block of `/usr/local/openresty/nginx/conf/nginx.conf`
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
# /etc/init.d/openresty start
# /etc/init.d/php7.1-fpm start
```

## Dispatcher

- Dispatcher requires [JDK8](http://www.oracle.com/technetwork/pt/java/javase/downloads/jdk8-downloads-2133151.html) and [Tomcat 8](https://tomcat.apache.org/download-80.cgi)

- Update `conf/catalina.properties`, ${ENV} must be in one of `local`, `test` or `prod`:
```
+ sentry.dsn=${DSN}
+ spring.profiles.active=${ENV}
```

- Update `application-*.properties` for each environment.

- Deploy .war and start Tomcat
```bash
# mkdir -p /var/log/justice/code
# bin/startup.sh
```

## Sandbox

- Sandbox requires [Go](https://golang.org/doc/install)

- Checkout project in `/opt`
```bash
# cd /opt
# git clone https://github.com/liupangzi/justice-sandbox.git
```

- Init(compile binaries, enable [cgroup auto clean-up](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/sec-common_tunable_parameters), etc)
```bash
# cd /opt/justice-sandbox
# ./build.sh
```

- C/C++ language support
```bash
# apt install -y gcc g++
```