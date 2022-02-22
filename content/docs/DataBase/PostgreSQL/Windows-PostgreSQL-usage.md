---
title: "Windows PostgreSQL 使用"
weight: 2
typora-root-url: ..\..\..\..\static
---

直接官网下载安装，安装时记得记下安装路径。

根目录如下图所示，bin下是 `PostgreSQL` 的命令行控制程序，`pgAdmin4` 下是 `PostpreSQL` 的可视化管理程序。

![image-20210715095216419](/images/Windows-PostgreSQL-usage/assets/image-20210715095216419.png)

## 添加 bin 到环境变量之中

记下到 bin 文件夹的绝对路径， 我这里是 `D:\Program Files\PostgreSQL\12\bin` ，添加到系统变量和用户变量的 `Path` 之中。

![image-20210715100017968](/images/Windows-PostgreSQL-usage/assets/image-20210715100017968.png)

然后打开 Terminal ，输入

```powershell
pg_ctl.exe start -D "D:\Program Files\PostgreSQL\12\data"
```

然后打开 `pgAdmin` ， 输入 host 和 Password 。

![image-20210715115840448](/images/Windows-PostgreSQL-usage/assets/image-20210715115840448.png)

## 角色 "postgres" 不存在

Terminal 中出现角色不存在，导致无法正常连接到postgre服务。

![image-20210715115940981](/images/Windows-PostgreSQL-usage/assets/image-20210715115940981.png)

使用管理员权限打开 `Powershell` ， 输入

```powershell
 pg_ctl.exe register -N "pgsql"  -D "D:\Program Files\PostgreSQL\12\data"
```

将 `postgresql` 注册为服务

