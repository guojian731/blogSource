---
title: rabbitmq-4-rabbitmq中常用设置语句
date: 2018-01-24 15:13:05
tags: [rabbitMQ]
---

![jvm装载步骤](rabbitmq-4-rabbitmq中常用设置语句/r4.png)

我们在之前讲解了很多关于rabbitmq的构成以及设置语句，其实也很多，所以单独写一篇来记录。


# 启动/关闭rabbitmq

启动服务 ./rabbitmq-server 

守护程序启动服务 ./rabbitmq-server -detached

启动守护进程会显示警告：Warning: PID file not written; -detached was passed.在后台启动服务器进程。请注意，这将导致pid不被写入到pid文件中。

停止服务 ./rabbitmqctl stop

停止远程节点 ./rabbitmqctl stop -n rabbit@192.168.14.130

# 用户

添加用户和密码： ./rabbitmqctl add_user admin admin 

查看用户列表： ./rabbitmqctl list_users

修改用户密码： ./rabbitmqctl change_password admin admintest

设置/修改用户在test_vhost的权限(配置、写、读)：
``` xml 
./rabbitmqctl set_permissions -p test_vhost \admin ".*" ".*" ".*"
```

# vhost 

添加vhost: ./rabbitmqctl add_vhost test_vhost 

查看vhost列表: ./rabbitmqctl list_vhosts 

查看test_vhost下的用户权限：./rabbitmqctl list_permissions -p test_vhost

删除用户权限：./rabbitmqctl clear_permissions -p test_vhost admin

# 队列

显示test_vhost下的队列：./rabbitmqctl list_queues -p test_vhost

# 交换器

显示test_vhost下的交换器及其绑定：./rabbitmqctl list_exchanges -p test_vhost

查看交换器与队列的绑定关系: ./rabbitmqctl list_bindings -p test_vhost





