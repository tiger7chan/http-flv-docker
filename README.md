# http-flv-docker

基于Apline镜像、nginx-http-flv-module模块制作的docker镜像，优点主要是控制了大小，目前制作好的镜像只有16M。

Nginx-http-flv-module版本为1.2.7，https://github.com/winshining/nginx-http-flv-module，感谢作者[winshining](https://github.com/winshining)。

Nginx的版本为1.17.8。

本文主要参考了https://www.cnblogs.com/zhujingzhi/p/9742085.html，感谢作者！！

### Dockerfile说明

#### 编译阶段

```
# 指定Nginx版本
ARG NGINX_VERSION=1.17.8

# 指定nginx的位置
ARG NGINX_PATH=/usr/local/nginx
ARG NGINX_CONF=/usr/local/nginx/conf
```

这里指定了nginx的版本，以及我们编译时nginx的安装位置和配置文件默认位置。

```
    # 下载nginx和最新的http-flv-module
    && curl -fSL http://nginx.org/download/nginx-$NGINX_VERSION.tar.gz -o nginx.tar.gz \
    && git clone https://github.com/winshining/nginx-http-flv-module.git --depth 1 \
```

这里在dockerfile文件中下载nginx和http-flv模块，也可以自己下载好后，拷贝进去。

```
&& ./configure $CONFIG --with-debug \
```

目前编译的是debug版本，稳定后可以去掉--with-debug。

```
    # 获取nginx运行需要的库（下面的打包阶段无法获取，除非写死）
    && runDeps="$( \
        scanelf --needed --nobanner $NGINX_PATH/sbin/nginx $NGINX_PATH/modules/*.so /tmp/envsubst \
            | awk '{ gsub(/,/, "\nso:", $2); print "so:" $2 }' \
            | sort -u \
            | xargs -r apk info --installed \
            | sort -u \
    )" \
    # 保存到文件，给打包阶段使用
    && echo $runDeps >> $NGINX_PATH/nginx.depends
```

为了在尽量缩小打包镜像，这里获取我们程序运行需要的库，保存下来，打包阶段可以从文件中读取出来，然后安装。

#### 打包阶段

```
# 定义一个环境变量，方便后面运行时可以进行替换
ENV NGINX_CONF /usr/local/nginx/conf/nginx.conf
```

打包阶段这里定义成ENV，是为了在后面运行docker镜像时，可以修改成自己的配置文件。

```
# 从build中拷贝编译好的文件
COPY --from=Builder $NGINX_PATH $NGINX_PATH
# 下面链接stderr和stdout需要的文件
COPY --from=Builder /var/log/nginx /var/log/nginx
```

这里从前面的阶段拷贝出编译好的文件。

```
# 将目录下的文件copy到镜像中(默认的配置文件)
COPY nginx.conf $NGINX_CONF
COPY rtmp.conf $NGINX_PATH/conf/rtmp.conf
# 拷贝启动命令
COPY start.sh /etc/start.sh
```

拷贝我们默认配置，同时将启动脚本也拷贝进去，脚本里面主要是指定了加载的配置文件（用环境变量表示，以及关闭daemon模式），如下：

```
/usr/local/nginx/sbin/nginx -g 'daemon off;' -c $NGINX_CONF
```

```
# 安装启动依赖项
RUN apk update \
    # 从编译阶段读取需要的库
    && runDeps="$( cat $NGINX_PATH/nginx.depends )" \
    #&& echo $runDeps \
    # 通过上面查找nginx运行需要的库
    && apk add --no-cache --virtual .nginx-rundeps $runDeps \
```

这里根据编译阶段获取的依赖库信息，进行安装。

#### 制作镜像

镜像的制作是根据Docker v17.05版本后开始支持多阶段构建 (`multistage builds`)来实现的，所以docker的版本必须大于17.05.

> docker build -t docker.io/tiger7/http-flv:latest <font color='red'>--target=nginx-flv</font> .

这里就是只构建我们的打包阶段的镜像。

### 运行说明

>  docker run -d -p 80:80 -p 1935:1935 --env NGINX_CONF=/etc/nginx/nginx.conf --name=http-flv -v /opt/nginx-http-flv/:/etc/nginx -v /opt/nginx-http-flv/log/:/var/log/nginx/ tiger7/http-flv:1.0.0.0

#### 映射端口

```
-p 80:80 -p 1935:1935
```

这里将80（http）和1935（rtmp）端口映射出来。

#### 设置环境变量

```
--env NGINX_CONF=/etc/nginx/nginx.conf
```

这里指定容器运行时的nginx配置文件路径的环境变量，替换默认的值（/usr/local/nginx/conf/nginx.conf）。

#### 挂载卷（Volume）

```
-v /opt/nginx-http-flv/:/etc/nginx -v /opt/nginx-http-flv/log/:/var/log/nginx/
```

这里挂载了2个，一个是我们的配置文件，因为前面环境变量指定了/etc/nginx路径，所以把我们本地/opt/nginx-http-flv/的配置文件挂载进去；

另一是挂载nginx在运行时的日志路径，这样我们就可以不用进入容器，直接在本机的/opt/nginx-http-flv/log/位置就可以读取到日志内容。

### 运行截图

#### 成功运行

 ![image](https://github.com/tiger7chan/http-flv-docker/raw/master/img/run.jpeg)

#### 推流和拉流成功

 ![image](https://github.com/tiger7chan/http-flv-docker/raw/master/img/run2.jpeg)