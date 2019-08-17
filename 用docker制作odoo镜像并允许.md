#下面是整理在docker部署odoo的过程和其中的坑
___

##一、准备资料
>* 1、 从**odoo**官方docker下载这三个文件：**Dockerfile**、**entrypoint.sh**、**odoo.conf**，参考链接如下
   https://github.com/odoo/docker/tree/master/12.0 
>* 2、 下载要部署的**wkhtmltopfd**发行版deb包（包括SHA1SUMS码）（对应操作系统，我这里选用debain：stretch）
   **把文件名改简短：wkhtmltox.deb:**
   https://github.com/wkhtmltopdf/wkhtmltopdf/releases/download/0.12.5/wkhtmltox_0.12.5-1.stretch_amd64.deb  
   **SHA1SUM:**
    https://github.com/wkhtmltopdf/wkhtmltopdf/releases/download/0.12.5/SHA1SUMS  
>* 3、 下载要部署的**odoo12.0**发行版deb包（包括对应日期的odoo_12.0.xxxxxxxx.dsc文件）
    **把文件名该简短：odoo12.deb**
   http://nightly.odoo.com/12.0/nightly/deb/  
---

##二、准备环境
>* 1、 **云服务器**或者**本地服务器** linux环境（本人环境是阿里云ubuntu16.04）
>* 2、 已经安装**docker**（详情自行了解安装）
>* 3、 安装好**docker**后先安装**postgresql**数据库,选择对应版本的数据库把镜像pull下来，然后启动一个容器：
```shell
    #docker pull postgres:10.0
    #docker run -d -e POSTGRES_USER=odoo -e POSTGRES_PASSWORD=odoo -e POSTGRES_DB=postgres --name db postgres:10.0
```
> <提示：运行的容器镜像版本号必须与上面的的镜像版本号对应，如10.0对应10.0，否则会报错>
---
##三、修改Dockerfile和odoo的配置文件
>* 1、在运行dockerfile时候，由于wkhtmltopdf和odoo12.0这两个包太大，经常断线造成下载失败
故需要修改这两处的代码,改成复制当前目录下的deb包到容器的/var/local/目录下。（此目录可根据
自己习惯选择，稍后会在容器里面安装这两个包)如下：

``` 
---注释或者删除这两部分代码（25-30行，59-69行）---
25         xz-utils \#**记得把这个\也删掉**
26         #&& curl -o wkhtmltox.deb -sSL https://github.com/wkhtmltopdf/wkhtmltopdf/releases/download/0.12.5/wkhtmltox_0.12.5-1.stretch_amd64.deb \
27         #&& echo '7e35a63f9db14f93ec7feeb0fce76b30c08f2057 wkhtmltox.deb' | sha1sum -c - \
28         #&& dpkg --force-depends -i wkhtmltox.deb\
29         #&& apt-get -y install -f --no-install-recommends \
30         #&& rm -rf /var/lib/apt/lists/* wkhtmltox.deb

59    # Install Odoo
60    #ENV ODOO_VERSION 12.0
61    #ARG ODOO_RELEASE=20190424
62   #ARG ODOO_SHA=3885be6791b9b8c2a74115299e57213c71db4363
63    #RUN set -x; \
64    #       curl -o odoo.deb -sSL http://nightly.odoo.com/${ODOO_VERSION}/nightly/deb/odoo_${ODOO_VERSION}.${ODOO_RELEASE}_all.deb \
65    #        && echo "${ODOO_SHA} odoo.deb" | sha1sum -c - \
66    #        && dpkg --force-depends -i odoo.deb \
67    #        && apt-get update \
68    #        && apt-get -y install -f --no-install-recommends \
69    #        && rm -rf /var/lib/apt/lists/* odoo.deb

---最后两行启动命令也注释或删除---
91    #ENTRYPOINT ["/entrypoint.sh"]
92    #CMD ["odoo"]
```
改成：
```
COPY ./wkhtmltox.deb /var/local/
COPY ./odoo12.deb /var/local/
```
>* 2、修改entrypoint.sh内容，odoo.conf不用修改（如需可按自己生产环境修改）
```
entrypoint.sh部分代码
# set the postgres database host, port, user and password according to the environment
# and pass them as arguments to the odoo process if not present in the config file
: ${HOST:=${DB_PORT_5432_TCP_ADDR:='db'}} #此处db对应上面postgres数据名db
: ${PORT:=${DB_PORT_5432_TCP_PORT:=5432}} #默认数据库端口
: ${USER:=${DB_ENV_POSTGRES_USER:=${POSTGRES_USER:='odoo'}}} #数据库用户名
: ${PASSWORD:=${DB_ENV_POSTGRES_PASSWORD:=${POSTGRES_PASSWORD:='odoo'}}} #数据库用户密码
```
---
##四、生成docker镜像文件(操作都在/var/local/目录下)
>* 1、使用Xshell来上传文件到服务器，在服务器安装lrzsz
```shell
安装 lrzsz
#apt-get update
#apt-get insatll lrzsz
```
>* 2、上传文件到/var/local/目录下
```shell
#cd /var/local/
#rz
然后选择本地上传的文件：
odoo12.0.deb 
wkhtmltox.deb
Dockerfile
entrypoint.sh
odoo.conf
```
>* 3、**提示1：** 因为配置文件都是在Windows系统里修改过，上传到linux系统需要用**dos2unix**转换一下文件才能执行，不然run镜像会报错
```shell
安装dos2unix
#apt-get install dos2unix

执行文件转换
#dos2unix Dockerfile
#dos2unix entrypoint.sh
#odoo.conf
```
>* 4、**提示2：**entrypoint.sh脚本为shell可执行文件，显示成绿色
```shell
#chmod +x entrypoint.sh
```
>* 5、制作镜像:  **提示3：** odoo12是镜像名字后面跟**空格** 和“**.**”，这个点代表当前目录，执行当前目录下的Dockerfile文件
```shell
#docker build -t odoo12 .
```
然后静候。。。知道镜像安装成功（部分过程会安装一些odoo系统的依赖包和postgres客户端）
___

