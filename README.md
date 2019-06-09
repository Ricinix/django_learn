# django_learn
按照django官方文档敲出来的代码，没啥东西。综合大多数服务器部署教程，自己写了下在服务器上的部署方法

---

## 常用Linux命令

netstat -tunlp|grep 8888		查看端口8888是否被占用

ps aux|grep uwsgi					   查看使用了uwsgi命令的进程

nginx												启动nginx

nginx -s stop								 停止nginx

uwsgi --ini uwsgi.ini				利用ini配置文件启动项目

uwsgi --stop uwsgi.pid			利用pid文件终止进程

uwsgi --reload uwsgi.pid		利用pid文件重启进程

nohup command > myout.file 2>&1 &		后台运行并且输出日志到当前目录的myout.file中

tail -f file.log								 动态查看日志

tail -n 20 file.log				  		查看最后20行日志

tail file.log									   查看日志（默认10行）

head -n 1000 file.log					查看开头1000行日志

wget url											下载

tar -zxvf filename.tar.gz			解压

./configure --prefix=/  				编译的时候用来指定程序存放路径 。

make & make install					编译加安装



**另外**：若不指定prefix来安装

可执行文件默认放在/usr /local/bin，
库文件默认放在/usr/local/lib，
配置文件默认放在/usr/local/etc。
其它的资源文件放在/usr /local/share

---

## 正式流程

### 1.安装python3

​	看过很多的安装，如果用yum或者apt-get来安装的话，会比较难管理，而且版本也难以控制，因为文件会分散在bin、lib、etc、share等文件夹中，而且下载源可能比较旧。所以我们可以用传统的“下载+编译+安装”的三步走方法来安装。

​	首先我们先安装一些依赖包：

```shell
yum install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gcc make
```

​	我们先找一个文件比较少的地方（随你放哪了），用wget命令下载python的压缩包

```shell
wget https://www.python.org/ftp/python/3.6.8/Python-3.6.8.tar.xz
```

​	3.6.8会比较容易装一点。我试过安装3.7.0，但是3.7以上的版本会有一个新的依赖包需要装，如果有需要的话可以查一查3.7版本的安装。我这里就挑简单一点的安装了。

​	然后我们调用命令解压

```shell
tar xf Python-3.6.8.tar.xz -C /usr/local/src/
```

​	这个命令是指解压到/usr/local/src/目录中，毕竟这只是源码，还不是最后的程序。如果说没有该目录的话就去用mkdir创建一个就好了

​	接着我们切换到/usr/local/src/目录中，然后调用配置安装路径的命令

```shell
./configure --prefix=/usr/local/python3
```

​	这行命令的意思是利用源码中的configure文件，把编译和安装路径设置为/usr/local/python3。然后我们调用编译安装命令就搞定了

```shell
make && make install
```

​	然后我们调用下一个命令先升级下pip

```shell
/usr/local/python3/bin/pip3 install --upgrade pip
```

​	如果你嫌每次都要打一长串路径的话，你可以用以下命令设置一下

```shell
ln -s /usr/local/python3/bin/pip3 /usr/bin/pip3
```

​	这样每次打pip3就相当于打了/usr/local/python3/bin/pip3了。

​	设置后软连接后，我们可以看一下pip是否升级成功了：

```shell
pip3 list
```

​	然后我们再给新安装的python设置软连接：

```shell
ln -s /usr/local/python3/bin/python3 /usr/bin/python3
```

​	如果说设置软连接的时候说已存在python3或者说前面的已存在pip3，那就把/usr/bin/中的python3、pip3删掉就好了。

​	

### 2.安装Django及uwsgi

​	这个的话直接用pip3的命令来安装就好了

```shell
pip3 install django
```

​	如果对版本有要求的话，可以自己再稍微微调一下。

​	然后我们再接着安装uwsgi网关接口：

```
pip3 install uwsgi
```

​	我们其实这样就已经可以做到部署一个小型的项目到服务器上了。

​	或者说连uwsgi可能都不需要。直接在项目目录下调用命令：

```shell
python manage.py runserver 0.0.0.0:8000
```

​	然后我们打开浏览器，输入相应服务器ip:8000就可以看到页面了。但是这个样子只是前台运行而已，一关闭与服务器shell的连接就没了。或许你会说用nohup &命令让其在后台运行，那也可以，但是这个样子的话，我们可以设置的地方就太少了，比如说连接数量、线程数等等。所以我们就需要用到uwsgi网关接口了。

​	当然我们先来配置以下软连接

```
ln -s /usr/local/python3/bin/uwsgi /usr/bin/uwsgi
```

​	然后我们在原来项目的根目录下(也就是与app同级的地方)，新建一个wsgi.py。往里面写：

```python
import os

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "mysite.settings")

from django.core.wsgi import get_wsgi_application
application = get_wsgi_application()
```

​	这一段代码的作用是载入了项目里的设置，还有找到了app并提供给了uwsgi网关接口。

​	然后我们再在刚才那个目录下新建一个uwsgi.ini文件，往里面写：

