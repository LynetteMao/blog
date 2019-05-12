---
title: 搭建git服务器
tag: 环境搭建
categories: operate
---

- [背景](#%E8%83%8C%E6%99%AF)
- [环境](#%E7%8E%AF%E5%A2%83)
- [搭建步骤](#%E6%90%AD%E5%BB%BA%E6%AD%A5%E9%AA%A4)
  - [租云服务](#%E7%A7%9F%E4%BA%91%E6%9C%8D%E5%8A%A1)
  - [安装mysql5.7](#%E5%AE%89%E8%A3%85mysql57)
  - [安装`ssh`](#%E5%AE%89%E8%A3%85ssh)
  - [安装`git`](#%E5%AE%89%E8%A3%85git)
  - [安装`gogs`](#%E5%AE%89%E8%A3%85gogs)
  - [配置](#%E9%85%8D%E7%BD%AE)
- [踩过的坑](#%E8%B8%A9%E8%BF%87%E7%9A%84%E5%9D%91)
  - [mysql版本](#mysql%E7%89%88%E6%9C%AC)
  - [服务器端口问题](#%E6%9C%8D%E5%8A%A1%E5%99%A8%E7%AB%AF%E5%8F%A3%E9%97%AE%E9%A2%98)

<br>

## 背景
一开始是使用github进行远程多人合作，不得不说作为最大的开源社区，在合作这一方面做的真的很好。但是因为接的项目老师是商业版的，无奈只能寻找私人仓库，国内的那几家免费私人仓库合作人数都有限制，就开始琢磨着自己搭建一个git服务器。通过收集资料，发现`gitlab`配置要求过于高，不过算是github的代替品吧。穷学生还是找自己能租得起的服务器能承受的起的，然后就发现了`gogs`，但是还没试过合作的功能怎么样。
<!-- more -->

<br>

## 环境
- 阿里云（Centos7）
- gogs
- mysql5.7
- ssh

<br>

## 搭建步骤

<br>

### 租云服务
   这个就不细说了，我用的是阿里云，不过感觉限制好多（好多设置都不能直接通过命令行修改，还要去控制台），之前用的是digitalocean,用的就很爽了，就是速度会慢一点。

<br>

### 安装mysql5.7
   这个是我踩坑踩的最多的一个，记住版本还是安装mysql5.7的会省去很多麻烦
   进入mysql的官网，点击 *Yum Repository* ，点击 *A Quick Guide to Using the MySQL Yum Repository* 就有详细的教程了（发现真的官网的文档绝对的最详细），不过我还是记录一下我的步骤：

*  下载安装包   
    在官网是下好来，通过SFTP会话传输到服务器上
* 安装rpm包
    ```shell
    shell> rpm -Uvh mysql80-community-release-el6-n.noarch.rpm

    #查看已安装的包
    shell> yum repolist all | grep mysql
    ```
* 选择要安装的包（就是因为忽略了这个，然后安装成了8.0）
  enabled为1的包就是等会yum会安装的包，打开文件选择自己要安装的版本，吧enabled改为1，其他的改为0，就行了。
    ```shell
    vi  /etc/yum.repos.d/mysql-community.repo

    ## Enable to use MySQL 5.7
    [mysql57-community]
    name=MySQL 5.7 Community Server
    baseurl=http://repo.mysql.com/yum/mysql-5.7-community/el/6/$basearch/
    enabled=1      
    gpgcheck=1
    gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql

    ###查看即将安装的包
    shell> yum repolist enabled | grep mysql
    ```
* 开始安装、开启服务
    ```
    shell> sudo yum install mysql-community-server

    shell> sudo systemctl start mysqld.service
    ```
*  查找、修改临时密码
    ```shell
    #查看临时密码
    shell> sudo grep 'temporary password' /var/log/mysqld.log

    #登录
    shell> mysql -uroot -p

    #修改密码
    mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';
    ```
修改密码的时候，简单的密码都不能用，因为policy有限制，除非密码有大小写还有字符，要使用简单密码就要降低police的值
```shell
#如果要修改简单密码
mysql>  ALTER USER 'root'@'localhost' IDENTIFIED BY '123456';

ERROR 1819 (HY000): Your password does not satisfy the current policy requirements   

#查看密码策略

mysql> SHOW VARIABLES LIKE 'validate_password%';
+--------------------------------------+-------+
| Variable_name                        | Value |
+--------------------------------------+-------+
| validate_password_check_user_name    | OFF   |
| validate_password_dictionary_file    |       |
| validate_password_length             | 8     |
| validate_password_mixed_case_count   | 1     |
| validate_password_number_count       | 1     |
| validate_password_policy             |MEDIUM |
| validate_password_special_char_count | 1     |
+--------------------------------------+-------+
7 rows in set (0.00 sec)

#修改密码策略
mysql> set global validate_password_policy=0;
Query OK, 0 rows affected (0.00 sec)

mysql> SHOW VARIABLES LIKE 'validate_password%';
+--------------------------------------+-------+
| Variable_name                        | Value |
+--------------------------------------+-------+
| validate_password_check_user_name    | OFF   |
| validate_password_dictionary_file    |       |
| validate_password_length             | 8     |
| validate_password_mixed_case_count   | 1     |
| validate_password_number_count       | 1     |
| validate_password_policy             | LOW   |
| validate_password_special_char_count | 1     |
+--------------------------------------+-------+
7 rows in set (0.00 sec)

mysql> set global validate_password_length=4;
Query OK, 0 rows affected (0.00 sec)

mysql> SHOW VARIABLES LIKE 'validate_password%';
+--------------------------------------+-------+
| Variable_name                        | Value |
+--------------------------------------+-------+
| validate_password_check_user_name    | OFF   |
| validate_password_dictionary_file    |       |
| validate_password_length             | 4     |
| validate_password_mixed_case_count   | 1     |
| validate_password_number_count       | 1     |
| validate_password_policy             | LOW   |
| validate_password_special_char_count | 1     |
+--------------------------------------+-------+
7 rows in set (0.00 sec)


#再修改密码
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY '12345678';

Query OK, 0 rows affected (0.04 sec)
```
这样的话呢，mysql就算安装完成了！

<br>

### 安装`ssh`    

centos默认安装了，查看有没有安装，并开启22端口即可
```shell
#安装ssh（centos默认安装，并开启22端口号）
#查看是否安装
[root@izwz96ngzxfrp8l0los1suz ~]# rpm -qa | grep ssh
openssh-clients-7.4p1-16.el7.x86_64
openssh-server-7.4p1-16.el7.x86_64
libssh2-1.4.3-10.el7_2.1.x86_64
openssh-7.4p1-16.el7.x86_64

#开启服务
[root@izwz96ngzxfrp8l0los1suz ~]# service sshd start
Redirecting to /bin/systemctl start sshd.service

#查看端口号时否开启
[root@izwz96ngzxfrp8l0los1suz ~]# netstat -ntpl |grep 22
    tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      2990/sshd
```

<br>

### 安装`git`

```shell
    yum install git
```

### 安装`gogs`

* 下载安装包，解压
   第一次安装的时候报错了，原因是下载的是32位版本的，在64位的机器上不能运行。
    ```
    tar xzvf gogs_0.11.43_linux_amd64.tar.gz
    ```
* 创建系统用户
    ```shell
    #添加用户
    useradd git

    #设置密码
    passwd git
    ```
* 数据准备(root)
   ```shell
   cd gogs/scripts/
   mysql -u root -p < mysql.sql

   #建立数据库用户
   create user 'gogs'@'localhost' identified by 'gogs';
   grant all privileges on gogs.* to 'gogs'@'localhost';

   #开启远程连接（不确定）
   GRANT ALL PRIVILEGES ON *.* TO 'gogs'@'%' IDENTIFIED BY 'password' WITH GRANT OPTION;
   ```
   如果配置的时候一直提示数据库连接失败，一般就是没有开启远程连接，还有如果是8.0的，密码加密策略改了，所以连接不上。
   ```shell
   mysql> use mysql;

   #表格上localhost表示只允许本地访问
   mysql> select host, user, authentication_string, plugin from user; 
   mysql> GRANT ALL ON *.* TO 'gogs'@'%';
   mysql> flush privileges;
   ```   
* 要去阿里云控制台开放规则，放行端口，不然连接不上
* 运行
   ```shell
   cd gogs
   ./gogs web
   ```

<br>

### 配置

   跑`./gogs web`的时候会显示日志，打开浏览器输入：[http://服务器ip:3000](http://服务器ip:3000)。
   根据网页信息进行配置即可

<br>

## 踩过的坑

### mysql版本
  
* mysql的安装方式错误
    ```shell
    yum install mysql
    yum -y install mysql mysql-server mysql-devel

    #但是启动不了服务器
    systemctl start mysqld
    error reading information on service mysqld: No such file or directory

    #发现安装的是mariadb
    Installed:
    mariadb.x86_64 1:5.5.41-2.el7_0        
    ```
* 一开始安装的是5.6，但是gogs要求是5.7以上，所以就只能卸载重新安装咯，卸载后重新安装可能会冲突，所以要卸载冲突项。错因：直接按照网上的教程，没仔细看什么安装包
    ```shell
    #卸载mysql 5.6
    yum remove mysql

    #下载更高版本的rpm包
    rpm -Uvh mysql80-community-release-el6-n.noarch.rpm
    error: Failed dependencies:
        mysql-community-release conflicts with mysql57-community-release-el7-7.noarch
    #报错，然后就卸载了冲突项

    #查找已经安装的包
    rpm -qa | grep mysql

    #卸载
    rpm -e --nodeps mysql-community-release-el7-5.noarch

    #重新安装
    rpm -Uvh mysql80-community-release-el6-n.noarch.rpm
    ```
* mysql8.0刚安装好，如果没有更改密码会报错，密码太简单了也会报错
    ```shell
    #查看mysql版本
    [root@izwz96ngzxfrp8l0los1suz ~]# mysql -V
    mysql  Ver 8.0.11 for Linux on x86_64 (MySQL Community Server - GPL)
    
    #查看初始密码
    vim /var/log/mysqld.log 
    2018-07-14T09:13:37.318628Z 0 [System] [MY-013169] [Server] /usr/sbin/mysqld (mysqld 8.0.11) initializing of server in progress as process 2735
    2018-07-14T09:13:40.567571Z 5 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: MhU%*C-l:5oe
    2018-07-14T09:13:42.058117Z 0 [System] [MY-013170] [Server] /usr/sbin/mysqld (mysqld 8.0.11) initializing of server has completed
    2018-07-14T09:13:44.346383Z 0 [System] [MY-010116] [Server] /usr/sbin/mysqld (mysqld 8.0.11) starting as process 2782
    2018-07-14T09:13:45.016804Z 0 [Warning] [MY-010068] [Server] CA certificate ca.pem is self signed.
    2018-07-14T09:13:45.062845Z 0 [System] [MY-010931] [Server] /usr/sbin/mysqld: ready for connections. Version: '8.0.11'  socket: '/var/lib/mysql/mysql.sock'  port: 3306  MySQL Community Server - GPL.
    
    #修改密码
    #不修改密码，随便输入什么语句都报错
    mysql> SHOW DATABASES;ERROR 1820 (HY000): You must reset your password using ALTER USER statement before executing this statement.

    #修改密码太简单，也报错
    mysql>  ALTER USER 'root'@'localhost' IDENTIFIED BY '123456'
        -> ;

    ERROR 1819 (HY000): Your password does not satisfy the current policy requirements

    #降低police的值
    mysql> SET GLOBAL validate_password.policy=0;Query OK, 0 rows affected (0.00 sec)

    #再修改密码
    mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY '12345678';
    
    Query OK, 0 rows affected (0.04 sec)
    ```
* mysql8.0密码加密策略和5.7不一样，在网页上对gogs进行配置的时候，怎么输入都报错，原因就是加密策略。
   解决办法：
   * 更改加密策略（试过了，但是失败了，最后在服务器上都登入不了mysql）
   * 卸载8.0，重新安装5.7   
   * 卸载如5.6，就不写了

<br>

### 服务器端口问题

  执行`./gogs web`后怎样都打不开网页，就觉得应该是端口问题，或者防火墙的问题
* 查看所有打开的端口
    ```
   firewall-cmd --zone=public --list-ports
    ```
* 没有这个端口，就开启这个端口
    ```
   firewall-cmd --zone=public --add-port=3000/tcp --permanent
    ```
* 重新载入
    ```
    firewall-cmd --reload
    ```
* 但是还是不行，后来发现是阿里云开放端口是要去控制台添加规则（没有提示，很不人性化），在控制台加入了规则之后就可以访问了

<br>