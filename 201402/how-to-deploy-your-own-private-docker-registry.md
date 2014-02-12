![enter image description here][1]

#How to Deploy your own Private Docker Registry
#部署自己的私有Docker Registry

Matthew Fisher, January 21, 2014

Matthew Fisher, 2014年1月21日

This blog post shows how you can deploy your own private Docker Registry behind your firewall with SSL encryption and HTTP authentication. A Docker Registry is a service which you can push Docker images to for storage and sharing. We will be installing the registry on Ubuntu, but it should work on any operating system that supports upstart. SSL encryption and HTTP basic authentication will be managed by Nginx, which will be a proxy server in front of the Docker Registry. Upstart will manage the gunicorn processes that will run the registry. We will also be using a LRU cache to reduce roundtrips to the storage backend. For this cache we will use Redis.

这篇博客讨论了如何部署一个带SSL加密、HTTP验证并有防火墙防护的私有Docker Registry。[Docker Registry](https://github.com/dotcloud/docker-registry)是一个存储和分享Docker镜像的服务。本例中，我们使用的操作系统是Ubuntu，当然，任何支持[upstart](http://upstart.ubuntu.com/)的系统都应该都是可以的。我们用Nginx作为Docker Registry的前端代理服务器，同时也用Nginx完成SSL加密和基本的HTTP验证。我们用gunicorn运行Docker Registry并用Upstart管理gunicorn。另外，我们还会用Redis实现一个LRU(Least Recently Used，近期最少使用算法)缓存机制来减少Docker Registry和硬盘之间的数据存取。


##Why do you need a Docker Registry?
##为什么需要Docker Registry?
When you create new Docker images for use in your environment - whether that'd be a Redis server, a Hipache daemon, or an IRC logbot - you're going to want to store the images somewhere safe. Maybe you're working on a project where you also want to create a Docker image with Jenkins or Buildbot on each commit, bag and tag (read: docker commit && docker tag) the image, and then push that to the registry. But what if your code is proprietary, and you don't want to push that image to the public registry? Docker Inc. has already thought of that for you, and has created the docker-registry project. This project will allow you to push your own images to your own in-house registry. Woo!

当你在自己的环境中创建Docker镜像的时候，无论是装[Redis](http://redis.io/)，[Hipache](https://github.com/dotcloud/hipache)，还是IRC协议的[logbot](https://github.com/dannvix/Logbot)，你都希望可以把镜像存到一个安全的地方。也许你项目中的Docker镜像需要[安装Jenkins](http://buildbot.net/)，或者每次commit都跑一遍[Buildbo](http://buildbot.net/)t，又或者给镜像打上bag和tag（相关阅读：docker commit，docker tag），再发送到Docker Registry。可是如果镜像中的代码是私有的，你不想把镜像放到公共的Docker Registry上呢？Docker公司已经想到了这一点，并因此建立了[docker-registry](https://github.com/dotcloud/docker-registry)项目。docker-registry允许你把自己的镜像push到自己的registry中，酷！

***还没看bag tag的内容***

If you want to kick the proverbial tires, you can test the docker registry:

如果你想感受一下docker registry，可以用公共的registry来试试：

    $ docker pull samalba/docker-registry
    $ docker run -d -p 5000:5000 samalba/docker-registry
    $ # let's pull a sample image (or make one ourselves)
    $ docker pull busybox
    $ docker tag busybox localhost:5000/busybox
    $ docker push localhost:5000/busybox


This is great to get started working with the registry for testing, but this will be using plain HTTP. Anyone can push to your server as long as they have endpoint access, which is not good. Let's get started with setting up our own private registry for internal use.

对于registry入门，这个例子很有用，但是例子中仅用了一个简单的HTTP服务。任何知道服务器地址的人都可以随意push镜像，这不是个好方案。下面我们来建立自己的私有registry以供内部使用。


##Planning our Deployment
##准备自己的部署方案

Before we spawn an Ubuntu server to start deploying the registry, let's consider some things…
我们要创建一个Ubuntu服务器来部署registry，在此之前，我们先考虑几件事情...

###What Storage Backend?
###用什么作为后台存储？

What storage backend do we want to use? Here's a short list of the supported backends for the registry:
local: use the local filesystem
s3: store inside an Amazon S3 bucket
swift: store inside a Openstack Swift container
glance: use Openstack's Glance project
elliptics: use the Elliptics key-value store

Sidenote: I created the backend for Openstack Swift. If you find any bugs with it, please feel free to file a bug on the registry's github page.

我们用什么来做后台存储呢？请看下面几种存储方案：

 - local：用本地存储
 - s3：存到Amazon S3的bucket
 - swift：存到OpenStack的Swift容器
 - glance：使用OpenStack的Glance项目
 - elliptics：使用Elliptics的键值存储方案

这些方案的python脚本在[这里](https://github.com/dotcloud/docker-registry/tree/master/lib/storage)，大家可以参考。

注：Openstack Swift这个方案是我自己写的，如果发现bug，欢迎在docker registry的github主页提出。

###Hosted or In-House Server?
###托管服务器还是用自己搭建服务器？

Where do we want to host our docker registry? Do we want to use our own Openstack cluster, Amazon Web Services, Rackspace, or our own bare metal servers? Any option will work for us!

我们要把docker registry服务部署到哪里呢？用自己的OpenStack集群？Amazon的网络服务？还是Rackspace？或者自己购买服务器？答案是：用什么都行！

One thing to consider when using cloud-hosted infrastructure is the advantage of using an external volume for your data. This gives you control over managing your own backups, which is a huge win for us.

使用云服务，我们可以使用可扩展的存储空间，便于我们管理自己的备份，非常方便。


###What Operating System?
###用什么操作系统？

Since the docker registry is a python project, it's ridiculously simple to port over to other operating systems. You can quite easily write up a systemd config file, or launch it as a Windows Service. Because we will be installing it on Ubuntu, we will be using upstart to manage our gunicorn processes.

I will be demonstrating the deployment process using the local storage backend, where all of our assets will be held on our own hardware. We have an internal Openstack cluster over here in our Vancouver office (we love Openstack!), so we will use that for our hosting solution. docker-internal.example.com will be the fully qualified domain name, and we will be using Ubuntu's 12.04.3 cloud image as the server.
All right. Let's get down to deploying!

docker registry是用python写的，所以把它导入到各种操作系统中真是太简单了。你可以轻轻松松的写一个[systemd配置文件](https://wiki.archlinux.org/index.php/systemd#Writing_custom_.service_files)，或者把它做成[Widnows服务](http://en.wikipedia.org/wiki/Windows_service)。本例中，我们在Ubuntu上安装docker registry，因此，我们用[upstart](http://upstart.ubuntu.com/)来管理gunicorn进程。

接下来的部署工作都在本地硬盘里进行。我们自己的办公室里有一个内部的Openstack集群（我爱Openstack！），因此我们用它来作服务器托管，域名用“docker-internal.example.com”，服务器系统则用[Ubuntu Cloud image](http://cloud-images.ubuntu.com/releases/12.04.3/release/)，版本为12.04.3。

万事俱备，开工！

###Boot the Server
###启动服务器

First, let's boot up a server. Since I'll be using our internal Openstack cluster, I'll just use the nova client to boot up my server. If you're following this post line by line, here are the credentials you'll need to set up:

首先，启动服务器。因为我是用的内部Openstack，我用[nova客户端](https://github.com/openstack/python-novaclient)来启动就可以了。如果你按照本例来操作，请在.bashrc文件中设置下面列出的验证信息：


	$ cat ~/.bashrc
	[...]
	export OS_AUTH_URL=http://******/v2.0
	export OS_TENANT_ID=******
	export OS_TENANT_NAME="******"
	export OS_USERNAME=******
	export OS_PASSWORD="******"
	[...]

Once you set that up, test by running:

设置完成后，请用下面的命令测试一下：

	$ sudo pip install python-novaclient
	$ nova list

Before we boot the server, let's upload the Ubuntu cloud image, as well as your own SSH key...

启动服务器之前，我们先上传Ubuntu cloud image和自己的SSH key文件：

	$ nova keypair-add --pub-key ~/.ssh/id_rsa.pub bacongobbler
	$ sudo pip install python-glanceclient
	$ glance image-create --name ubuntu-12.04.3-server-cloudimg-amd64 --disk-format qcow2 --container-format bare --location http://cloud-images.ubuntu.com/releases/12.04.3/release/ubuntu-12.04-server-cloudimg-amd64-disk1.img

And create a security group that allows external access to port 80 and 443...

再创建一个安全组，来允许外部对80和443端口的访问：

	$ nova secgroup-create web-server "security group for standard web servers"
	$ nova secgroup-add-rule web-server tcp 80 80 0.0.0.0/0
	$ nova secgroup-add-rule web-server tcp 443 443 0.0.0.0/0

now we will create the volume, which will be 512GB in size. We will be using this to store our docker images:

现在，我们来创建一个512G的分区，用于存储我们的docker镜像：

	$ nova volume-create 512 --display-name docker-internal

***根据上下文，nova volume-create命令应该是分配了一个分区，因为后面用mount把这个512GB的分区挂在到了一个目录里***
Finally, we can boot the server!

最后，启动服务器吧！

	$ nova boot docker-internal --image ubuntu-12.04.3-server-cloudimg-amd64 --flavor m1.medium --security-groups web-server --key-name bacongobbler
	$ # do some grepping for the volume ID
	$ VOLUME_ID=$(nova volume-list | grep docker-internal | awk '{print $2}')
	$ nova volume-attach docker-internal $VOLUME_ID /dev/vdb
	$ nova floating-ip-list
	+----------------+--------------------------------------+---------------+------+
	| Ip             | Instance Id                          | Fixed Ip      | Pool |
	+----------------+--------------------------------------+---------------+------+
	| 192.168.68.222 | 79caf450-7b23-46bd-839a-abec7408a2c0 | 192.168.32.26 | nova |
	| 192.168.68.224 | a10cb949-09b6-4533-9733-860a5f8fdff4 | 192.168.32.19 | nova |
	| 192.168.68.225 | None                                 | None          | nova |
	| 192.168.68.236 | None                                 | None          | nova |
	| 192.168.68.237 | dc835a69-2894-4278-aebe-4f9ca6363724 | 192.168.32.12 | nova |
	| 192.168.68.238 | 4a8835b6-a318-44b5-897d-2320977cfe01 | 192.168.32.20 | nova |
	| 192.168.68.239 | afde96f2-9bac-441a-a0c7-589ace2ac6b9 | 192.168.32.15 | nova |
	| 192.168.68.246 | 00ceedf4-8d85-4ea5-8f42-78a1ab521a62 | 192.168.32.13 | nova |
	| 192.168.68.250 | c1ef2314-6067-464d-85ec-de2a26a80f3e | 192.168.32.4  | nova |
	| 10.3.4.1       | 192dadcc-e786-4366-8091-2e9a364a65cf | 192.168.32.17 | nova |
	+----------------+--------------------------------------+---------------+------+
	$ nova add-floating-ip docker-internal 192.168.68.236

Wait a couple seconds, and set up your domain registrar to map the subdomain docker-internal to this IP address. After that, run:

等一小会儿，并把子域名docker-internal绑定到当前的IP，然后用SSH登陆：

	$ ssh ubuntu@docker-internal.example.com


Hooray!

哦耶！搞定！

###Deploy and configure the registry
###部署和配置registry

Now that we have our server, let's install some packages to get started.

我们已经有自己的服务器了，下面我们来装几个必要软件吧。

    # 安装软件之前，我们先更新一下软件源列表，然后重启
    ubuntu@docker-internal:~$ sudo apt-get update
    ubuntu@docker-internal:~$ sudo apt-get upgrade
    ubuntu@docker-internal:~$ sudo reboot now

    #登陆到docker registry
    $ ssh ubuntu@docker-internal.example.com
    
    # 切换到root用户
    ubuntu@docker-internal:~$ sudo su
    
    # 安装nginx-extras包，我们要用其中的chunkin模块
    root@docker-internal:~# apt-get install git nginx-extras
    
    # 安装apache2-utils包，这样我们就可以用htpasswd命令来设置密码
    root@docker-internal:~# apt-get install apache2-utils
    
    # 安装一些必要的依赖
    root@docker-internal:~# apt-get install build-essential libevent-dev libssl-dev liblzma-dev python-dev python-pip
    
    # 安装redis来实现我们的LRU缓存策略
    root@docker-internal:~# apt-get install redis-server
    root@docker-internal:~# apt-get clean

Now that we have that out of the way, let's install the docker registry:

必要的软件都装好了，下面我们就来安装docker registry：

    root@docker-internal:~# git clone https://github.com/dotcloud/docker-registry.git /opt/docker-registry
    root@docker-internal:~# cd /opt/docker-registry
    
    # 切换到最新的registry版本 
    root@docker-internal:~# git checkout 0.6.3
    
    # 创建日志路径
    root@docker-internal:~# mkdir -p /var/log/docker-registry
    
    # 安装pip包
    root@docker-internal:~# pip install -r requirements.txt
    root@docker-internal:~# cp config/config_sample.yml

***这里作者没有指定要拷贝到哪里***

If you've done this all correctly, we should now be able to test the registry will run with:

如果一切顺利，我们现在应该可以用下面的命令来测试一下docker registry了：

    root@docker-internal:~# ./wsgi.py
    2014-01-13 23:38:38,470 INFO:  * Running on http://0.0.0.0:5000/
    2014-01-13 23:38:38,470 INFO:  * Restarting with reloader

If you see this, you're doing great! Now, we just need to set up a couple more things. Remember that volume we mapped to this server earlier? Let's set that up now:

如果你看到的结果和上面一样，那么，恭喜你，成功了！接下来我们需要设置一些选项，记得我们之前分配给docker registry的分区吗？现在我们就来挂在这个分区：

    root@docker-internal:~# mkdir -p /data/registry
    root@docker-internal:~# mkfs.ext4 /dev/vdb
    root@docker-internal:~# mount /dev/vdb /data/registry

And now, let's edit our configuration file for the docker registry. You can use http://uuidgenerator.net/ to generate a secret key:

现在，我们来编辑docker registry的配置文件，我们用http://uuidgenerator.net/在线生成密钥：

    root@docker-internal:~# cat << EOF > /opt/docker-registry/config/config.yml
    # The 'common' part is automatically included (and possibly overriden by
    # all other flavors)
    common:
        # Set a random string here
        secret_key: REPLACEME
        standalone: true
     # This is the default configuration when no flavor is specified
    dev:
        storage: local
        storage_path: /tmp/registry
        loglevel: debug
    # To specify another flavor, set the environment variable SETTINGS_FLAVOR
    # $ export SETTINGS_FLAVOR=prod
    prod:
        storage: local
        storage_path: /data/registry
        loglevel: info
        # Enabling LRU cache for small files. This speeds up read/write on
        # small files when using a remote storage backend (like S3).
        cache:
            host: localhost
            port: 6379
        cache_lru:
            host: localhost
            port: 6379
    EOF

Once this is done, set up an upstart job for the registry:

然后为docker registry设置一个upstart作业：

    root@docker-internal:~# cat << EOF > /etc/init/docker-registry.conf
    description "Docker Registry"
    version "0.6.3"
    author "Docker, Inc."
    
    start on runlevel [2345]
    stop on runlevel [016]
    
    respawn
    respawn limit 10 5
    
    # set environment variables
    env REGISTRY_HOME=/opt/docker-registry
    env SETTINGS_FLAVOR=prod
    
    script
    cd $REGISTRY_HOME
    exec gunicorn -k gevent --max-requests 100 --graceful-timeout 3600 -t 3600 -b 0.0.0.0:5000 -w 8 --access-logfile /var/log/docker-registry/access.log --error-logfile /var/log/docker-registry/server.log wsgi:application
    end script
    EOF

And then start it with:

用下面的命令启动registry的作业：

    root@docker-internal:~# start docker-registry
    docker-registry start/running, process 10872

Verify that it's running by checking:

用下面的命令检查registry的作业是否运行：

    root@docker-internal:~# cat /var/log/docker-registry/server.log
    2014-01-14 00:33:44 [15051] [INFO] Starting gunicorn 18.0
    2014-01-14 00:33:44 [15051] [INFO] Listening at: http://0.0.0.0:5000 (15051)
    2014-01-14 00:33:44 [15051] [INFO] Using worker: gevent
    2014-01-14 00:33:44 [15056] [INFO] Booting worker with pid: 15056
    2014-01-14 00:33:44 [15057] [INFO] Booting worker with pid: 15057
    2014-01-14 00:33:44 [15062] [INFO] Booting worker with pid: 15062
    2014-01-14 00:33:45 [15067] [INFO] Booting worker with pid: 15067
    2014-01-14 00:33:45 [15068] [INFO] Booting worker with pid: 15068
    2014-01-14 00:33:45 [15069] [INFO] Booting worker with pid: 15069
    2014-01-14 00:33:45 [15070] [INFO] Booting worker with pid: 15070
    2014-01-14 00:33:45 [15071] [INFO] Booting worker with pid: 15071

Now for nginx:

接下来，我们设置nginx：

    root@docker-internal:~# rm /etc/nginx/sites-enabled/default
    root@docker-internal:~# cat << EOF > /etc/nginx/sites-enabled/docker-registry
` nginx的配置：`

    upstream docker-registry {
      server localhost:5000;
    }
    
    server {
      listen 443;
      server_name docker-internal.example.com;
    
      ssl on;
      ssl_certificate /etc/ssl/certs/docker-registry.crt;
      ssl_certificate_key /etc/ssl/private/docker-registry.key;
    
      proxy_set_header Host             $http_host;   # required for docker client's sake
      proxy_set_header X-Real-IP        $remote_addr; # pass on real client's IP
      proxy_set_header Authorization    ""; # see https://github.com/dotcloud/docker-registry/issues/170
    
      client_max_body_size 0; # disable any limits to avoid HTTP 413 for large image uploads
       
      # required to avoid HTTP 411: see Issue #1486 (https://github.com/dotcloud/docker/issues/1486)
      chunkin on;
      error_page 411 = @my_411_error;
      location @my_411_error {
        chunkin_resume;
      }
    
      location / {
        auth_basic              "Restricted";
        auth_basic_user_file    docker-registry.htpasswd;
    
        proxy_pass http://docker-registry;
        proxy_set_header Host $host;
        proxy_read_timeout 900;
      }
    
      location /_ping {
        auth_basic off;
        proxy_pass http://docker-registry;
      }
    
      location /v1/_ping {
        auth_basic off;
        proxy_pass http://docker-registry;
      }
    }
    EOF
` `    
    
    root@docker-internal:~# service nginx restart

And the associated htpasswd file (ensuring to replace USERNAME and PASSWORD):

别忘了在htpasswd文件里设置账号密码：

    root@docker-internal:~# htpasswd -bc /etc/nginx/docker-registry.htpasswd USERNAME PASSWORD


Let's install an SSL key onto the server. In this example, I am assuming that someone has handed you an SSL key that has been signed and verified by a certificate authority. This SSL key could be for either 'docker-internal.example.com' or '*.example.com':

我们还要在服务器上安装一个SSL密钥。本例中，假设我们已经有认证机构颁发的SSL证书了，SSL授权给'docker-internal.example.com'或者'*.example.com'，用下面的命令来安装SSL密钥：

    root@docker-internal:~# mv server.key /etc/ssl/private/docker-registry.key
    root@docker-internal:~# mv server.crt /etc/ssl/certs/docker-registry.crt

If you don't have the cash to fork out for a new SSL key, or you are just testing out this process before deploying, you can install a self-signed SSL key by following the instructions from Akadia:

如果你不打算花钱去搞一个认证机构授权的SSL密钥，或者你只是练习着部署docker registry，那么你也可以按照[Akadia的教程](http://www.akadia.com/services/ssh_test_certificate.html)装一个自己授权的SSL key，如下：

    root@docker-internal:~# openssl genrsa -des3 -out server.key 1024
    root@docker-internal:~# openssl req -new -key server.key -out server.csr
    root@docker-internal:~# cp server.key server.key.org
    root@docker-internal:~# openssl rsa -in server.key.org -out server.key
    root@docker-internal:~# openssl x509 -req -days 3650 -in server.csr -signkey server.key -out server.crt

Please note that using self-signed certificates is currently waiting on pull request #2687. You will have to sit tight until it is merged into master, or you can try building Docker from source.

请注意，现在官方的docker还不能用自授权的证书，要等到编号[`#2687`](https://github.com/dotcloud/docker/pull/2687)的pull request合并到官方master分支后才能使用。或者，你也可以试着修改docker的源代码来让它支持自授权证书。

###Verification
###测试

Finally, let's test this:

最后，我们来测试一下自己的docker registry：

    root@docker-internal:~# exit
    ubuntu@docker-internal:~$ exit
    $ curl -u bacongobbler:******* https://docker-internal.example.com
    "docker-registry server (prod)"
    $ docker login https://docker-internal.example.com
    Login against server at https://docker-internal.example.com/v1/
    Username (): bacongobbler
    Login Succeeded
    $ docker pull busybox
    Pulling repository busybox
    e9aa60c60128: Download complete 
    $ docker tag busybox docker-internal.example.com/busybox
    $ docker push docker-internal.example.com/busybox
    The push refers to a repository [docker-internal.example.com/busybox] (len: 1)
    Sending image list
    Pushing repository docker-internal.example.com/busybox (1 tags)
    Pushing tags for rev [e9aa60c60128] on {https://docker-internal.example.com/v1/repositories/busybox/tags/latest}
    e9aa60c60128: Image already pushed, skipping

And we're done! One docker registry, deployed on Openstack and ready to go.

完成！现在我们在Openstack上部署了一个docker registry，随时可用！

###What's Next?
###下一步

So, after deploying the registry, what are some things that we can do to improve or enhance this project? I can think of a couple:

部署好docker registry后，我们还可以进一步让它跑的更好，比如：

- set up email notifications on registry exceptions
- ship the logs off to logstash or some other log aggregation tool
- deploy the registry on CentOS or RHEL
- do some benchmarking to see how well the registry scales

- 当docker registry崩溃的时候，发出邮件通知
- 用[logstash](http://logstash.net/)或其他日志软件来管理日志
- 把docker registry部署到CentOS或者RHEL
- 做一些测试，看看docker registry的性能如何

What other suggestions can you think of? Leave a comment below!

这只是我想到的一些，如果你有什么好点子，请在下面留言！

Here at ActiveState, we're proud to say that we are actively using the Docker project in Stackato v3. If you missed Phil's amazing post on everything that's in Stackato v3, please take a look at his post, as well as the section about where Docker fits in with Stackato.

在ActiveState，我们在[Stackato v3](http://www.activestate.com/stackato)项目中大量的使用了Docker。Phil在博客中全面的介绍了Stackat v3，如果你还没来得及看，一定要去拜读他的[大作](http://www.activestate.com/blog/2013/11/technical-look-stackato-v30-beta)，特别是其中[Stackato在哪些地方适合用Docker](http://www.activestate.com/blog/2013/11/technical-look-stackato-v30-beta#docker)这一节。

图片来自：[Glyn Lowe Photoworks](http://www.flickr.com/photos/glynlowe/)




  [1]: http://www.activestate.com/sites/default/files/images/blog/ship-with-containers.jpg