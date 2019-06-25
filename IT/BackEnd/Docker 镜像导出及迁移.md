# Docker 镜像导出及迁移
## Docker 目录分析
安装docker时, 默认的安装位置是/var/lib/docker;
```sh
$sudo ls /var/lib/docker/
aufs  containers  graph  init  linkgraph.db  repositories-aufstmp  trust  volumes
```
repositories-aufs: 记录了镜像名称以及对应的Id的json文件
graph: 保存的是下载镜像的元数据, 包括json和layersize, 其中json文件记录了相应的image id、依赖关系、创建时间和配置信息等; layersize为对应层的大小; 进入graph文件会发现下面包含着多个文件夹, 进入其中一个文件夹
```sh
root:/var/lib/docker/graph#cd 09694f91574ea3fca8558306c55abbbd47e01b8cb9ae782c66b9682a95c7f71e/
root:/var/lib/docker/graph/09694f91574ea3fca8558306c55abbbd47e01b8cb9ae782c66b9682a95c7f71e#ls json  layersize
```
可以看到json和layersize文件;
json文件内容中layersize可能显示为0, 这一层为0, 不表示镜像大小就是0; Graph存储镜像时, 是分层存储的, graph目录下多出的文件夹其实都对应一个layer; 这些layer都与我们的镜像名命名的layer有关联, 关系就记录在json文件中;  从这个json文件中, 可以看到起父镜像或者上一层镜像就是……, graph目录下也存储着这一层的信息, 再往下看, 可以看到层次关系; 在graph这个目录里并没有找到我们想找到的镜像内容存放地; graph目录下只是一些镜像相关的信息数据; image应该包含一个类似linux的文件系统才对;
containers: 这个下面记录的是容器相关的信息, 每运行一个容器, 就在这个目录下面生成一个容器Id对应的子目录;
init: 保存的是docker init相关的信息;
tmp: 是一个空目录, 具体起什么作用还不清楚;
volumes: 与docker的数据券相关;
aufs目录: mnt是aufs的挂载目录, diff是实际数据来源, 也就是我们image实际存储的地方, 包括只读层和可读写层, 所有这些层最终都一起挂载到mmt所在的目录; layers下为每层依赖有关的描述文件;
在diff、mnt、layers下面有6个文件或子目录, 但是从graph目录下看我们的image应该是4层, 为什么会多出来2个呢; 仔细观察多出来的来个文件或者子目录, 会发现其名称和容器Id一致, 且有一个包含init; 其实, 在容器启动之前, mnt和layers都是空目录, diff下面也只有graph目录下我们看到的镜像层对应的4个目录; 在Docker利用image启动一个容器时, 会在aufs下新建容器id对应的文件和子目录, 同时在镜像的可读层执行新建一个可读写的layer; 至于id-init文件或者子目录记录的都是与容器内环境相关的信息, 与镜像无关;
既然我们现在已经知道了镜像实际是存储在diff目录下的, 那么我们就看看diff目录下各个子目录中的内容; 依照镜像的层次关系查看; 最终我们看到linux一样的文件系统; 已表示image确实是存储在diff目录下的;
综上, docker image最终是存储在在/var/lib/docker/aufs/diff中的, 同时在graph中有有关进行的记录; 在容器启动时, diff下的可读层image和新增的可读写层“容器Id”都将挂载到mmt目录下以容器id命名的子目录下;