#五、运行docker容器

>* 1、运行一个odoo容器：
**提示4：** 
IP映射 -p 8069:8069 主机端口：容器端口（p为小写）
连接数据库 --link db:db  连接到另一个容器可以写成--link=db
参考连接：
[docker run命令]: https://www.runoob.com/docker/docker-run-command.html
[odoo dockerhub]: https://hub.docker.com/_/odoo/
```shell
#docker run -p 8069:8069 --name odoo12 --link db:db -t odoo12
```
>* 2、**提示5：** 如需要把addons映射到本地目录进行应用开发部署可以加上：-v /本地目录:/mnt/extra-addons
```shell
#docker run -p 8069:8069 --name odoo12 -v /var/odoo12/addons:/mnt/extra-addons --link db:db -t odoo12
```
---
#六、进入容器安装wkhtmltox和odoo

>* 1、查看运行中的容器：**提示6：** COMMAND显示的是:bash，因为没有启动odoo服务，只是运行了debain
```shell
#docker ps -a
```
例如
```shell
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                     PORTS                              NAMES
99ece4411b8e        odoo12             "bash"                    2 days ago          Up 2 days                  0.0.0.0:8069->8069/tcp             odoo
be6af1fe4639        postgres:10.0       "docker-entrypoint.s…"   2 months ago        Up 3 days                  5432/tcp                           db 
```
>* 2、进入容器：
**提示7：**
 -it <CONTAINER ID或者NAMES> :-it后面跟容器id或者名字都可以
 -u root :以root用户身份进入容器，默认odoo用户权限不够
```shell
#docker exec -u root -it 99ece4411b8e /bin/bash
```
>* 3、进入到复制的文件目录进行安装：/var/local
**提示8：**
**先安装wkhtmltox，确定是安装成功后再回来删除文件**
1-安装wkhtmltox会提示安装失败，要依赖一些包如：libx11-6,libxcb1,libxextb,libxrender,xfonts-75,xfonts-base,libicu57,libuv1,,,等等注意提示信息，要安装的就先把这些安装完后再装一次；
2-如果sha1sum码错误或者提示版本不是debain格式，请检查你下载的sha1码和版本是否正确

```shell
#cd /var/local
检查文件
#echo '7e35a63f9db14f93ec7feeb0fce76b30c08f2057 wkhtmltox.deb' | sha1sum -c -
#dpkg --force-depends -i wkhtmltox.deb
#apt-get -y install -f --no-install-recommends 

安装成功后再执行这步，删除文件的不建议删除wkhtmltox.deb
rm -rf /var/lib/apt/lists/* wkhtmltox.deb
```
>* 4、安装odoo
**提示9：**
**确定系统安装成功能启动后再回来删除文件**
有时候会提示sha1码不正确，可以略过这步执行下一步的dpke解包安装

```shell
#echo '3885be6791b9b8c2a74115299e57213c71db4363 odoo.deb'| sha1sum -c - 
#dpkg --force-depends -i odoo12.deb 
#apt-get update 
#apt-get -y install -f --no-install-recommends 

安装成功后再执行这步，删除文件的不建议删除wkhtmltox.deb
#rm -rf /var/lib/apt/lists/* odoo.deb
```
>* 5、wkhtmltox和odoo安装成功后,回到根目录
**提示：**
如需优化配置请在容器的/etc/odoo/odoo.conf，修改odoo.conf文件
```shell
#cd /
#./entrypoint.sh odoo --http-port 8069 --longpolling-port 8071

成功运行odoo如下
2019-08-16 12:34:08,330 14 INFO ? odoo: Odoo version 12.0-20190423 
2019-08-16 12:34:08,331 14 INFO ? odoo: Using configuration file at /etc/odoo/odoo.conf 
2019-08-16 12:34:08,331 14 INFO ? odoo: addons paths: ['/mnt/extra-addons', '/usr/lib/python3/dist-packages/odoo/addons'] 
2019-08-16 12:34:08,332 14 INFO ? odoo: database: odoo@172.17.0.2:5432 
2019-08-16 12:34:08,459 14 INFO ? odoo.addons.base.models.ir_actions_report: Will use the Wkhtmltopdf binary at /usr/local/bin/wkhtmltopdf 
2019-08-16 12:34:08,595 14 INFO ? odoo.service.server: HTTP service (werkzeug) running on 57b3e0b6d24c:869
```

#大功告成，撒花！！！
#提示下一篇写配置nginx提高系统性能

> 备注：部分odoo相关的目录如下
> *  /var/lib/odoo/addons/12.0 addons目录
> * /mnt/extra-addons #addons模块应用目录，可设置映射到宿主机的目录上
> * /usr/lib/python3/dist-packages/odoo #odoo核心文件
> * /etc/init.d/odoo #odoo启动文件
> * ./entrypoint.sh #启动配置文件
> * /etc/odoo/odoo.conf #启动配置文件 