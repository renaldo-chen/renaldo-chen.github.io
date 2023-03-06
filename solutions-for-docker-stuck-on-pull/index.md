# docker镜像下载慢/失败的解决方法 


## 方案一：修改镜像源

这个方法在csdn、知乎那些地方说得太多了，我相信大部分人搜索到的第一个方法就是这个。尽管如此我还是重新说一下方法。

编辑配置文件，如果文件不存在，以下命令会自动创建。

```bash
sudo nano /etc/docker/daemon.json
```

将配置信息粘贴到配置文件中，配置信息为 json 格式，可以根据实际需要设置多个国内的镜像服务器。
分别对应Github、阿里云、docker官方中国镜像、网易云、和百度云的源。

需要特别注意的是，第二个阿里云的源需要自己到【阿里云】的【容器镜像服务】处获取。

实践表明其实前两个源就够用了，阿里云的源相当不错，而Github的源在部分情况下比阿里云的源好用。

```json
{
    "registry-mirrors": [
    "https://ghcr.io",
    "https://5yt01a9i.mirror.aliyuncs.com",
    "https://registry.docker-cn.com",
    "http://hub-mirror.c.163.com",
    "https://mirror.baidubce.com"
    ]
}
```

刷新docker源.

```bash
sudo systemctl daemon-reload 
sudo systemctl restart docker
```

查看docker源是否更新成功。

```bash
sudo docker info
```

## 方案二：有网络的地方下载后上传/同步到目标服务器

这也是我写这篇博客的目的。

一方面如果你在玩一些很新的东西，比如说Firefly III的docker镜像，我在国内服务器上尝试了所有我知道的源后都没有顺利pull下来。另一方面如果你的机器不方便联网，那么可以试试下面说的本地转存的方法。

假定国内服务器为Server A，国外服务器为Server B。理论上Server B为你自己的电脑也行，只要能连接到Docker的网站，但我没有测试过。

我在DigitalOcean上还有一台学生机，而这台Server B连接Docker相当顺畅。首先在Server B上使用下列命令拉取镜像，查看镜像ID。

```bash
# Server B

export Docker_Image=******/****:***
# export Docker_Image=fireflyiii/core:version-5.7.18
docker pull ${Docker_Image}
docker image ls
```

得到镜像`id`后，将镜像保存为tar文件。其中`<id>`替换为镜像id，`<dir>/<file_name>.tar`替换为保存在Server B上的路径。

```bash
# Server B
docker save -o <dir>/<file_name>.tar <id>
```

使用SFTP工具将Serve B上的tar文件转移到Server A上，在Server A上解压读取docker镜像。

```bash
# Server A
docker load < <dir>/<file_name>.tar
```

用docker image ls查看镜像id。

```bash
# Server A
docker image ls
```

为刚刚导入的docker镜像添加镜像名和标签。其中`<id>`替换为镜像id。

```bash
# Server A
export Docker_Image=******/****:***
# export Docker_Image=fireflyiii/core:version-5.7.18
docker tag <id> ${Docker_Image}
```

后续在Server A上就可以正常使用docker-compose等命令读取和操作镜像了。

