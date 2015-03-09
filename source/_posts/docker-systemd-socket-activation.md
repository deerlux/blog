title: 利用systemd按需激Docker容器
date: 2015-03-08 19:54:04
tags:
---

能够延迟启动网络服务应用直到此服务被请求时，是[systemd](http://www.freedesktop.org/wiki/Software/systemd/)的一个特性，它通过一个[socket激活](http://0pointer.de/blog/projects/socket-activation.html)的进程实现。这倒并不是一个新的创意，systemd借用了[OS X](https://www.apple.com/osx/)自2005年[Tiger](http://en.wikipedia.org/wiki/Launchd)版本以来的[launched](http://en.wikipedia.org/wiki/Launchd)的实现思路，再往前追溯，古老的Unix [inetd](http://en.wikipedia.org/wiki/Inetd)在上世纪80年代就实现了这种启动方式的一个简单版本。不管是在脚本驱动还是在事件驱动的启动系统中，Socket激活的方式都具有很多优点，尤其是它能够有效地对应用的启动顺序和服务描述进行解耦合，甚至可以解决一些应用的循环依赖问题。Docker容器（或者其他类型的应用服务例如[systemd's nspawn](http://lwn.net/Articles/572957/)或[LXC](https://linuxcontainers.org/)）大都是一些专用的网络进程，所以将socket激活的方式来启动这些进程会比较有用。

# Socket激活

![Creation of Adam](https://developer.atlassian.com/blog/2015/03/docker-systemd-socket-activation/creation.jpg)

Socket激活是这样工作的：systemd的守护进程代表其他应用监听相关的sockets，只有对应的连接到来的时候才会启动相应的应用服务，然后便将传入的连接交给新启动的应用服务，此应用负责对这个连接进行响应。

但是这里的限制之一是，它需要被激活的应用知晓它是可以被socket激活的，并且对现成的socket的处理——尽管[简单](http://0pointer.de/blog/projects/socket-activation.html)——并不同于从头创建一个监听socket。因此，很多广泛应用的软件（如[ngix](http://trac.nginx.org/nginx/ticket/237)）并不支持这种方式，软件容器化的趋势添加了另外需要激活的层则进一步加剧了这种限制条件的影响。但是可以通过将其交给容器的方式针对任意容器化的应用解决此问题。

# Socket激活和Docker

如果你Google "docker container systemd socket activation" 你会发现很少的相关的讨论，大部分的结论都是除非Docker给出了这方面的支持否则不可能，但Dcoker支持虽然是最优的解决方式但并不是全部。Sytemd的开发者已经知识在任意情况下都可以激活可能解决起来颇费周折，因此在[209](http://article.gmane.org/gmane.comp.sysutils.systemd.devel/16993)版本时引入了[systemd-socket-proxyd](http://www.freedesktop.org/software/systemd/man/systemd-socket-proxyd.html)——一个小的TCP和UNIX域socket代理。这个东西才是真正的理解了激活的含义，它可以坐在网络和我们的容器之间，透明地在二者之间转发包，因此我们可以通过少许的[units](http://www.freedesktop.org/software/systemd/man/systemd.unit.html)（systemd的配置系统）为Docker容器创建一个socket激活的框架。

**警示**：当前的Ubuntu发行版本systemd是208版本，还没有systemd-socket-proxyd，要想试验下述例子必须systemd 209版本或更高，Debian的Jessie pre-release版本可以，基于RedHat的最新的发行版如Fedora应该也可以。

# 如何工作的

通常，我们用一个演示来简化说明问题：

![system activation animation](https://developer.atlassian.com/blog/2015/03/docker-systemd-socket-activation/systemd-docker.gif)

我们看一下到底发生了什么事情：

1. 我们创建了一个socket监听相应的端口，它是由代理/容器依事件驱动合并进行服务的；
1. 当第一个连接来的时候，systemd激活了代理服务用来处理这个socket；
1. 还有一个由容器提供的被动式服务——代理依赖于此服务，代理启动之前首先要启动这个容器；
1. 代理负责在网络和容器之前传送所有的流量。

虽然需要一些技巧，但从概念上讲这是相当简单的，那么我们以systemd和一个ngix的容器来练习一下如何实现。

# 使其运行
## 创建容器

首先我们要创建目标容器，这里我们利用[官方镜像](https://registry.hub.docker.com/_/nginx/)创建一个空的ngix容器。

``` shell
docker create --name ngix8080 -p 8080:80 ngix
```

这与你执行任何容器均类似，不过注意我们用的是*create*而不是*run*，因为我们并不想这个容器马上启动，我们还对此窗口进行了命名以便于后面启动它。

这里唯一一点技巧是你需要将此容器绑定到另外一个端口（这里我们用的是8080代替的80），这是因为这个端口（80）

现在我们有了一个容器，这样便可以创建激活管道，首先我们需要初始的监听socket也就是[socket unit](http://www.freedesktop.org/software/systemd/man/systemd.socket.html)，这个socket的行为是高度可配置的，但是我们的例子中非常简单。

``` ini
[Socket]
ListenStream=80
[Install]
WantedBy=socktes.target
```

*[Socket]*节表示一个简单的80端口的和TCP监听，*[Install]*节告诉systemd什么时候启动这个socket，在此例中它与系统配置的其他socket一起启动。我们将此unit写入名为*/etc/systedm/system/nginx-proxy.socket*的文件中，这只是启动后需要激活的链条中的一部分，我们还需要告诉systemd启动它：

``` shell
systemctl enable nginx-proxy.socket
systemctl start nginx-proxy.socket
```

## 代理服务

当systemd接收到一个到此socket的连接时它化自动查找一个相同名称的[服务](http://www.freedesktop.org/software/systemd/man/systemd.service.html)并启动它。我们需要创建服务文件*/etc/systemd/systedm/nginx-proxy.service*：

``` ini
[Unit]
Requires=nginx-docker.service
After=nginx-docker.service

[Service]
ExecStart=/lib/systemd/systemd-socket-proxyd 127.0.0.1:8080
```

*[Unit]*节描述了此服务的依赖，此例中我们告诉systemd在代理启动之前必须先将实际的容器（下面我们将要配置此服务）启动。*[Section]*节为启动代理进程的响应，当前此节还可以配置[大量的内容](http://www.freedesktop.org/software/systemd/man/systemd.service.html)，比如进程启动失败的处理，但是对于我们的应用一个简单的启动就OK了。注意我们将socket转发到了8080端口，这是我们的容器将要监听的端口；我们也并没有告诉systemd监听80端口，它只是用了systemd处理的socket。还要注意到上述配置文件中没有*[Install]*节，此服务并非是缺省启动，而是由socket激活它。

## 启动Docker容器

正如前面提到了，代理利用*Requre/After*机制触发了容器的启动，此容器的启动文件在*/etc/systemd/system/nginx-docker.service*，它看起来是这样的：

``` ini
[Unit]
Description=nginx container

[Service]
ExecStart=/usr/bin/docker start -a nginx8080
ExecStartPost=/bin/sleep 1

ExecStop=/usr/bin/docker stop nginx8080
```

与上述代理服务的基本概念相同，*ExecStart*行告诉systemd如何启动这个容器，systemd喜欢让进程不要在后台运行，所以我们添加了*-a*参数，这使得此容器在前台运行并且将nginx的运行日志转发到[journald](http://0pointer.de/blog/projects/journalctl.html)logger中，*ExecStop*告诉systemd当运行*systemctl stop ngix-docker*时如何停止容器。

这里我们还利用*ExecStartPost*行耍了一个小聪明，这是systemd在主进程启动后立马要运行的进程，此例中我们在继续往下走之前让其sleep了1秒钟。这是必要的，因为尽管docker启动容器非常快，但是容器中的进程的启动可能需要稍微长一点的时间完成初始化，systemd也非常快，所以有可能代理在nginx准备好接收之前就开始转发了，所以我们添加了一点延迟，给Dcoker/nginx一个空闲启动，这有点耍赖但是很有效（但是看[下面](#容器服务改进)）

# 一切就绪

现在你可以：当systemd启动的时候开启一个socket，当第一个连接到达时，级联的依赖关系会使nginx Docker通过代理对此连接进行响应。Easy。

# 容器服务改进

任何想要优化系统启动时间的人都恨死了在代码或配置文件中随意添加诸如*sleep*的语句，他们要么是因系统很快而不需要此延迟，要么是得到了很多难以定位的随机错误。上述*ExecStartPost*中的*sleep*语句也同样使我神经过敏，所以我也想把它删了。其实我们真正想做的是确认端口的监听已经启动了，但是通过了一种sleep的方式。我们可以通过一种循环检测端口是否已经正常启动的方式来实现，我用[netcat](http://en.wikipedia.org/wiki/Netcat)写了一个wrapper的脚本。

``` shell
#!/bin/bash

host=$1
port=$2
tries=600

for i in `seq $tries`; do
    if /bin/nc -z $host $port > /dev/null ; then
        # Ready
        exit 0
    fi

    /bin/sleep 0.1
done

# FAIL
exit -1
```

这个脚本需要一个host和port的参数，检查一下是否有响应，每秒钟检查10次，如果1分钟之内还没有响应就返回失败。我们将此脚本安装到*/usr/local/bin/waitport*（设置其为可运行），现在我们的*nginx-docker.service*文件改为这样：

``` ini
[Unit]
Description=nginx container

[Service]
ExecStart=/usr/bin/docker start -a nginx8080
ExecStartPost=/usr/local/bin/waitport 127.0.0.1 8080

ExecStop=/usr/bin/docker stop nginx8080
```

也就是说，waitport脚本可以根据你的系统应用灵活调整，如果你的容器启动速度很快的话通常都会立即返回。


