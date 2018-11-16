# docker灵活的构建php环境
    使用docker搭建灵活的线上php环境 有时候你可能不太需要一些别人已经集成了的包或者镜像 
    我们就可以使用以下方式自己动手逐一构建自己所需要的环境结构 并在最后实现一键自动化部署 
    一步一步点亮docker技能树
                        ##         .
                  ## ## ##        ==
               ## ## ## ## ##    ===
           /"""""""""""""""""\___/ ===
      ~~~ {~~ ~~~~ ~~~ ~~~~ ~~~ ~ /  ===- ~~~
           \______ o           __/
             \    \         __/
              \____\_______/


### \* 首先git拉取[server](https://github.com/ydtg1993/server.git)项目 放到服务器根目录（到后面你也可以构建自己风格的环境结构）


##  (一阶)使用docker逐一构建

#### 1.下载镜像
`sudo docker pull php:7.2-fpm`      冒号后选择版本

`sudo docker pull nginx`

`sudo docker pull mysql:8.0`    不需要本地数据库可忽略

`sudo docker pull redis:3.2`    不需要本地redis可忽略

`sudo docker images`  查看已下载的所有镜像

#### 2.下载完成镜像后运行容器 [以下采用--link方式创建容器 注意创建顺序]
    注：
    -i 表示允许我们对容器进行操作
    -t 表示在新容器内指定一个为终端
    -d 表示容器在后台执行
    /bin/bash 这将在容器内启动bash shell
    -p 为容器和宿主机创建端口映射
    --name 为容器指定一个名字
    -v 将容器内路径挂载到宿主机路径
    --privileged=true 给容器特权,在挂载目录后容器可以访问目录以下的文件或者目录
    --link可以用来链接2个容器，使得源容器（被链接的容器）和接收容器（主动去链接的容器）之间可以互相通信，解除了容器之间通信对容器IP的依赖
    
    
<运行mysql容器>

`sudo docker run --name mydb -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:8.0`

    注：-MYSQL_ROOT_PASSWORD=123456 给mysql设置初始密码
    如果不需要搭建本地数据库直接下一步


 <运行redis容器>

`sudo docker run --name myredis -p 6379:6379 -d redis:3.2` 

    注: 如果不需要搭建本地redis直接下一步

 <运行php容器>

`sudo docker run -d -p 9000:9000 --name myphp -v /server/www:/var/www/html -v /server/php:/usr/local/etc/php --link mydb:mydb --link myredis:myredis --privileged=true  php:7.2-fpm`

    注： 如果不需要搭建本地数据库或者redis可以省去--link mydb:mydb --link myredis:myredis


<运行nginx容器> 

`sudo docker run --name mynginx -d -p 80:80 -v /server/www:/usr/share/nginx/html -v /server/nginx:/etc/nginx -v /server/logs/nginx.logs:/var/log/nginx --link myphp:myphp --privileged=true  nginx`
    
    注：
    -v语句冒号后是容器内的路径 我将nginx的网页项目目录 配置目录 日志目录分别挂载到了我事先准备好的/server目录下
    --link myphp:myphp 将nginx容器和php容器连接 通过别名myphp就不再需要去指定myphp容器的ip了 


`sudo docker ps  -a`    查看所有容器运行成功 这里环境也就基本搭建完成了

###### 挂载目录后就可以不用进入容器中修改配置，直接在对应挂载目录下改配置文件 修改nginx配置到 /server/nginx/conf.d/Default.conf
![default.conf](https://github.com/ydtg1993/server/blob/master/nginx_default_explain.PNG)
    
    
#### 3.PHP扩展库安装

`sudo docker exec -ti myphp  /bin/bash`     首先进入容器

`docker-php-ext-install pdo pdo_mysql`      安装pdo_mysql扩展

`docker-php-ext-install  redis`

    注: 此时报错提示redis.so 因为一些扩展并不包含在 PHP 源码文件中

###### 方法一：

`tar zxvf /server/php_lib/redis-4.1.0.tgz`      解压已经下载好的redis扩展包

`sudo docker cp /server/php_lib/redis-4.1.0 myphp:/usr/src/php/ext/redis`       将扩展放到容器中 再执行安装

    注：
    直接将扩展包放到容器ext目录里可能会报错Error: No such container:path: myphp:/usr/src/php/ext
    你可以多开一个服务器窗口 进入php容器中执行docker-php-ext-install  redis此时报错error: /usr/src/php/ext/redis does not exist
    保持这个状态然后在你的第一个服务器窗口执行上条命令就成功了 
    (具体原因未知但确实要执行一次docker-php-ext-install命令 容器中才会有/usr/src/php/ext这个目录)
 
 ###### 方法二：
 
    注: 
    官方推荐使用 PECL（PHP 的扩展库仓库，通过 PEAR 打包）。用 pecl install 安装扩展，然后再用官方提供的 docker-php-ext-enable 
    快捷脚本来启用扩展
 
`pecl install redis && docker-php-ext-enable redis`

`docker restart myphp`      装完扩展 退出容器 重启容器

#### \*其它命令
`docker stop $(docker ps -q)`   停止所有容器

`docker rm $(docker ps -aq)`    删除所有容器

`docker inspect myphp`      查看容器配置信息

#### \*构筑自己的目录结构
    你也可以构建自己所要的server目录结构
    例如: 创建一个临时容器 sudo docker run --name mydb -p 3306:3306 -it -d mysql:8.0
    然后进入到容器中查看自己所要的目录地址 例如: /etc/mysql/conf.d 退出容器 
    拷贝容器中所要的目录结构到宿主机 例如: sudo docker cp mydb:/etc/mysql /server/mysql
    删除容器 创建新容器时就可以挂载该目录了 方便以后对容器的配置文件的修改
    sudo docker run --name mydb -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -v /server/mysql:/etc/mysql -d mysql:8.0
   
   
##  (二阶)docker-compose自动化构建

    完成以上步骤你就已经初步了解了docker的基本容器操作
    docker-compose是编排容器的。例如，你有一个php镜像，一个mysql镜像，一个nginx镜像。如果没有docker-compose，
    那么每次启动的时候，你需要敲各个容器的启动参数，环境变量，容器命名，指定不同容器的链接参数等等一系列的操作，
    相当繁琐。而用了docker-composer之后，你就可以把这些命令一次性写在docker-composer.yml文件中，以后每次启动
    这一整个环境（含3个容器）的时候，你只要敲一个docker-composer up命令就ok了

 ####  1.安装docker-compose
    curl -L https://github.com/docker/compose/releases/download/1.8.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
    
    chmod +x /usr/local/bin/docker-compose
    
    docker-compose --version

#### 2.一键部署环境
    /server/compose/docker-compose.yml已经配置好了 直接输入命令即可
    
`cd /server/compose`
    
`docker-compose up -d`

![docker_yml](https://github.com/ydtg1993/server/blob/master/docker_yml_explain.PNG)

    对比上面运行容器命令来看docker_yml的配置结构和语义就一目了然了 
   
   
##  (三阶)dokcer-compose和dockerfile 完整构建

    用了docker-compose实现一键式操作 但问题是PHP的扩展库还是得自己单独装 所以这里需要用到Dockerfile来构建自定义容器镜像
    实现真正的一键完成
    
    目录:
    server__|                       |__docker-compose.yml
            |__compose.dockerfiles__| 
                                    |__mysql__|Dockerfile 这里设置我们自定的dockerfile来构建mysql镜像
                                    |           
                                    |           
                                    |__nginx__|Dockerfile 这里设置我们自定的dockerfile来构建nginx镜像
                                    |          
                                    |
                                    |__php__|Dockerfile 这里设置我们自定的dockerfile来构建php镜像
                                    |       |
                                    |
                                    |__redis__|Dockerfile 这里设置我们自定的dockerfile来构建redis镜像
                                              | 
    
![dockerfile](https://github.com/ydtg1993/server/blob/master/docker_file_explain.PNG)
   
    自定义php的dockerfile构建自定义镜像同时安装扩展  完成了所有dockerfile配置后 docker-compose.yml文件就不需要
    再用官方镜像image:php-fpm:7.2 而是直接build：./php 直接引用目录配置好的Dockerfile
       
`cd /server/compose.dockerfiles`
    
`docker-compose up -d`

    以上就是docker所有的环境配置方式
    
---

###### 最后推荐一个远程docker客户端 portainer 方便远程管理你的线上docker容器

`docker volume create portainer_data`

`docker run -d -p 9010:9000 -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer`   portainer内部使用9000端口 因为php已经使用了9000 所以映射到宿主机时端口要改成其他未占用端口

直接访问http://[服务器ip]/#/init/admin

    

    
    
    
