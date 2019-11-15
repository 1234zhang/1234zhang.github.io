---
layout: '[layout]'
title: mongodb 相关配置
date: 2019-11-15 13:19:28
tags:
    - mongodb
categories:
    - 中间件
---
### mongodb社区版本 ubuntu安装(ubuntu 18.04, mongodb 4.0.13)
`mongodbDB.inc`维护的包是`mongodb-org`，而不是`mongodb`，我们要确保系统没有安装mongodb，否则会发生包冲突。

**导入公钥**
```
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 9DA31620334BD75D9DCB49F368818C72E52529D4
```

**源列表添加新的仓库**
```
echo "deb [ arch=amd64 ] https://repo.mongodb.org/apt/ubuntu $(lsb_release -cs)/mongodb-org/4.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.0.list
```
**更新源**
```
sudo apt update
```

**安装最新版mongodb**
```
sudo apt install -y mongodb-org
```

### 启动mongodb

**运行mongodb**
```
sudo service mongod start
```
**验证mongodb是否启动**
```
sudo cat /var/log/mongodb/mongod.log
```
![avatar](https://brandonxcc.top/mongodb.png)

如果出现[initandlisten] waiting for connections on port 27017，说明正在运行

**重启mongodb**
```
sudo service mongod stop
sudo service mongod restart
```

**打开mongo shell**
```
mongo
```

### 修改mongodb相关配置
**添加用户**
首先进入mongodb shell操作界面
```
mongo
```
然后切换到admin数据库
```
use admin
```

输入下面的指令创建一个root用户， 而且密码为root, 可以按照自己的需求更改用户密码。

```
db.craeteUser(
    {
        user: "root",
        pwd: "root",
        roles: [{role: "userAdminAnyDatabase", db: "admin"}, "readWriteAnyDatabase"]
    }
)
```
密码可以使用密码生成器生成的随机密码，提高安全性。
其中的userAdminAnyDatabase角色赋予了root账户在该mongodb实例中，管理数据库的权限。

**开启数据库的权限验证**
更改配置文件
```
vim /etc/mongod.conf
```

在配置文件中，加上如下配置
```
security:
    authorization: enabled
```
然后重启mongodb
```
sudo service mongod stop
sudo service mongod restart
```

**添加单个数据库的用户权限**

首先切换到admin数据库，使用root验证权限
```
use admin
db.auth("root", "root")
```

创建一个数据库， 并添加一个拥有者test
```
use testDb
db.createUser({
    user: "test",
    pwd: "test",
    roles: [{role: "dbOwner", db: "test"}]
})
```

我们为test，赋予了dbOwner权限，保证test，用户有操作所有collection读和写的权限

**连接mongodb**
```
mongodb://test:test@localhost/testDb

用户名：test
密码：test
数据库：testDb
```

**远程连接mongodb**
修改配置将默认端口127.0.0.1，修改为0.0.0.0，此种情况下，允许所有机器访问mongodb
