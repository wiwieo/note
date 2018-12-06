# 使用docker搭建redis集群

引用自[Creating Redis Cluster using Docker](https://medium.com/commencis/creating-redis-cluster-using-docker-67f65545796d)

在宿主机上无法使用IP连接docker容器（dial timeout），还没找到对应的设置方法，此处使用的方法是将程序打包成docker，使其和redis集群使用docker同样的网络，就可以连接了