## Docker 镜像导入导出
导出容器 docker export
导出容器快照到本地文件
```sh
$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                    PORTS               NAMES
7691a814370e        ubuntu:14.04        "/bin/bash"         36 hours ago        Exited (0) 21 hours ago                       test
$ sudo docker export 7691a814370e > ubuntu.tar
```
导入容器快照docker import
从容器快照文件中再导入为镜像
```sh
$ cat ubuntu.tar | sudo docker import - test/ubuntu:v1.0
$ sudo docker images
REPOSITORY          TAG                 IMAGE ID            CREATED              VIRTUAL SIZE
test/ubuntu         v1.0                9d37a6082e97        About a minute ago   171.3 MB
```
通过指定 URL 或者某个目录来导入
导入远程的包:This will create a new untagged image.
```sh
$ sudo docker import http://example.com/exampleimage.tgz
```
导入本地文件:Import to docker via pipe and STDIN.
```sh
$ cat exampleimage.tgz | sudo docker import- exampleimagelocal:new
```
导入本地目录:
```sh
$ sudo tar -c dir/ | sudo docker import - docker-image-name
```
Note: 用户既可以使用 docker load 来导入镜像存储文件到本地镜像库, 也可以使用 docker import 来导入一个容器快照到本地镜像库; 这两者的区别在于容器快照文件将丢弃所有的历史记录和元数据信息(即仅保存容器当时的快照状态), 而镜像存储文件将保存完整记录, 体积也要大; 此外, 从容器快照文件导入时可以重新指定标签等元数据信息;
皮皮blog


Docker镜像迁移
下面的$DOCKER_DIR代表/data/docker
Note: 如果迁移到windows下的磁盘如/media/pika/files/mine/env/docker会出现问题:
```sh
sudo service docker restart
stop: Unknown job: docker
start: Unknown job: Docker
[Starting Docker as Daemon on Ubuntu]
```
在#下可以运行, 但是docker run还是运行不了, 会出问题！
解决: 重装docker
[docker的概念及安装:Docker卸载和重装]
[ubuntu14.04 docker容器无法通过service管理, 求解]
所以一般没必要的话就不要到处乱迁移了!

Docker的镜像以及一些数据都是在/var/lib/docker目录下, 它占用的是Linux的系统分区, 也就是下面的/dev/sda2,当有多个镜像时, 空间可能不足, 我们可以把docker的数据挂载到数据盘, 如$DOCKER_DIR目录下;
```sh
$df -lhT
Filesystem     Type      Size  Used Avail Use% Mounted on
udev           devtmpfs  3.9G  4.0K  3.9G   1% /dev
tmpfs          tmpfs     781M  1.3M  780M   1% /run
/dev/sda2      ext4      148G   14G  127G  10% /
none           tmpfs     4.0K     0  4.0K   0% /sys/fs/cgroup
none           tmpfs     5.0M     0  5.0M   0% /run/lock
none           tmpfs     3.9G   36M  3.8G   1% /run/shm
none           tmpfs     100M   72K  100M   1% /run/user
/dev/sda6      fuseblk   110G   99G   11G  91% /media/pika/files
/dev/sda5      fuseblk    98G   78G   20G  80% /media/pika/softwares
```
1 停止docker #service docker stop
2 在数据分区中建立要挂载的目录 #mkdir -p $DOCKER_DIR
3 使用rsync工具同步 rsync -aXS /var/lib/docker/. $DOCKER_DIR, 这可能需要花费的较长的时间, 取决于/var/lib/docker的大小
4 修改fstab文件中把下面一行添加到fstab里, 将新位置挂载到 /var/lib/docker
```sh
$vim /etc/fstab
...
$DOCKER_DIR /var/lib/docker none bind 0 0
```
5 重新挂载:
```sh
mount –a
```
6 使用下面的命令检查一下
###df -h /var/lib/docker/
```sh
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda6       110G  100G  9.9G  92% /var/lib/docker
```
7 进入Container查看我们的空间
```sh
bash-4.1# df -lhT
```
8 宿主机中的分区大小信息:
```sh
root:/home/pika#df -lhT
Filesystem     Type      Size  Used Avail Use% Mounted on
udev           devtmpfs  3.9G  4.0K  3.9G   1% /dev
tmpfs          tmpfs     781M  1.3M  780M   1% /run
/dev/sda2      ext4      148G   14G  127G  10% /
none           tmpfs     4.0K     0  4.0K   0% /sys/fs/cgroup
none           tmpfs     5.0M     0  5.0M   0% /run/lock
none           tmpfs     3.9G   51M  3.8G   2% /run/shm
none           tmpfs     100M   44K  100M   1% /run/user
/dev/sda6      fuseblk   110G  100G  9.9G  92% /media/pika/files
/dev/sda5      fuseblk    98G   78G   20G  80% /media/pika/softwares
```