```python
[uwsgi]
socket=127.0.0.1:8888
chidr=/usr/myprog/python/mysite  #项目目录
wsgi-file=/usr/myprog/python/mysite/wsgi.py #项目目录下的wsgi.py 可以通过这个py文件找到app
processes=4
threads=2
master=True
pidfile=/usr/myprog/python/mysite/uwsgi.pid #这个路径别瞎设，以后还得靠这个pid来结束uwsgi进程的
daemonize=/usr/myprog/python/mysite/uwsgi.log #日志文件，一旦用ini来启动项目，所有的输出都在这里了，不看log都不知道bug在哪
```

​	其中项目目录得写你自己的项目放的地方，有备注的四个设置项都得改成自己的。

​	这时候我们最好用命令检测下8888端口有没有被占用：

```shell
netstat -tunlp|grep 8888
```

​	如果没有显示东西的话就是没有进程占用了，那就可以使用了。如果有进程占用了，那就只好换一个没有被使用的端口号了。如果你偏要用这个端口号的话，那可以把相关的进程给kill掉

​	至于为什么是127.0.0.1呢？不应该是服务器ip地址吗，怎么是本地地址呢。如果我们单单使用uwsgi网关来启动项目的话，那么这个地方是该写成服务器的ip地址然后设置成http而不是socket的。

​	但是，为了搞更多的花样，我们等下还得弄nginx。所以这个地方就这样设置了。因为配置了nginx之后，nginx监听服务器某个端口有没有收到请求，如果收到请求的话，那就发给uwsgi网关，然后uwsgi网关再把数据给django处理，然后再原路返回。而nginx和uwsgi之间的通信是在本地进行的，所以nginx就向本地的一个端口发数据，然后uwsgi也监听本地的那个端口来接受nginx的数据。

​	怎么感觉这么蛋疼的...这一套又嵌一套的...



### 3.安装nginx

​	这个的安装我是直接用yum命令的：

```shell
yum install -y nginx
```

​	试过是可以的，所以就没有用下载源码再编译的方法了（不到万不得已还是想偷下懒...）。

​	然后我们开始给nginx设置了。我们先切换到nginx的配置目录下

```shell
cd /etc/nginx/conf.d
```

​	然后我们新建一个conf文件：

```shell
vi mysite.conf
```

​	然后我们按i然后往里面写：

```
server {
        listen       80;
        server_name  xxx.xxx.xxx.xxx;
        charset      utf-8;
        client_max_body_size 75M;

        location / {
            include  uwsgi_params;
            uwsgi_pass  127.0.0.1:8888;
        }
    }

```

​	当然server_name不是真的让你写那么多个x，这个地方你就写你自己的服务器ip就可以了，这里我写的是监听80端口号。

​	然后按Esc然后：wq保存退出

​	这下我们就配置好nginx了。我先来稍微解释一下为啥这样配置。首先，上面这段里面的uwsgi_pass就是往哪个地方把数据传给uwsgi，这就是我们上面uwsgi中设置的本地地址127.0.0.1:8888了，当然端口肯定要一样的。其他的参数有兴趣的话也可以去搜索一下其他资料

​	那为什么我新建一个conf，nginx会知道要使用这个新增的配置文件呢？其实在上一层目录，就是/etc/nginx中有一个文件叫做nginx.conf，每次nginx启动就会载入这个配置文件，而这个配置文件里面的最后一行又include了conf.d目录下的所有conf文件。所以我们直接在conf.d目录下新建一个conf文件，就能直接配置到nginx中了。

​	nginx大多数是拿来做负载均衡的，所以一般会连上多个服务器，也就是在conf.d目录下会有不少conf文件，不只是连向本服务器的uwsgi。如果你有访问量大的需求的话，可以在conf.d中让nginx把数据发到别的服务器上处理。具体的就不细讲了，有兴趣的话可以另外查一查相关资料...



### 4.正式启动项目

​	终于到了激动人心的时候了。我们先启动一下nginx

```shell
nginx
```

​	这行代码就是这么简单 ...

​	如果要停止nginx的话就：

```shell
nginx -s stop
```

​	如果成功开启nginx的话，用浏览器访问正确的url，然后就会有nginx默认的成功启动的页面了。

​	然后我们切换到django项目目录下，然后输入uwsgi启动命令：

```shell
uwsgi --ini uwsgi.ini
```

​	然后会有一行提示，说正在从ini文件中获取配置(大致好像是这个意思吧...)，然后我们这个时候看一下日志，就能看到启动消息了：

```shell
tail -n 50 uwsgi.log
```

​	观察一下，如果没有说什么错误的话，就是ok了。这时候我们再刷新一下浏览器。你原本在电脑上调试的样子就又出现在屏幕中了，只不过这次代码不在电脑本地上跑了，而是在服务器上了。

​	如果要关闭uwsgi的话，可以用以下命令：

```shell
uwsgi --stop uwsgi.pid
```

​	所以说别把pid文件搞不见了...



## 最后说几点

​	在项目的settings中，把debug设置为False，毕竟是上线了的东西了，就没必要出错还显示错哪了...

​	然后有个settings中还有个ALLOWED_HOSTS的配置，这里项目中我是设置了“\*”。但是为了安全起见，最好设置成自己的域名，如果没有域名的话，就写服务器的ip地址就好了。写啥都好过“\*”了。

​	

​	如果说有什么好的软件可以连接到服务器shell的话，那么putty是个不错的选择，轻量且免费。具体的putty设置相关我这里就不细说了，如果想要知道putty的免密码登陆啥的话，另找资料...