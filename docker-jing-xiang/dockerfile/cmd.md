# CMD--与进程前后台和容器存活的关系

设置镜像运行出来的容器的缺省命令

有两种写法，写多个和FROM一个已经有CMD的镜像的话，以最后一个为准

```text
CMD ["executable", "param1", "param2"] 
CMD command param1 param2 
```

前者是exec格式也是推荐格式，后者是/bin/sh格式，exec和CMD还有ENTRYPOINT这三者之间联系非常紧密，后面单独将相关的知识点。这里先用一个例子讲/bin/sh格式啥意思

![](../../.gitbook/assets/image%20%2815%29.png)

我们发现pid为1的是一个/bin/sh的进程，而我们的进程在容器里在后面。容器是单独一个pid namespaces的。这里懒得去做个图了，借用下别人的图

![](../../.gitbook/assets/image%20%281%29.png)

默认下所有进程在一个顶级的pid namespaces里，pid namespaces像一个树一样。从根到最后可以多级串。容器的pid namespaces实际上是在宿主机上能看到的，也就是下面，我们可以看到容器在宿主机上的进程，由于子namespaces无法看到父级的namespaces，所以容器里第一个进程\(也就是cmd\)认为自己是pid为1，容器里其余进程都是它的子进程。

![](../../.gitbook/assets/image%20%286%29.png)

在Linux中，只能给init已经安装信号处理函数的信号，其它信号都会被忽略，这可以防止init进程被误杀掉，即使是superuser。所以，kill -9 init不会kill掉init进程。但是容器的进程是在容器的ns里是init级别，我们可以在宿主机上杀掉它，之前线上的低版本docker 命令无法使用，同事无法停止错误容器，我便询问了进程名在宿主机找到后kill掉的。

![](../../.gitbook/assets/image%20%2831%29.png)

接下来说说为啥推荐exec格式，exec格式的话第一个进程是我们的sleep进程，大家可以自己去构建镜像试试。推荐用exec格式是因为pid 为1的进程承担着pid namespaces的存活周期，听不懂的话我举个例子

```text
[root@guan ~]# docker run -d centos ls
24b2195731fef5b3e52898bcb7e2c6cebdb9afb8cfc929c1e69ed7126e967699
[root@guan ~]# docker run -d centos sleep 10
8c0a7cba4af9a847e0092e1855426149cf093ef90fd4b91b1cbf452001176a38
[root@guan ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                     PORTS                                      NAMES
8c0a7cba4af9        centos              "sleep 10"               4 seconds ago       Up 3 seconds                                                     cocky_visvesvaraya
24b2195731fe        centos              "ls"                     10 seconds ago      Exited (0) 9 seconds ago                                         friendly_mirzakhani
[root@guan ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS                          PORTS                                      NAMES
8c0a7cba4af9        centos              "sleep 10"               About a minute ago   Exited (0) 15 seconds ago                                       cocky_visvesvaraya
24b2195731fe        centos              "ls"                     About a minute ago   Exited (0) 29 seconds ago                                       friendly_mirzakhani
```

先看下docker run命令格式

```text
$ docker run --help
Usage:	docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

docker run 后面镜像后面的command和arg会覆盖掉镜像的CMD。上面我那个例子覆盖掉centos镜像默认的CMD bash。我们可以看到ls的容器直接退出了，但是sleep 10的容器运行了10秒后就退出了。以上也说明了**容器不是虚拟机，容器是个进程**

这说明了容器的存活是容器里pid为1的进程运行时长决定的。所以nginx的官方镜像里就是用的exec格式让nginx充当pid为1的角色

```text
CMD ["nginx", "-g", "daemon off;"]
```

这里nginx启动带了选项是什么意思呢，我举个初学者自己造轮子做nginx镜像来举例，也顺带按照初学者重复造轮子碰到错误的时候应该怎样去排查

![](../../.gitbook/assets/image%20%2821%29.png)

上面我是按照初学者虚拟机的思维去做一个nginx镜像，结果构建错误，我们发现有个失败的容器就是RUN那层创建出来的，前面我说的实际上docker build就是运行容器执行步骤然后最后底层调用commit的原因。

现在我们来手动排下错，哪步报错可以把那步到后面的全部注释掉后构建个镜像，然后我们run起来的时候带上-ti选项分配一个能输入的伪终端，最后的command用sh或者bash，这样容器的主进程就是bash或者sh了，我们在里面执行报错的RUN\(这里我例子简单，所以我直接run -ti centos bash\)

实际上会发现nginx是在epel-release的源里

![](../../.gitbook/assets/image%20%2827%29.png)

接下来改下Dockerfile再构建试试

```text
$ cat Dockerfile
FROM centos
RUN yum install -y epel-release \
    && yum install -y nginx
CMD ["nginx"]
$ docker build -t test .
```

然后又是一个新手自己做镜像遇到的问题了，这个镜像运行了根本跑不起来，我们手动bash或者sh进去排查

```text
$ docker run -d -p 80:80 test
f13e98d4dc13b6fa13e375ca35cc58a23a340a07b677f0df245fc1ef3b7199c6
$ docker ps -a
CONTAINER ID     IMAGE        COMMAND      CREATED             STATUS                    PORTS               NAMES
f13e98d4dc13     test         "nginx"      3 seconds ago       Exited (0) 1 second ago                       determined_elgamal
```

![](../../.gitbook/assets/image%20%2819%29.png)

为什么这里nginx明明可以起来呢\(当然如果上面run命令加映射的话此时也可以被外部访问到的\)？

不考虑镜像大小和制作精细规范否，现在我这个错误实例的CMD就是和nginx官方的CMD不同，我们在容器里执行下官方的完整的CMD试试

![](../../.gitbook/assets/image%20%2825%29.png)

似乎是卡主了？我们可以访问宿主机的ip:80看看会发现实际能访问到的，也就是说这样也是在运行，当然我们把CMD改成和官方一样直接`docker run -d -p 80:80 test`的话容器是不会退出的。

至于说为啥？答案就是**前台**的概念！

我们有没有发现我们手动执行nginx带关闭daemon选项发现类似于hang住一样，实际上它就是前台跑。

单独的nginx回车，实际上是它拉起来了nginx，然后它退出了，**但是！！！**，别忘记了你这个nginx是pid为1的角色，你退出了你下面子进程全部完蛋，容器也会显示退出。所以既然你最终要跑nginx，你nginx得是前台跑。

判断一个命令\(或者说带上选项\)是不是前台跑的最简单一个验证就是\(主进程sh或者bash进去后\)执行它看它有没有回到终端。例如ls和yes命令，我们会发现yes命令一直刷y没有回到终端。

其实发展到现在，很多以前只有daemon后台跑的进程都慢慢的在docker火热下开始有前台运行的选项或者配置了，例如

* redis的配置文件不写日志文件路径它就默认前台跑
* uwsgi也是一样，命令行参数或者配置文件指定了日志文件路径就后台跑，否则前台跑
* node本身是前台跑，但是一些信号可能不好处理，于是有了pm2
* zabbix 的日志路径写console的话就是前台跑
* 也有其他的话欢迎读者提issue或者pr

其实我们用上前台选项的话也无法用docker logs看容器的log，是因为docker logs查看的是容器里的标准输出信息，我们可以看到官方nginx镜像Dockerfile是这样做的

```text
	# forward request and error logs to docker log collector
	&& ln -sf /dev/stdout /var/log/nginx/access.log \
	&& ln -sf /dev/stderr /var/log/nginx/error.log
```


