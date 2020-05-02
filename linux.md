启动docker

sudo systemctl docker start

sudo systemctl daemon-reload

sudo systemctl docker restart



运行镜像

sudo docker run -it --rm ubuntu:16.04 bash

-it：i 交互式  t 终端

--rm  退出后终止运行  （避免浪费空间）

sudo docker ps 所有正在访问的docker容器

sudo docker ps -a 所有正在运行容器

sudo docker ls 镜像列表

sudo docker rm 运行容器ID //关闭容器

exit 退出

sudo docker image prune 删除虚悬镜像<none>

sudo docker image rm 镜像名或镜像ID