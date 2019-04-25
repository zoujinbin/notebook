## 参考资料

[官网](https://www.keepalived.org/)  
[官方英文文档](https://www.keepalived.org/manpage.html)  
[推荐博客]()

### 简介


### 安装

1. 软硬件环境

| Software | Version |
| :---: | :---: |
| Ubuntu | 16.04 |
| Keepalived | 2.0.15 |

2. 安装依赖包

```
apt-get install libssl-dev
```

3. 编译安装

```
wget https://www.keepalived.org/software/keepalived-2.0.15.tar.gz
tar -zxvf keepalived-2.0.15.tar.gz 
cd keepalived-2.0.15/
./configure --prefix=/opt/keepalived
make && make install
```

### 配置