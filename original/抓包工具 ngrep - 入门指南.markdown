# 抓包工具 ngrep - 入门指南

标签（空格分隔）： ngrep 抓包

---

## 安装

下载 ngrep 的源码包

```
wget http://prdownloads.sourceforge.net/ngrep/ngrep-1.45.tar.bz2?download
```
解压刚刚下载的源码包：

```
tar -xvf ngrep-1.45.tar.bz
```

编译安装：

```
cd ngrep-1.45
./configure
make && make install
```

> 注：在编译过程中，如果你的系统未安装 libpcap 就如下错误：


```
checking for a broken redhat glibc udphdr declaration... no
checking for a complete set of pcap headers... no
**!!! couldn't find a complete set of pcap headers**
```

解决办法，就是安装 `libpcap` 包，我的系统是 CentOS，所以安装方法是：

```
yum -y install libpcap*
```


## 使用示例





