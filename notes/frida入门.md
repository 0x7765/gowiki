# 搭建frida环境

## mac端

- `pip3 install -U frida`
- `pip3 install -U frida-tools`

## android端

1. 下载frida-server 打开 [https://github.com/frida/frida/releases](https://github.com/frida/frida/releases) 选择符合自己cpu 架构的版本 
2. push server 到远端 `adb push frida-server-12.11.9-android-arm64 /data/local/tmp`
3. 更改权限 `adb shell "chmod 755 /data/local/tmp/frida-server-12.11.9-android-arm64"`
4. 运行

```bash
adb shell
su
cd /data/local/tmp
./frida-server-12.11.9-android-x86 -D -d .
```

# 说明

## 查看手机的cpu架构

`getprop ro.product.cpu.abi`

![3.png](https://i.loli.net/2020/08/22/jXOCiRhkm47Zr9o.png)

## 执行可能的报错

`Unable to load SELinux policy from the kernel: Failed to open file ?/sys/fs/selinux/policy?: Permission denied`

- 尝试关闭`selinux` `setenforce 0`
- 还是没有权限 需要root手机 可以刷入 `magisk manager`  解决

# 验证

![Untitled 1.png](https://i.loli.net/2020/08/22/lZXsTtfz97uRiIj.png)

# hello frida

**目标: hook activity 的 onResume 方法并打印堆栈**
![carbon.png](https://i.loli.net/2020/08/22/dthqFCvIZYbRTLf.png)
输出结果

![2.png](https://i.loli.net/2020/08/22/pUR5xNDiygHL1jJ.png)
