---
title: "HGX A100 8-GPU install"
weight: 31
---

## 安装 CUDA

{{< tip >}}
没啥好说的，按照[这份文档](https://docs.nvidia.com/datacenter/tesla/tesla-installation-notes/index.html#ubuntu-lts)里面的代码直接跑就行了，因为里面一些链接可能会变动，这里就不再写一遍了。
{{< /tip >}}

特别注意的的是，在第五步 `sudo apt-get -y install cuda-drivers` 时，建议加上 `--no-install-recommends` ，这样可以避免安装图形化界面。

## 安装 Fabric Manager

{{< tip >}}
[参考文档](https://docs.nvidia.com/datacenter/tesla/pdf/fabric-manager-user-guide.pdf)
{{< /tip >}}

### 简介

NVIDIA DGX™ A100 和 NVIDIA HGX™ A100 8-GPU 服务器系统使用 NVIDIA® NVLink® 交换机 (NVIDIA® NVSwitch™)，可通过 NVLink 结构实现全对全通信。DGX A100 和 HGX A100 8-GPU 系统均由一个 GPU 基板、八个 NVIDIA A100 GPU 和六个 NVSwitch 组成。每个 A100 GPU 都有两个 NVLink 连接到同一 GPU 基板上的每个 NVSwitch。此外，可以将两个 GPU 基板连接在一起以构建一个 16 GPU 系统。在两个 GPU 基板之间，唯一的 NVLink 连接在 NVSwitch 之间，每个交换机从一个 GPU 基板连接到另一个 GPU 基板上的单个 NVSwitch，总共有 16 个 NVLink 连接。

### 安装

跳转到参考文档的 2.6 章节。

安装 cuda-drivers-fabricmanager

```shell
sudo apt-get install cuda-drivers-fabricmanager
```

启动服务并设置自动启动

```shell
sudo systemctl start nvidia-fabricmanager
sudo systemctl enable nvidia-fabricmanager
```

查看服务状态

```shell
sudo systemctl status nvidia-fabricmanager
```


## A100 GPU 切割实战

{{< tip >}}
此章节是聚焦于通过 `nvidia-smi mig` 来对 A100 进行 GPU 切割以及计算实例的创建与删除。
{{< /tip >}}

### Before all

如果刚上手玩 A100 的 GPU 切割，最好先要先检查是否有残存的计算实例以及切割出来的 GPU。

```shell
# 列出计算实例
sudo nvidia-smi mig -lci
# 列出 GPU 实例
sudo nvidia-smi mig -lgi
```

如果有建议确保数据保存的情况下删除它们

```shell
# 删除计算实例
sudo nvidia-smi mig -dci
# 删除 GPU 实例
sudo nvidia-smi mig -dgi
```

如果出现错误，错误提示被占用，建议使用 `sudo fuser -v /dev/nvidia*` 命令来列出被占用的进程，请在确保安全的情况下停掉他们。

### 开始切割

使用 `-lgip` 列出支持的 GPU 实例配置文件。

```shell
sudo nvidia-smi mig -lgip
```

可能会得到如下结果

```
+-----------------------------------------------------------------------------+
| GPU instance profiles:                                                      |
| GPU   Name             ID    Instances   Memory     P2P    SM    DEC   ENC  |
|                              Free/Total   GiB              CE    JPEG  OFA  |
|=============================================================================|
|   0  MIG 1g.10gb       19     7/7        9.50       No     14     0     0   |
|                                                             1     0     0   |
+-----------------------------------------------------------------------------+
|   0  MIG 1g.10gb+me    20     1/1        9.50       No     14     1     0   |
|                                                             1     1     1   |
+-----------------------------------------------------------------------------+
|   0  MIG 2g.20gb       14     3/3        19.50      No     28     1     0   |
|                                                             2     0     0   |
+-----------------------------------------------------------------------------+
|   0  MIG 3g.40gb        9     2/2        39.25      No     42     2     0   |
|                                                             3     0     0   |
+-----------------------------------------------------------------------------+
|   0  MIG 4g.40gb        5     1/1        39.25      No     56     2     0   |
|                                                             4     0     0   |
+-----------------------------------------------------------------------------+
|   0  MIG 7g.80gb        0     1/1        78.75      No     98     5     0   |
|                                                             7     1     1   |
+-----------------------------------------------------------------------------+
|   1  MIG 1g.10gb       19     7/7        9.50       No     14     0     0   |
|                                                             1     0     0   |
......
......
```

这里面可以看到各个 GPU 的几个切割方案下的剩余容量，Nvidia A100 最多支持切割为 7 块 GPU 实例。比如在 A100-80G 中支持如下切割方案：

| GPU  | Name           | ID   | Instances Free/Total | Memory  GiB | P2P  | SM CE | DEC JPEG | ENC OFA |
| :--- | :------------- | :--- | :------------------- | :---------- | :--- | :---- | :------- | :------ |
| 0    | MIG 1g.10gb    | 19   | 7/7                  | 9.50        | No   | 14    | 0        | 0       |
|      |                |      |                      |             |      | 1     | 0        | 0       |
| 0    | MIG 1g.10gb+me | 20   | 1/1                  | 9.50        | No   | 14    | 1        | 0       |
|      |                |      |                      |             |      | 1     | 1        | 1       |
| 0    | MIG 2g.20gb    | 14   | 3/3                  | 19.50       | No   | 28    | 1        | 0       |
|      |                |      |                      |             |      | 2     | 0        | 0       |
| 0    | MIG 3g.40gb    | 9    | 2/2                  | 39.25       | No   | 42    | 2        | 0       |
|      |                |      |                      |             |      | 3     | 0        | 0       |
| 0    | MIG 4g.40gb    | 5    | 1/1                  | 39.25       | No   | 56    | 2        | 0       |
|      |                |      |                      |             |      | 4     | 0        | 0       |
| 0    | MIG 7g.80gb    | 0    | 1/1                  | 78.75       | No   | 98    | 5        | 0       |
|      |                |      |                      |             |      | 7     | 1        | 1       |

可以看到 Nvidia 提供了 6 种切割方案，记住这里的 ID ，之后创建的时候要用到。

开始切割，如果我们要在 0 号 GPU 上切割出 7 个 1g.10gb 的实例，我们需要如下命令

```shell
sudo nvidia-smi mig -cgi 19,19,19,19,19,19,19 -i 0 -C 
```

具体的指令解析，详情看[这份文档](../nvidia-smi/)。

删除实例

```shell
sudo nvidia-smi mig -dci [device ID] -gi [GPU ID] -i [GPU ID]
```

列出实例

```shell
# 列出计算实例
sudo nvidia-smi mig -lci
# 列出 GPU 实例
sudo nvidia-smi mig -lgi
```