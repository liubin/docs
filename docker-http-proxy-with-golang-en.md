##Writing a simple HTTP proxy using Golang

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


The program checks the target URL on each HTTP request, if the target is not docker.io or any of its sub sites, it will return an error. The purpose is to protect the server from abusing or misusing which possibly leads to being banned by Great Firewall because it's used to visit websites which are unaccessible in mainland China.


##Writing the init.d script for Ubuntu


We have to write the init.d script file to let the proxy service run as a daemon in Ubuntu:


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


##Modify docker’s configuration to use HTTP proxy in Ubuntu


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


Or we can remove the comment on http_proxy in etc/default/docker:

```
# If you need Docker to use an HTTP proxy, it can also be specified here.
export http_proxy=http://192.241.209.203:8384/
```


Restart docker and it will download images through the proxy 192.241.209.203:8384


###Available proxies

```
http://192.241.209.203:8384
```


The [dockerboard](http://www.dockboard.org/) is now providing a proxy service (deployed on DigitalOcean’s VPS) for the domestic Chinese users to use and study [Docker](http://docker.io/). The proxy is not guaranteed to be available 24/7. If you encounter any problem, please contact the dockboard's [official Weibo account](http://weibo.com/dockboard) for consultation.


If you have any idle VPS and you would like to deploy a proxy to support Chinese docker lovers, we would be very grateful and happy to provide total technical support. To make the greatness, please contact: [meaglith.ma@aliyun.com](mailto:meaglith.ma@aliyun.com)
