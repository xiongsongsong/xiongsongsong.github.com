---
layout: post
title:  "在Docker中部署GitLab"
date:   2014-06-01
categories: gitlab
---

## 前言

越来越多的团队，开始使用GitLab来管理项目，相比于SVN来说，GitLab更能满足开发者的需求，界面也更加美观。然而，搭建好GitLab并非易事。你需要准备相当多的工作，即是如此，仍可能会遇到各种问题。
而如今Docker这种技术的发展，让GitLab的搭建简单了很多，你只需要专注于如何进行配置和使用即可。

本文翻译自：[https://index.docker.io/u/sameersbn/gitlab/](https://index.docker.io/u/sameersbn/gitlab/) ，翻译的目的源于2个好处，一是了解Docker中是如何部署应用的，由于GitLab的安装环境比较复杂，因此是一个很好的学习点。二来，也方便未来能自助的搭建一个GitLab应用，相当多的团队已经在这么做了。

--------------------------------

## 引言

通过Dockefile建立一个Gitlab容器的镜像。

## 版本

当前版本：6.9.2

## 硬件要求

### CPU

* 1核能满足个100用户，但响应速度可能会受到影响。
* 2核最多支持100个用户。
* 4核最多支持1,000个用户。
* 8核最多支持10,000个用户。

### 内存
* 如果使用512Mb内存，那么Gitlab会运行得非常缓慢，并且需要250Mb的虚拟内存。
* 768Mb是最低内存大小，我们並不推荐。
* 1GB内存能支持100个用户（仓库需要占用250Mb，否则Git需要使用交换空间）
* 2GB是推荐的大小，能支持1,000个用户。
* 4GB能支持10,000个用户。

### 存储
所需要的存储空间，很大程度上取决于Gitlab中有多少仓库需要存储。尽管如此，你应该准备2倍于实际所占大小的空间。因为[Gitlab satellites](https://github.com/gitlabhq/gitlabhq/blob/master/doc/install/structure.md)会建立每个仓库的额外副本。

如果未来想灵活的增加存储空间，请考虑使用LVM，当需要的时候，可以添加更多的硬盘。

除了支持本地存储以外，还支持挂载卷：网络文件（NFS）协议。卷可以位于一个文件服务器、网络存储（NAS）、存储区域网络（SAN）、亚马逊云服务（AWS），弹性存储（EBS）。

如果你有足够的内存和较新的CPU，那么Gitlab的速度取决于硬盘的寻道速度。使用高速硬盘（7200转/分钟）或固态硬盘（SSD）将提高Gitlab的响应速度。

### 浏览器支持
* Chrome(最新稳定版)
* Firefox（最新稳定版）
* Safari 7+（已知问题：html 5的必填字段无法工作）
* Opera（最新稳定版）
* IE 10+

## 安装

推荐从Docker index中拉取最新的镜像，将来更新镜像时将更加容易。这些镜像，都是被Docker所信任的：

{% highlight bash %}

docker pull sameersbn/gitlab:latest

{% endhighlight %}

从6.3.0版本开始，构建好的镜像都有Tag标记。因此，你可以拉取一个指定Tag标记的版本，例如：
{% highlight bash %}

docker pull sameersbn/gitlab:6.9.2

{% endhighlight %}
或者您也可以构建自己的镜像：

{% highlight bash %}

bash git clone https://github.com/sameersbn/docker-gitlab.git cd docker-gitlab docker build --tag="$USER/gitlab" .

{% endhighlight %}
## 快速起步

运行镜像：
{% highlight bash %}

docker run --name='gitlab' -i -t --rm \ -p 127.0.0.1:10022:22 -p 127.0.0.1:10080:80 \ -e "GITLAB_PORT=10080" -e "GITLAB_SSH_PORT=10022" \ sameersbn/gitlab:6.9.2

{% endhighlight %}

> 提醒：Gitlab运行起来需要等好几分钟，请耐心等待下。

使用浏览器打开 ```http://localhost:10080```，并试用默认用户名和密码登陆：
* 用户名：admin@local.host
* 密码：5iveL!fe

到此为止，你得到了一个运行中的GitLab应用，可以开始进行测试。如果你想在生产环境使用这个镜像，请继续阅读下方内容。


## 配置

### 数据存储

GitLab是代码托管应用，你一定不想容器在停止或被删除的时候，丢失掉你的数据。为了避免任何数据的丢失，你应该指定一个卷来存储：

* /home/git/data

{% highlight bash %}

mkdir /opt/gitlab/data docker run --name=gitlab -d \ -v /opt/gitlab/data:/home/git/data \ sameersbn/gitlab:6.9.2

{% endhighlight %}

### 数据库

GitLab使用数据库系统来存储产生的数据。

### MySQL

#### 集成MySQL服务
> **警告
> 集成的MySQL服务将很快从镜像中删除。
> 请使用mysql或postgresql相关容器替代，或者连接到外部的msyql或postgresql服务。
> 别怪我没有提醒你。

这个镜像被配置为使用MySQL数据库来存储数据。数据库连接还可以使用环境变量，如果没什么特别的设置，镜像将启动内部的MySQL服务并使用它。然而在这种情况下，存储在MySQL中的数据将在容器停止、或删除时丢失。为了避免这种情况，你应该将卷挂载在```/var/lib/mysql```：
{% highlight bash %}

mkdir /opt/gitlab/mysql docker run --name=gitlab -d \ -v /opt/gitlab/data:/home/git/data \ -v /opt/gitlab/mysql:/var/lib/mysql sameersbn/gitlab:6.9.2

{% endhighlight %}
这将确保数据库中的数据，不会在容器停止或重新启动后丢失。
#### 外部MySQL服务

GitLab可被配置为使用外部MySQL服务器，而不是集成的MySQL服务。启动GitLab镜像时，通过使用环境变量来指定数据库配置。

在启动GitLab镜像前，需要创建用户和```gitlab```数据库。

{% highlight sql %}
sql CREATE USER 'gitlab'@'%.%.%.%' IDENTIFIED BY 'password'; CREATE DATABASE IF NOT EXISTS `gitlabhq_production` DEFAULT CHARACTER SET `utf8` COLLATE `utf8_unicode_ci`; GRANT SELECT, LOCK TABLES, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER ON `gitlabhq_production`.* TO 'gitlab'@'%.%.%.%';
{% endhighlight %}
请确保数据库已经完成了初始化，然后配置```app:rake gitlab:steup```，并完成容器的启动。

假设MySQL服务器IP为：192.168.1.100

{% highlight bash %}

bash docker run --name=gitlab -i -t --rm \ -e "DB_HOST=192.168.1.100" -e "DB_NAME=gitlabhq_production" -e "DB_USER=gitlab" -e "DB_PASS=password" \ -v /opt/gitlab/data:/home/git/data \ sameersbn/gitlab:6.9.2 app:rake gitlab:setup

{% endhighlight %}
> 提醒: 仅在第一次运行时使用上述设置

这将完成GitLab数据库的初始化，现在，正常启动容器。
{% highlight bash %}

bash docker run --name=gitlab -d \ -e "DB_HOST=192.168.1.100" -e "DB_NAME=gitlabhq_production" -e "DB_USER=gitlab" -e "DB_PASS=password" \ -v /opt/gitlab/data:/home/git/data \ sameersbn/gitlab:6.9.2

{% endhighlight %}

#### 连接MySQL容器

你可以使用一个MySQL容器来与镜像连接。需要对这个mysql服务容器设置别名为```mysql```，同时与镜像关联起来。

如果mysql容器已经链接，则只需要设置DB_HOST和DB_PORT，就可以自动连接成功。当然，你也可以设置数据库的其他连接参数，如DB_NAME,DB_USER,DB_PASS等等。

为了演示连接到mysql容器，我们将使用[sameersbn/mysql](https://github.com/sameersbn/docker-mysql)镜像。当在生产环境中使用它时，你应该挂载卷来实现mysql的数据存储。详情请参见[docker-mysql的自述文件](https://github.com/sameersbn/docker-mysql/blob/master/README.md)。

首先，从docker index中拉取mysql的镜像：
{% highlight bash %}

docker pull sameersbn/mysql:latest

{% endhighlight %}

为了达到数据持久化，我们先创建并启动一个用于存储的mysql容器：
{% highlight bash %}

mkdir -p /opt/mysql/data docker run --name=mysql -d \ -v /opt/mysql/data:/var/lib/mysql \ sameersbn/mysql:latest

{% endhighlight %}

你现在应该有一个mysql服务器在运行了。默认情况下，```sameersbn/mysql```镜像默认拥有```root```权限，并允许用户远程连接到```172.17.%.%```，这意味着你可以从宿主以```root```权限登陆到mysql服务器。

现在，登陆到mysql服务器，并创建一个用户和GitLab应用的数据库。

```
mysql -uroot -h $(docker inspect mysql | grep IPAddres | awk -F'"' '{print $4}')
```

{% highlight sql %}
sql CREATE USER 'gitlab'@'172.17.%.%' IDENTIFIED BY 'password'; CREATE DATABASE IF NOT EXISTS `gitlabhq_production` DEFAULT CHARACTER SET `utf8` COLLATE `utf8_unicode_ci`; GRANT SELECT, LOCK TABLES, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER ON `gitlabhq_production`.* TO 'gitlab'@'172.17.%.%'; FLUSH PRIVILEGES;
{% endhighlight %}

现在，我们已经为GitLab创建了数据库，可以开始安装数据库表，这些操作是通过```app:rake gitlab:setup```命令，在启动容器时完成的：

```
docker run --name=gitlab -i -t --rm --link mysql:mysql \ -e "DB_USER=gitlab" -e "DB_PASS=password" \ -e "DB_NAME=gitlabhq_production" \ -v /opt/gitlab/data:/home/git/data \ sameersbn/gitlab:6.9.2 app:rake gitlab:setup
```

> 提示：只在初次运行时应用上述设置。

现在，我们准备启动GitLab：
```
docker run --name=gitlab -d --link mysql:mysql \ -e "DB_USER=gitlab" -e "DB_PASS=password" \ -e "DB_NAME=gitlabhq_production" \ -v /opt/gitlab/data:/home/git/data \ sameersbn/gitlab:6.9.2
```

#### PostgreSQL

##### 外部PostgreSQL 服务

GitLab镜像还支持外postgresql服务器，同外部mysql服务一样，也是通过设置环境变量的方式来控制的。

{% highlight sql %}
sql CREATE ROLE gitlab with LOGIN CREATEDB PASSWORD 'password'; CREATE DATABASE gitlabhq_production; GRANT ALL PRIVILEGES ON DATABASE gitlabhq_production to gitlab;
{% endhighlight %}

确保数据库已经完成初始化，并且在启动容器时设置好``` app:rake gitlab:setup```选项。

假设PostgreSQL服务器地址为```192.168.1.100```：

{% highlight bash %}

docker run --name=gitlab -i -t --rm \ -e "DB_TYPE=postgres" -e "DB_HOST=192.168.1.100" -e "DB_NAME=gitlabhq_production" -e "DB_USER=gitlab" -e "DB_PASS=password" \ -v /opt/gitlab/data:/home/git/data \ sameersbn/gitlab:6.9.2 app:rake gitlab:setup

{% endhighlight %}
> 提示: 仅在初次运行时使用上述设置。

这将初始化```gitlab```数据库。现在，数据库已经完成初始化，开始启动容器：

{% highlight bash %}

bash docker run --name=gitlab -d \ -e "DB_TYPE=postgres" -e "DB_HOST=192.168.1.100" -e "DB_NAME=gitlabhq_production" -e "DB_USER=gitlab" -e "DB_PASS=password" \ -v /opt/gitlab/data:/home/git/data \ sameersbn/gitlab:6.9.2

{% endhighlight %}

##### 链接到PostgreSQL容器

你可以将镜像连接到一个postgresql容器中的数据库。设置好postgresql服务器容器的别名，并让gitlab镜像链接到postgresql。

如果一个postgresql容器被链接过，只耍要设置DB_HOST和DB_PORT即可自动完成链接。不过，您可能还需要设置DB_NAME,DB_USER,DB_PASS等其他连接参数。

为了演示连接到PostgreSQL容器，我们将使用sameersbn/PostgreSQL镜像。如果你想在生产环境中使用PostgreSQL镜像，你应该挂载一个卷来作为数据存储的地方。详情请参见[docker-postgresql的自述文件](https://github.com/sameersbn/docker-postgresql/blob/master/README.md)。

首先，从docker index获取postgresql镜像：

{% highlight bash %}

docker pull sameersbn/postgresql:latest

{% endhighlight %}

为了让数据持久化，我们创建好存储的位置，并启动容器：

{% highlight bash %}

mkdir -p /opt/postgresql/data docker run --name=postgresql -d \ -v /opt/postgresql/data:/var/lib/postgresql \ sameersbn/postgresql:latest

{% endhighlight %}

现在，PostgreSQL服务器应该在运行中了。postgres的用户名和密码，可以在PostgreSQL镜像的日志中找到：

{% highlight bash %}
docker logs postgresql
{% endhighlight %}

现在，登陆到postgresql服务器，并创建一个用户和GitLab应用所需的数据库。

{% highlight bash %}

POSTGRESQL_IP=$(docker inspect postgresql | grep IPAddres | awk -F'"' '{print $4}') psql -U postgres -h ${POSTGRESQL_IP}

{% endhighlight %}

{% highlight sql %}
sql CREATE ROLE gitlab with LOGIN CREATEDB PASSWORD 'password'; CREATE DATABASE gitlabhq_production; GRANT ALL PRIVILEGES ON DATABASE gitlabhq_production to gitlab;
{% endhighlight %}

到此为止，我们已经得到了一个```gitlab数据库```。接下来，安装数据表结构。这是在gitlab容器运行时，通过``` app:rake gitlab:setup command```命令完成的。

{% highlight bash %}

docker run --name=gitlab -i -t --rm --link postgresql:postgresql \ -e "DB_USER=gitlab" -e "DB_PASS=password" \ -e "DB_NAME=gitlabhq_production" \ -v /opt/gitlab/data:/home/git/data \ sameersbn/gitlab:6.9.2 app:rake gitlab:setup

{% endhighlight %}

> 提醒：仅在初次运行时使用上述设置。

所有准备已就绪，开始启动GitLab应用：

{% highlight bash %}

docker run --name=gitlab -d --link postgresql:postgresql \ -e "DB_USER=gitlab" -e "DB_PASS=password" \ -e "DB_NAME=gitlabhq_production" \ -v /opt/gitlab/data:/home/git/data \ sameersbn/gitlab:6.9.2

{% endhighlight %}

### Redis

##### 内部Redis服务。

> 警告
> 内部集成的redis服务，将很快从镜像中移除。
> 请使用redis容器连接或外部redis服务。
> 我提醒过你哟！

GitLab运用redis服务的来存储键值对数据。Redis连接的详细信息，可通过环境变量来指定。如果没有指定，将启动一个内部的Redis服务器，从而不需要额外的配置。

##### 外部Redis服务

GitLab镜像，可配置为使用外部的Redis服务，从而避免使用内部的Redis服务器。在启动GitLab镜像时，通过环境变量来进行配置：

假设redis服务器地址为192.168.1.100。

{% highlight bash %}

bash docker run --name=gitlab -i -t --rm \ -e "REDIS_HOST=192.168.1.100" -e "REDIS_PORT=6379" \ sameersbn/gitlab:6.9.2

{% endhighlight %}

##### 连接到Redis容器

可以将镜像链接到Redis容器，以满足GitLab需要使用redis的要求。Redis服务器的别名应该设置为```redisio```，然后将镜像链接到```redisio```。

为了演示连接到Redis容器，我们将使用[sammersbn/redis](https://github.com/sameersbn/docker-redis)镜像。详情参见[docker-redis自述文件](https://github.com/sameersbn/docker-redis/blob/master/README.md)。

首先，从docker index中获取redis镜像：
{% highlight bash %}

docker pull sameersbn/redis:latest

{% endhighlight %}

让我们启动redis容器：
{% highlight bash %}

docker run --name=redis -d sameersbn/redis:latest

{% endhighlight %}

准备工作已就绪，现在可以启动GitLab应用了：

{% highlight bash %}

docker run --name=gitlab -d --link redis:redisio \ sameersbn/gitlab:6.9.2

{% endhighlight %}

### 邮件

在启动GitLab时，通过设置环境变量来指定邮件配置。默认被配置为试用Gmail来发送邮件，因此，需要一个有效的用户名和密码来登陆到Gmail服务器上。

需要设置好下方的环境变量，来提供邮件支持：

* SMTP_DOMAIN (默认： www.gmail.com)
* SMTP_HOST (默认： smtp.gmail.com)
* SMTP_PORT (默认： 587)
* SMTP_USER：用户名
* SMTP_PASS：密码
* SMTP_STARTTLS (默认：true)
* SMTP_AUTHENTICATION (默认为“login”，如果SMTP_USER被设置。)

{% highlight bash %}

docker run --name=gitlab -d \ -e "SMTP_USER=USER@gmail.com" -e "SMTP_PASS=PASSWORD" \ -v /opt/gitlab/data:/home/git/data \ sameersbn/gitlab:6.9.2

{% endhighlight %}

### SSL

访问GitLab应用时，可以使用SSL，以防止未经授权的情况下，访问存储库中的数据。虽然CA认证的SSL证书可以得到信任，但一个自签名等证书也是可以的，只要每个用户花一些额外的步骤来验证您网站的身份，提供相同级别的信任验证。我将在本章节完成这一目标的实现。

要通过SSL保护你的应用，你基本上需要做2件事情：私人密钥（key），SSL证书（CRT）。

当使用CA认证证书时，这些文件是由CA提供给您的。当使用自签名证书时，则需要你自己生成这些文件。如果你是使用的CA证书，那么可以跳过下面的部分。

跳转到[负载均衡器中使用HTTPS](https://index.docker.io/u/sameersbn/gitlab/using-https-with-a-load-balancer)章节，如果你使用了负载均衡如hipache,haproxy或者nginx。

#### 生成自签名证书

生成自签名的SSL证书，需要3个简单的步骤：

* 步骤1：创建服务器私钥：```openssl genrsa -out gitlab.key 2048```
* 步骤2：创建证书签名：```openssl req -new -key gitlab.key -out gitlab.csr```
* 步骤3：使用私钥和证书签名来注册证书：```openssl x509 -req -days 365 -in gitlab.csr -signkey gitlab.key -out gitlab.crt```

恭喜，你已经生成一个有效期为365天的SSL证书了。

#### 增强服务器安全性

本章节为您提供指导，以[加强服务器的安全性](https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html)。为了实现这一目标，我们需要更强有力的DHE参数：
{% highlight bash %}

bash openssl dhparam -out dhparam.pem 2048

{% endhighlight %}

#### 安装SSL证书

前面我们生成了4个文件，我们需要安装gitlab.key,gitlab.crt,dhparam.pem文件在gitlab服务器。CSR文件是不需要的，但一定要确保您安全的备份他（如果您以后需要使用）。

GitLab默认配置为到```/home/git/data/certs```查找SSL证书，然而，可以通过配置```SSL_KEY_PATH```, ```SSL_CERTIFICATE_PATH```和```SSL_DHPARAM_PATH```选项来更改。

如果您还记得上面的内容：```/home/git/data```是数据存储的路径。这意味着我们必须创建一个名为```/opt/gitlab/data/```的证书文件夹，并将证书拷贝到其中。作为一个路径安全的措施，我们将更新```gitlab.key```为只读权限。

{% highlight bash %}

mkdir -p /opt/gitlab/data/certs cp gitlab.key /opt/gitlab/data/certs/ cp gitlab.crt /opt/gitlab/data/certs/ cp dhparam.pem /opt/gitlab/data/certs/ chmod 400 /opt/gitlab/data/certs/gitlab.key

{% endhighlight %}
太棒了，我们现在只差一个步骤，就可以让你的应用更加安全了。

#### 启用HTTPS支持

可将```GITLAB_HTTPS```选项设置为true来启用HTTPS。此外，使用自签名的SSL证书时，将```SSL_SELF_SIGNED```设置为true(假设我们使用的是自签名证书)。

{% highlight bash %}

docker run --name=gitlab -d \ -e "GITLAB_HTTPS=true" -e "SSL_SELF_SIGNED=true" \ -v /opt/gitlab/data:/home/git/data \ sameersbn/gitlab:6.9.2

{% endhighlight %}

在此配置中，所有的http协议都会自动重定向到https协议中。然而，在负载均衡中这不是一个最佳的方案。

#### 在负载均衡器中使用HTTPS

负载均衡器诸如haproxy/hipache，通过普通的http协议跟后端进行通信。因此，当不使用负载均衡器时，需要在容器中安装SSL密钥和证书。

当使用负载均衡器时，你应该设置```GITLAB_HTTPS_ONLY```为false，```GITLAB_HTTPS```选项为true。并且，```SSL_SIGNED```设置为一个适当的值。至此，你可以配置负载均衡器来支持HTTPS请求了，但这超出了本文讨论的范围。请参阅[在HAProxy中使用SSL/HTTPS](http://seanmcgary.com/posts/using-sslhttps-with-haproxy)主题获取更多信息。

请注意，当```GITLAB_HTTPS_ONLY```被禁用时，应用不会将HTTP自动重定向到HTTPS协议下，这在前面也说过。不幸的是，hipache没有选项来支持HTTP到HTTPS的重定向。因此，你只能使用haproxy或nginx之类的负载均衡器。

总而言之，在docker中运行命令会是这样的：

{% highlight bash %}

docker run --name=gitlab -d \ -e "GITLAB_HTTPS=true" -e "SSL_SELF_SIGNED=true" \ -e "GITLAB_HTTPS_ONLY=false" \ -v /opt/gitlab/data:/home/git/data \ sameersbn/gitlab:6.9.2

{% endhighlight %}
再次提醒，如果您正在使用CA认证的SSL证书，那么去掉```-e "SSL_SELF_SIGNED=true"```选项。

#### 与服务器建立信任连接

本节介绍自签名SSL证书。如果您使用CA认证，则已经达成目标。

这部分更多的是客户端的配置，以确保客户端100%信任他们在与谁进行通信。

通过简单的增加服务器证书到他们的信任列表中，来完成这一目标。在Ubuntu中，通过附加```gitlab.crt```文件的内容到```/etc/ssl/certs/ca-certificates.crt```来完成。

再次说明，这是一个客户端配置，也就是说每个人都将和服务器通信，应该在他们的计算机上执行此配置。总之，分发```gitlab.crt```文件给您的开发人员，并要求他们将其添加到自己的信任的SSL证书列表中，如果不这么做，会出现类似的错误：

{% highlight bash %}
git clone https://git.local.host/gitlab-ce.git fatal: unable to access 'https://git.local.host/gitlab-ce.git': server certificate verification failed. CAfile: /etc/ssl/certs/ca-certificates.crt CRLfile: none
{% endhighlight %}
您也可以在网页浏览器中做同样的事。要知晓在Firefox中安装根证书的办法，请[参考这里](http://portal.threatpulse.com/docs/sol/Content/03Solutions/ManagePolicy/SSL/ssl_firefox_cert_ta.htm)。你也可以在Chrome找到类似的选项，确保在您的证书管理器中安装好证书。

你明白了吧，大概就是这样。

#### 安装可信赖的SSL服务器证书

如果您的GitLab CI服务器使用自签名SSL证书，那么您应该确保GitLab CI服务器的证书和GitLab服务器能够互相通信。

镜像默认会去```/home/git/data/certs/ca.crt```查找证书，然而，可以使用```CA_CERTIFICATES_PATH```选项更改这个默认值。

将```ca.crt```复制到```DataStore```（前面定义的/home/git/data）的certs文件夹中。该ca.crt文件应该包含所有要信任的服务器根证书。对于GitLab CI来说，这位于在[docker-gitlab-ci容器](https://github.com/sameersbn/docker-gitlab-ci)的[自描述文件](https://github.com/sameersbn/docker-gitlab-ci/blob/master/README.md#ssl)。

默认情况下，我们自己的服务器证书gitlab.crt将被添加到受信任的证书列表中。

#### 最终的命令

{% highlight bash %}

docker run --name=gitlab -d -h git.local.host \ -v /opt/gitlab/data:/home/git/data \ -v /opt/gitlab/mysql:/var/lib/mysql \ -e "GITLAB_HOST=git.local.host" -e "GITLAB_EMAIL=gitlab@local.host" -e "GITLAB_SUPPORT=support@local.host" \ -e "SMTP_USER=USER@gmail.com" -e "SMTP_PASS=PASSWORD" \ sameersbn/gitlab:6.9.2

{% endhighlight %}

如果你使用外部mysql数据库：

{% highlight bash %}

docker run --name=gitlab -d -h git.local.host \ -v /opt/gitlab/data:/home/git/data \ -e "DB_HOST=192.168.1.100" -e "DB_NAME=gitlabhq_production" -e "DB_USER=gitlab" -e "DB_PASS=password" \ -e "GITLAB_HOST=git.local.host" -e "GITLAB_EMAIL=gitlab@local.host" -e "GITLAB_SUPPORT=support@local.host" \ -e "SMTP_USER=USER@gmail.com" -e "SMTP_PASS=PASSWORD" \ sameersbn/gitlab:6.9.2

{% endhighlight %}

#### 在子目录中运行

如果你喜欢，GitLab的URI可以为http://localhost/gitlab，设置```GITLAB_RELATIVE_URL_ROOT=/gitlab(或你喜欢的)```。路径应该以斜线开始，但末尾不要有斜线。

{% highlight bash %}

docker run --name=gitlab -d \ -v /opt/gitlab/data:/home/git/data \ -e "GITLAB_RELATIVE_URL_ROOT=/gitlab" \ sameersbn/gitlab:6.9.2

{% endhighlight %}

当你更改了URI子目录的路径，需要重新编译所有的Assets资源。需要删除data sotre位置的```tmp/cache/VERSION```文件，或者通过```rm -Rf /PATH/TO/DATA_STORE/tmp```。清理完缓存文件后，重新启动容器。

#### 可用的配置参数

下面是自定义GitLab安装时可用选项的完整列表：

* GITLAB_HOST: 服务器主机名，默认为localhost。
* GITLAB_PORT: 服务器端口，默认为80。
* GITLAB_EMAIL: GitLab的电子邮件服务器，普通HTTP协议时为80端口，如果是443则将启用https。
* GITLAB_SUPPORT: GitLab服务器的电子邮件地址，默认为：support@localhost。
* GITLAB_SIGNUP: 启用或禁用用户注册。
* GITLAB_SIGNIN: 如果设置为false，那么登陆表单将不会显示在登录页面。默认值是true。
* GITLAB_PROJECTS_VISIBILITY: 设置仓库的默认可见性。可选值：```public```、```private```、```internal```。默认为```private```。
* GITLAB_BACKUPS: 通过cron自动进行备份。可选值：```disable```、```daily```、```monthly```。默认值为```disable```。
* GITLAB_BACKUP_EXPIRY: 配置备份的有效期，有效期过后备份将会被删除。当自动备份被禁用时，备份将永久保存（0秒），否则将在7天后删除（604800秒）。
* GITLAB_SSH_PORT: SSH端口号，默认22。
* GITLAB_RELATIVE_URL_ROOT: GitLab服务器的子域名，例如```/gitlab```，该选项没有默认值。
* GITLAB_HTTPS: 是否启用HTTPS，默认为false。
* GITLAB_HTTPS_ONLY:当GITLAB_HTTPS被启用时，是否允许原始HTTP协议进行访问。在使用负债均衡器时应该设置为false，默认为true。
* SSL_SELF_SIGNED: 是否使用自签名SSL证书，默认为false。
* SSL_CERTIFICATE_PATH: SSL证书的位置，默认为```/home/git/data/certs/gitlab.crt```。
* SSL_KEY_PATH: SSL私钥的位置，默认为```/home/git/data/certs/gitlab.key```
* SSL_DHPARAM_PATH: dhparam文件的位置，默认为```/home/git/data/certs/dhparam.pem```。
* CA_CERTIFICATES_PATH: SSL证书信任清单，默认位于：```/home/git/data/certs/ca.crt```.
* NGINX_MAX_UPLOAD_SIZE: 最大允许上传的附件大小，默认为20M.
* REDIS_HOST: Redis服务器的主机地址，默认为localhost。
* REDIS_PORT: Redis服务器的连接端口，默认为6379。
* UNICORN_WORKERS: unicorn工作进程数，默认为2。
* UNICORN_TIMEOUT: unicorn超时时间，默认为60秒.
* SIDEKIQ_CONCURRENCY: Sidekiq并发数，默认为5。
* DB_TYPE: 数据库类型，可选值为：mysql,postgres。默认为mysql。
* DB_HOST: 数据库主机名，默认为本地主机.
* DB_PORT: 数据库端口。mysql默认为3306，postgresql为5432。
* DB_NAME: 数据库中的数据库名，默认为gitlabhq_production。
* DB_USER: 数据库用户名，默认为root。
* DB_PASS: 数据库密码，默认为空。
* DB_POOL: 数据库连接池计数。默认为10。
* SMTP_DOMAIN: SMTP域名，默认为www.gmail.com。
* SMTP_HOST: SMTP服务器主机地址，默认为smtp.gmail.com。
* SMTP_PORT: SMTP服务器端口，默认为587。
* SMTP_USER: SMTP服务器用户名。
* SMTP_PASS: SMTP服务器密码。
* SMTP_STARTTLS: 是否启用STARTTLS，默认为true。
* SMTP_AUTHENTICATION: 指定SMTP验证方法，默认为```login```，如果SMTP_USER被设置。
* LDAP_ENABLED: 启用LDAP，默认为false。
* LDAP_HOST: LDAP服务器主机地址。
* LDAP_PORT: LDAP端口，默认为636。
* LDAP_UID: LDAP UID. 默认为 sAMAccountName。
* LDAP_METHOD: LDAP模式，可选值为```ssl```,```tls```,```plain```,默认为ssl。
* LDAP_BIND_DN: 无默认值。
* LDAP_PASS: LDAP 密码。
* LDAP_ALLOW_USERNAME_OR_EMAIL_LOGIN: 如果启用，则GitLab会忽略掉@以及后面的部分再进行提交。默认为false，如果LDAP_UID是userPrincipalName的话。否则为true。
* LDAP_BASE:搜索用户的地址，无默认值。
* LDAP_USER_FILTER:过滤LDAP用户，无默认值。

## 维护
### SSH Login

有两种方法可以以root权限登陆到容器中。第一种办法是将RSA公钥添加到authorized_keys文件中并构建一个镜像。

第二种办法是使用动态生成的密码。每当容器开始使用pwgen给root分配一个随机密码。此后密码可通过```docker logs```来获取。

{% highlight bash %}

docker logs gitlab 2>&1 | grep '^User: ' | tail -n1

{% endhighlight %}

此密码会在镜像被执行的时候更改掉，因此他不是持久的。

### 创建备份

Gitlab定义了一个rake任务，可以轻松的对gitlab进行备份。备份的内容包括所有的git仓库，上传的文件和数据库。

在备份之前，请确保gitlab没有在运行中：
{% highlight bash %}

docker stop gitlab

{% endhighlight %}
你现在需要运行gitlab rake任务，创建一个备份。

{% highlight bash %}

docker run --name=gitlab -i -t --rm [OPTIONS] \ sameersbn/gitlab:6.9.2 app:rake gitlab:backup:create

{% endhighlight %}
备份将在Data Store文件夹中被创建。

### 还原备份

Gitlab中定义了一个rake任务来轻松恢复gitlab备份。在执行操做前，确保gitlab镜像没有处于运行状态：

{% highlight bash %}

docker stop gitlab

{% endhighlight %}

要恢复备份，需要在交互模式下（-i -t）模式下运行镜像，并向容器发送```app:restore```。
{% highlight bash %}

docker run --name=gitlab -i -t --rm [OPTIONS] \ sameersbn/gitlab:6.9.2 app:rake gitlab:backup:restore

{% endhighlight %}

还原操作将列出按时间倒序排列的所有可用备份，选择要恢复的备份文件，来完成恢复操作。

### 自动备份

镜像可以被配置为自动按照每日、每月进行备份。添加```-e``` "GITLAB_BACKUPS=daily"来启动每日备份，而使用```-e``` "GITLAB_BACKUPS=monthly"将启动每月备份。

每日凌晨4点（UTC）将开始备份任务，而月备份则在相同时间的每月1号开始创建。

默认情况下，当自动备份被启用，会自动删除早于7天的备份。当自动备份禁用时，备份将会被永久保留。此行为可以通过```GITLAB_BACKUP_EXPIRY```选项来更改。

### 升级

GitLabHQ会在每月的22号发布新版本，修复缺陷的版本则紧随其后。我几乎在新版本发布时，立刻进行更新（至少目前为止是这样的）。如果你在生产环境中使用，则我建议在gitlab发布2天后，再做更新。给予一些时间给gitlab，以保证稳定性。

要更新到最新版的GitLab，只需要简单的5步操作：
* 步骤1：停止当前运行的镜像：
{% highlight bash %}

docker stop gitlab

{% endhighlight %}

* 步骤2：备份应用数据
{% highlight bash %}

docker run --name=gitlab -i -t --rm [OPTIONS] \ sameersbn/gitlab:6.9.2 app:rake gitlab:backup:create

{% endhighlight %}

* 步骤3：更新镜像
{% highlight bash %}

docker pull sameersbn/gitlab:6.9.2

{% endhighlight %}

* 步骤4：迁移数据库
{% highlight bash %}

docker run --name=gitlab -i -t --rm [OPTIONS] \ sameersbn/gitlab:6.9.2 app:rake db:migrate

{% endhighlight %}

* 步骤5：启动镜像
{% highlight bash %}

docker run --name=gitlab -d [OPTIONS] sameersbn/gitlab:6.9.2

{% endhighlight %}

### Rake任务

```app:rake```命令允许您运行gitlab rake任务。要运行rake任务，只需要指定要执行的应用程序的任务：```app：rake```即可。例如，手机GitLab的运行时信息：
{% highlight bash %}

bash docker run --name=gitlab -d [OPTIONS] \ sameersbn/gitlab:6.9.2 app:rake gitlab:env:info

{% endhighlight %}
同样，导入一个空仓库到GitLab中：
{% highlight bash %}

bash docker run --name=gitlab -d [OPTIONS] \ sameersbn/gitlab:6.9.2 app:rake gitlab:import:repos

{% endhighlight %}

有关rake任务的完整列表，请参见[For a complete list of available rake tasks please refer https://github.com/gitlabhq/gitlabhq/tree/master/doc/raketasks or the help section of your gitlab installation].或gitlab安装的帮助部分。

## 引用

* https://github.com/gitlabhq/gitlabhq
* https://github.com/gitlabhq/gitlabhq/blob/master/doc/install/installation.md
* https://github.com/gitlabhq/gitlabhq/blob/master/doc/install/requirements.md
* http://wiki.nginx.org/HttpSslModule
* https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html
* https://github.com/gitlabhq/gitlab-recipes/blob/master/web-server/nginx/gitlab-ssl

（完）