# 实验五 blobFS原理和源码分析

## 实验目的

1. 学习和理解基于spdk的文件系统原理和实现。

## 实验内容

1. 学习BlobFS基本原理

2. 在Nvme上创建BlobFS

3. 通过Fuse挂载BlobFS

## 实验代码及结果

启动服务器，初始化环境

```
./scripts/setup.sh
```

直接安装·`libfuse3`会报错

```
sudo apt install libfuse3
```

![image-20221227174330532](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221227174330532.png)

需要安装`dev`版的

```
sudo apt install libfuse3-dev
```

执行编译

```
./configure --with-fuse &&  make 
```

生成配置文件

```
./scripts/gen_nvme.sh --json-with-subsystems > ./test/blobfs/nvme.json
```



创建blobfs

```
sudo ./mkfs/mkfs ./nvme.json Nvme0n1
```

![image-20221227173920573](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221227173920573.png)





创建挂载`fuse`的目录

```
sudo mkdir /mnt/fuse
```

运行`fuse`

![image-20221227173809615](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20221227173809615.png)