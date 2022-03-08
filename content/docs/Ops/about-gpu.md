---
title: "About GPU"
weight: 12
---

## 关于显卡驱动

### 查看显卡型号和驱动版本

#### 查看显卡型号

```shell
lspci | grep -i nvidia
```

#### 查看驱动版本

```shell
sudo dpkg --list | grep nvidia-*
```

#### 查看正在使用的显卡驱动所使用的内核版本

```shell
cat /proc/driver/nvidia/version
```