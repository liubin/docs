##使用 Golang 编写一个简单的 HTTP 代理
##Writing a simple HTTP proxy using Golang

[goproxy](https://github.com/elazarl/goproxy) 是一个轻量级的 HTTP 代理库，可以很容易的编写一个仅供 [Docker](http://docker.io) 使用的代理 [程序](https://github.com/dockboard/docker-proxy) 。

The [goproxy](https://github.com/elazarl/goproxy) is a light weight http proxy library, which allows you to easily write a [proxy program](https://github.com/dockboard/docker-proxy) for [Docker](http://docker.io/), here is the code:


```
package main

import (
  "github.com/elazarl/goproxy"
  "net/http"
  "regexp"
)

func main() {
  proxy := goproxy.NewProxyHttpServer()
  proxy.OnRequest().DoFunc(
    func(r *http.Request, ctx *goproxy.ProxyCtx) (*http.Request, *http.Response) {
      match, _ := regexp.MatchString("^*.docker.io$", r.URL.Host)
      if match == false {
        return r, goproxy.NewResponse(r, goproxy.ContentTypeText, http.StatusForbidden,
          "The proxy is used exclusively to download docker image, please don't abuse it for any purpose.")
      } else {
        return r, nil
      }
    })
  proxy.Verbose = false
  http.ListenAndServe(":8384", proxy)
}
```

程序在每次 HTTP 的请求过程中都检测目标地址的 URL，如果不是 docker.io 或子站会返回一个错误信息。这为了避免使用代理访问其它的网站，致使服务器被墙。



This program checks the target URL on each HTTP request, if the target is not docker.io or any of its sub sites, it will return an error. Thus avoid the server being ‘great fire walled’ for visiting too many web sites which ‘should not’ be accessible in mainland China.



##编写 Ubuntu init.d 的启动脚本
##Writing the init.d script for Ubuntu

为了保证服务在后台运行，编写 Ubuntu init.d 的启动脚本管理 Proxy 服务。


We’ll have to write the init.d script file to let the proxy service run as a daemon in Ubuntu:


```
#!/bin/sh

### BEGIN INIT INFO
# Provides:           dockboard.org
# Required-Start:     $syslog $remote_fs
# Required-Stop:      $syslog $remote_fs
# Default-Start:      2 3 4 5
# Default-Stop:       0 1 6
# Short-Description:  Create lightweight http proxy for Docker daemon.
# Description:
#       Author: Meaglith Ma
# 	    Email: genedna@gmail.com
#       Website: http://www.dockboard.org 
### END INIT INFO

BASE=$(basename $0)

DOCKER_PROXY=/usr/bin/$BASE
DOCKER_PROXY_PIDFILE=/var/run/$BASE.pid
DOCKER_PROXY_OPTS=

PATH=/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/sbin:/usr/local/bin

# Check docker is present
if [ ! -x $DOCKER_PROXY ]; then
	exit 1
fi

fail_unless_root() {
	if [ "$(id -u)" != '0' ]; then
		exit 1
	fi
}

case "$1" in
	start)
		fail_unless_root
		start-stop-daemon --start --background \
			--exec "$DOCKER_PROXY" \
			--pidfile "$DOCKER_PROXY_PIDFILE" \
			-- -d -p "$DOCKER_PROXY_PIDFILE" \
			$DOCKER_PROXY_OPTS
		;;

	stop)
		fail_unless_root
		start-stop-daemon --stop \
			--pidfile "$DOCKER_PROXY_PIDFILE"
		log_end_msg $?
		;;

	restart)
		fail_unless_root
		docker_pid=`cat "$DOCKER_PROXY_PIDFILE" 2>/dev/null`
		[ -n "$docker_pid" ] \
			&& ps -p $docker_pid > /dev/null 2>&1 \
			&& $0 stop
		$0 start
		;;

	force-reload)
		fail_unless_root
		$0 restart
		;;

	status)
		status_of_proc -p "$DOCKER_PROXY_PIDFILE" "$DOCKER" docker
		;;

	*)
		echo "Usage: $0 {start|stop|restart|status}"
		exit 1
		;;
esac

exit 0
```

##Ubuntu 中修改 Docker 的配置文件使用 HTTP Proxy

##Modify docker’s configuration to use HTTP proxy in Ubuntu

修改 /etc/init/docker.conf 加入 http proxy 的环境变量。

Let’s add the http proxy environment variables to /etc/init /docker.conf:

```
description "Docker daemon"

start on filesystem and started lxc-net
stop on runlevel [!2345]
 
respawn
 
env HTTP_PROXY="http://192.241.209.203:8384"
 
script
  DOCKER=/usr/bin/$UPSTART_JOB
  DOCKER_OPTS=
  if [ -f /etc/default/$UPSTART_JOB ]; then
    . /etc/default/$UPSTART_JOB
  fi
  "$DOCKER" -d $DOCKER_OPTS
end script
```

或者是修改 /etc/default/docker 文件，取消注释 http_proxy 的部分。

Or we can remove the comment on http_proxy in etc/default/docker:

```
# If you need Docker to use an HTTP proxy, it can also be specified here.
export http_proxy=http://192.241.209.203:8384/
```

重启 Docker 的服务后下载 images 就会通过  192.241.209.203:8384 这个代理进行访问。

Restart docker and it will download images throw the proxy 192.241.209.203:8384

## 可用的代理地址
###Available proxies

```
http://192.241.209.203:8384
```

目前 [dockboard](http://www.dockboard.org) 为了方便国内用户使用和学习 [docker](http://docker.io) 技术，使用 [DigitalOcean](http://www.digitalocean.com) 的 VPS 架设了一个代理。代理地址不保证 7x24 小时稳定，如果遇到网络问题，请咨询我们的微博账号：[Docker中文社区](http://weibo.com/dockboard)。

The [dockerboard](http://www.dockboard.org/) is now providing a proxy service (deployed on DigitalOcean’s VPS) for the domestic Chinese users to use and study [Docker](http://docker.io/). The proxy is not guaranteed to be available 24/7. If you encounter any problem, please contact the dockboard's [official Weibo account](http://weibo.com/dockboard) for consultation.

如果开发者有闲置的 VPS 愿意架设 Docker 的代理为社区提供支持，我们非常愿意提供技术帮助，请联络：meaglith.ma@aliyun.com。

If you have any idle VPS and you would like to deploy a proxy to support Chinese docker lovers, we would be very grateful and we are happy to provide total technical support. To make the greatness, please contact: [meaglith.ma@aliyun.com](mailto:meaglith.ma@aliyun.com)
