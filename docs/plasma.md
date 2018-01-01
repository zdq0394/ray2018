# The Plasma Object Store
Plasma是一个高性能的共享内存对象存储，最初由Ray project开发，现在在Apache Arrow下。
## Using Plasma with Huge Pages
Linux系统下，可以使用`huge pages`提高Plasma object store的写吞吐量。
首先需要创建一个文件系统，并激活`huge pages`：
```sh
sudo mkdir -p /mnt/hugepages
gid=`id -g`
uid=`id -u`
sudo mount -t hugetlbfs -o uid=$uid -o gid=$gid none /mnt/hugepages
sudo bash -c "echo $gid > /proc/sys/vm/hugetlb_shm_group"
sudo bash -c "echo 20000 > /proc/sys/vm/nr_hugepages"
```
创建文件系统需要root权限；运行Plasma object store不需要root权限。

可通过如下命令在单一节点上启动Ray，使用`huge pages`:
``` python
ray.init(huge_pages=True, plasma_directory="/mnt/hugepages")
```

在集群模式下，可以通过如下命令启动Ray：
* --huge-pages
* --plasma-directory=/mnt/hugepages

