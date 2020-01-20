# mysql常用命令

## 备份全库

```
mysqldump -u root -p --all-databases > backup.sql
```

## 还原

```
mysql -u root -p < C:\backup.sql
```

## 创建用户

```
CREATE USER 'test'@'localhost' IDENTIFIED BY 'test1234';
```

## 授权

```
GRANT ALL PRIVILEGES ON *.* TO 'test'@'%' IDENTIFIED BY 'test1234' ;
```

## 查询所有用户

```
SELECT User, Host, Password FROM mysql.user;
```

## 修改密码

```
set password for 'root'@'localhost' = password('123');
```

## 忘记root密码

* 编辑/etc/my.cnf配置文件,在\[mysqld\]下添加skip-grant-tables
* 重启mysql服务：service mysqld restart

* 执行mysql命令进入mysql命令行,修改root用户密码

```
UPDATE mysql.user SET Password=PASSWORD('新密码') where USER='root';
flush privileges;
exit
```

* 编辑/etc/my.cnf配置文件,注释掉skip-grant-tables，重启mysql服务

## 查看空闲连接

```
show processlist;
```


