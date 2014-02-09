![Why named docker?](http://docker.u.qiniudn.com/PPT_why_named_docker.jpg)

嗨，Docker 0.7 终于和各位见面了！希望大家喜欢。自 0.6.0 发布以来，经过无数的 bug 修复和使用细节的改进，新版本增加了 7 个主要功能：

- 功能1：标准 Linux 支持
- 功能2：存储驱动
- 功能3：离线传输
- 功能4：链接
- 功能5：容器命名
- 功能6：高级端口跳转
- 功能7：更高的软件质量

您可以在 [Git库更新日志](https://github.com/dotcloud/docker/blob/v0.7.0/CHANGELOG.md) 查看详情，也可以查看下文来了解各个新功能的详情。

###致谢！

130 名杰出的朋友参加了 Docker 的发布！部分人士列举如下：

Alex Larsson、Josh Poimboeuf、Lokesh Mandvekar、Vincent Batts、Adam Miller、Marek Goldmann and Matthew Miller from the Red Hat team、 Fred Kautz、Tianon “andrew” Gravi、Paul Nasrat、Sven Dowideit、James Turnbull、Edmund Wagner、David Calavera、Travis Cline、Gurjeet Singh、Justin Force、Johan Euphrosine、Jerome Petazzoni、Ole Reifschneider、Andy Rothfusz、Will Rouesnel、Greg Thornton、Scott Bessler、Todd Lunter、Vladimir Rutsky、Nicolas Dudebout、Jim Alateras、Roger Peppe

向以上各位，以及所有对 Docker 的发布伸出援助之手的人们致以谢意：谢谢你们！

###功能1：标准 Linux 支持

_0.7.0 版引入该功能，为此特别感谢 aufs 的作者们：Alex Larsson、（Red Hat 团队）、Junjiro R. Okajima。_

0.7.0 版引入了几个主要的新功能，其中最受期待的无疑是对标准 Linux 的支持。如今，使用 Docker 已经不再需要向 Linux 内核打补丁，感谢 Alex Larsson 为我们贡献了一个新的存储驱动（请参考下一个功能“存储驱动”）。这意味着 Docker 将会跳出内核限制，运行于所有主流 Linux 发行版中，包括 Fedora、RHEL、Ubuntu、Debian、Suse、 Gentoo、Arch 等）。[猛击此处](http://docs.docker.io/en/latest/installation/) 来查看你最喜爱的 Linux 发行版的 Docker 安装文档。
 
 
###功能2：存储驱动

_0.7.0 版本引入_

Docker 的一个核心功能是瞬间建出很多底层文件系统一致的 Docker 容器。Docker 在底层大量使用 AUFS（由 Junjiro R. Okajima 编写）作为 copy-on-write 的存储机制。AUFS 是一款超赞的软件，已经在过去的几年中安全地拷贝了无数个容器，其中相当大一部分用于生产环境。但非常可惜的是，AUFS 一直不是 Linux 标准内核的一部分，我们也不知道何时它才会加入到 Linux 标准内核中。

Docker 0.7 通过引入一个存储驱动 API 来处理不同驱动的传输，解决了这个问题。通过与 Alex Larsson 和出色的 Red Hat 团队合作，使用高度灵活的 LVM 快照技术实现了 copy-on-write，现在支持三种驱动：AUFS、VFS（使用简单目录和拷贝）和 DEVICEMAPPER 。实验性的 BERFS 驱动目前正在开发中，ZFS、Gluster、Ceph 也将推出。


Docker 的后台程序在启动时会根据情况自动选择一种合适的驱动。如果系统支持 AUFS，Docker 会自动选择 AUFS，升级后所有的容器和镜像都是可用的；如果系统不支持 AUFS，Docker 将使用 devicemapper 作为默认的驱动。同一台机器中的驱动之间不能共享数据，但驱动产生的镜像可以互相兼容，同时也兼容之前任何版本的 Docker。这意味着 [列表](http://index.docker.io) 中的所有镜像（包括 [此处](http://blog.docker.io/2013/11/introducing-trusted-builds/) 列出的受信任镜像），在任何已安装的 Docker 中都可以使用。

###功能3：离线传输

_0.6.7 版中引入，特别感谢：Frederic Kautz_

**离线传输**可以把容器镜像导出到本地的一个独立文件包，该文件包可以加载到任何其他的 Docker 后台中。加载后的镜像 100% 保留了原始镜像的所有内容，包括配置、创建日期以及构建日志等等。导出的文件就是一个普通的文件夹，可以用任何传输方式进行传输，例如 ftp 、磁盘、安装文件等等。

离线传输功能对于软件供应商特别有用，通常他们需要把软件做成封装好的软件包发给“企业级”用户。现在，通过 Docker 的离线传输，供应商可以把更新后的软件放入 Docker 容器再发给用户，既不影响软件发行机制，用户也不必担心安全问题。

正如 Github 企业版团队的 David Calavera 所说：“有了离线传输，用 Docker 容器构建内部部署[^1]的软件产品变得非常容易。无需注册，你的容器就可以部署到任何地方。”

###功能4：高级端口重定向

_0.6.5版中引入_

_注意：这项功能使安全性获得了两个突破性的小改进。本节最后有详细说明。_

新版Docker扩展了 `run` 命令的 `-p` 参数，用户可以更灵活的控制端口重定向。老版本中，主机上的所有网卡都只能自动重定向，新版中，用户可以自行指定要重定向的网卡。使用现有的命令就可以使用新的重定向功能。

例如：

- `p 8080` 把主机上的所有动态端口重定向到容器的 80 端口。
- `p 8080:8080` 把主机上所有以 8080 为静态端口的服务重定向到容器的8080端口。
- `p 127.0.0.1:80:80` 把主机上所有 localhost 的静态 80 端口的服务重定向到容器的 80 端口。

您也可以禁止主机上所有网卡的重定向，来有效的阻止外部对端口的访问。结合“链接”（请参见后面的“链接”一节），端口重定向技术会很有用：假设您要绕过公共网络，把一个未被保护的数据库端口直接暴露给一个应用程序容器（而不把端口开放到公共网络），只需要增加一个 **_-expose_** 参数，而不必指定 Dockerfile 。

0.7.0 在安全方面有了两个突破性的改变：

首先，`docker run` 命令默认不再对主机进行端口重定向了。这样做可以提高安全性：端口默认是私有的，用户可以指定 `-p` 参数对外开放接口。如果您现在默认把端口暴露到所有网卡，请注意，这样在 0.6.5 之后就行不通了。如果您还想以前一样开放端口，请用 `-p` 参数指定端口的值。

其次，我们不再推荐 [EXPOSE文档](http://docs.docker.io/en/latest/use/builder/#expose) 中提到的高级“ `<public>:<private>` ”语法。这项特殊语法允许用户在 Dockerfile 中提前指定网卡端口的重定向。我们认为现这种方式提前限制了系统管理员对主机端口重定向的控制，不利于开发和运维理的隔离，因此这种方式不再推荐使用，而常规的“ `EXPOSE <private>` ”语法不受影响。

例如：

- `EXPOSE 80` 暴露 tcp 的 80 端口，和老版本一样。

- `EXPOSE 80:80` 不推荐使用，这种方式实际上等价于  `EXPOSE 80` ，将产生警告，指定的公共端口 80 将会被无情的忽略。

非常抱歉，我们不得不对 Docker 进行了这样的"突破性"的改动。我们尽量减小改动带来的不便，也希望大家看到，这些改进是值得的！

###功能5：链接

_0.6.5 版引入_

链接使容器能够安全的互相访问。要关闭容器间的互访，可以设置启动参数 `-icc=false` ， icc 参数设置为 false 时， A 容器无法访问 B 容器，除非 B 容器通过链接的方式允许 A 的访问。这对容器的安全性有极大的好处。当两个容器连接在一起时，Docker 为两个容器创建一个父子关系。父容器可以通过名称、开放端口、ip 等环境参数来访问子容器的信息。

当 Docker 为两个容器创建链接时，会使用子容器暴露出来的端口为父容器创建一个安全的访问通道。如果一个数据库容器只暴露 8080 端口并关闭了容器间的互相通信，那么其连接的容器将只能访问 8080 端口。

例如：

当我们运行 WordPress 容器时，需要连接到数据库，如 MySQL 或 MariaDB 。使用链接，可以轻松访问后端数据库，无需修改 WordPress 站点的配置文件。

为了让 WordPress 容器可以访问不同的数据库，我们要用到别名，在本例中，我们用 ”**db**” 作为数据库的别名，这样，无论数据库容器叫什么名字，我们都可以通过一个统一的别名来访问数据库。使用下例中的两个数据库容器，我们可以将其和我们的 WordPress 容器连接起来。

要创建链接，只需在 `docker run` 命令中加入 `-link` 参数：

- `docker run -d -link mariadb:db user/wordpress` 或者

- `docker run -d -link mysql:db user/wordpress`

通过别名 db 链接到数据库容器后，我们可以查看 WordPress 容器的环境，并可以查看数据库的 ip 和端口。

`$DB_PORT_3306_TCP_PORT`

`$DB_PORT_3306_TCP_ADDR`

`-link` 中指定的参数将作为环境变量的前缀。另外，我们推荐您阅读 Docker 文档中的例子《 [Building a redis container to link as a child of our web application](http://docs.docker.io/en/latest/use/working_with_links_names/#links-service-discovery-for-docker) 》。

###功能6：容器命名

_0.6.5 版开始支持_

很高兴宣布，我们终于可以关闭 [issue#1](https://github.com/dotcloud/docker/issues/1) 了！现在，为 `docker run` 指定 `-name` 参数，就可以给你的容器起一个有意义的、容易记住的名字了。如果没有指定名称，Docker 会自动为容器生成一个名称。要把一个容器和另一个容器连接起来，您需要提供子容器的名称和别名，格式如下：

`-link child_name:alias`

多说无益，案例教学马上开始！你可以像下面这样，为两个数据库建立相应的别名，并用别名来运行对应的数据库。

- `docker run -d -name mariadb user/mariadb`

- `docker run -d -name mysql user/mysql`

然后，所有通过 id 访问容器的命令中，都可以用别名来代替 id

- `docker restart mariadb`

- `docker kill mysql`
 

###Quality功能7：软件质量

_特别感谢：Tianon Gravi、 Andy Rothfusz 和 Sven Dowideit_

应该说，“质量”不能算一个真正的功能，但是软件质量对我们来说太重要了！我们还是决定把“质量”加入到新功能列表中。0.7 版本中，我们特别关注了软件质量，并且在后续版本中，我们会更加注重软件品质。

Docker 的活跃发展令人难以置信，而且在短时间内获得了巨大成功。但是像 Docker 这样活跃的项目，持续的跟进是很困难的。尽管 Docker 已多次改进，但很多代码依然不成熟，或者尚未完成、或者需要重构甚至重写。Docker 的发布就像打开一扇防洪闸门，尽管经历6个月、2844个 pull request 的演进，代码问题依旧淹没在每天无数贡献和功能需求的洪流中。

我们花了一些时间来忍受并最终适应 Docker 疯狂的发展速度；最终，我们找到了解决途径。从 0.7 版本开始，我们将把质量放在第一位，并从多方面面严格把关：用户界面、测试覆盖率、代码的健壮性、易开发性、文档以及 API 的前后连贯性。我们深知，对于 Docker 的那些 bug ，目前还没有银弹[^2]。我们最初规划了宏大的“代码重写”计划，但最终我们决定放慢速度，稳扎稳打，一步步处理问题。日复一日，不断提交、再提交，我们会让 Docker 逐步发展起来。

对于大家的支持，我们心怀感激。无数人看到了 Docker 的潜力，面对 bug 和缺陷，他们会说”还好啦“，“ Docker 很有用，我们不介意它还有点儿粗糙”。我们希望有一天，人们会说：“我用 Docker，因为它很有用。而且，非常可靠！”

那一天，不远了！

感谢所有人，感谢你们的支持！

向0.8进军！

Solomon、 Michael、 Victor、 Guillaume 以及所有的 Docker 维护者们，加油！

#####[^1] 原文为" on-premises software "，是指运行在用户自己的生产、营业场所的软件产品，区别于部署在远程的软件。维基百科页面：http://en.wikipedia.org/wiki/On-premises_software （译者注）

#####[^2] 银弹（ _Silver Bullet_ ）的典故在 IT 经典图书《人月神话——软件项目管理之道》（ _The Mythical Man-Month: Essays on Software Engineering_ ）中提到过。在古老的传说中，只有银弹才能杀死巫士、巨人、狼人等。一般是指威力无穷或者效率高超的武器/技术，或者万灵药，或者是妙手回春之高招。（译者注）

----
这篇文章由[ Solomon Hykes ](http://blog.docker.io/author/solomon/)发表，点击[此处](http://blog.docker.io/2013/11/docker-0-7-docker-now-runs-on-any-linux-distribution/)可查阅原文。

The article was contributed by [Solomon Hykes](http://blog.docker.io/author/solomon/), click [here](http://blog.docker.io/2013/11/docker-0-7-docker-now-runs-on-any-linux-distribution/) to read the original publication.
