##使用 Golang 编写一个简单的 HTTP 代理

[goproxy](https://github.com/elazarl/goproxy) 是一个轻量级的 HTTP 代理库，可以很容易的编写一个仅供 [Docker](http://docker.io) 使用的代理 [程序](https://github.com/dockboard/docker-proxy) 。

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

##编写 Ubuntu init.d 的启动脚本

为了保证服务在后台运行，编写 Ubuntu init.d 的启动脚本管理 Proxy 服务。
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

修改 /etc/init/docker.conf 加入 http proxy 的环境变量。

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

```
# If you need Docker to use an HTTP proxy, it can also be specified here.
export http_proxy=http://192.241.209.203:8384/
```

重启 Docker 的服务后下载 images 就会通过  192.241.209.203:8384 这个代理进行访问。


## 可用的代理地址

```
http://192.241.209.203:8384
```

目前 [dockboard](http://www.dockboard.org) 为了方便国内用户使用和学习 [docker](http://docker.io) 技术，使用 [DigitalOcean](http://www.digitalocean.com) 的 VPS 架设了一个代理。代理地址不保证 7x24 小时稳定，如果遇到网络问题，请咨询我们的微博账号：[Docker中文社区](http://weibo.com/dockboard)。

如果开发者有闲置的 VPS 愿意架设 Docker 的代理为社区提供支持，我们非常愿意提供技术帮助，请联络：meaglith.ma@aliyun.com。
