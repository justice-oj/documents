## RabbitMQ

Install [RabbitMQ][1] with yum:
```bash
# yum install -y rabbitmq-server
```

Enable `rabbitmq_management` plugin:
```bash
rabbitmq-plugins enable rabbitmq_management
```

Add user `justice` for justice-oj:
```bash
rabbitmqctl add_user justice <PASSWORD>
rabbitmqctl set_user_tags justice administrator
rabbitmqctl set_permissions -p / justice ".*" ".*" ".*"
```

Update config file `/etc/rabbitmq/rabbitmq.config`:
```
[
  {rabbit, [
    {tcp_listeners, [{"<IP_ADDR>", 5672}]}
  ]},
  {rabbitmq_management, [
    {listener, [{port, 15672}, {ip, "<IP_ADDR>"}]}
  ]}
].
```

Start RabbitMQ Server:
```bash
systemctl start rabbitmq-server
```

  [1]: https://www.rabbitmq.com/