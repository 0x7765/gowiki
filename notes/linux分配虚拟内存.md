# linux 扩大虚拟内存

**场景:低配的服务器，需要更大的内存才能跑程序**

linux 实现虚拟内存的两种方式

- [ ]  交换分区 `swap分区`
- [ ]  交换文件 `swapfile`

## 交换文件

1. 创建空的文件`sudo touch /root/swapfile`
2. 转换 `sudo dd if=/dev/zero of=/root/swapfile bs=1M count=2048` 
   1. if:`input file` 输入文件
   2. of:`output file` 输出文件
   3. bs:`block size` 块大小
   4. count: 块的数量
3. 格式化交换文件 `sudo mkswap /root/swapfile`
4. 启用交换文件 `sudo swapon /root/swapfile`
5. 开机加载虚拟内存 
   1. `sudo vim /etc/fstab` 
   2. add `/root/swapfile swap             swap    defaults          0       0`
6. reboot 生效

## 删除交换文件

1. `sudo vim /etc/fstab` delete `/root/swapfile swap             swap    defaults          0       0`
2. 关闭交换文件`sudo swapoff /root/swapfile`
3. 删除交换文件 `sudo rm -fr /root/swapfile`

## 交换分区

- [ ]  todo

