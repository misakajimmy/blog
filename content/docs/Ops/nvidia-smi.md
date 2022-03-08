---
title: "Nvidia smi"
weight: 21
---

## nvidia-smi mig

{{< tip >}}
此章节源自于 `nvidia-smi mig --help`，在此基础上进行汉化批注以及添加示例。
{{< /tip >}}

mig -- Multi Instance GPU management.

多实例 GPU 管理。

Usage: 

```shell
nvidia-smi mig - [options]
```

Options include:

- [`-h` | `--help`]: 

Display help information.

显示帮助信息

- [`-i` | `--id`]: 

枚举索引、PCI 总线 ID 或 UUID。当拥有多个设备时，使用逗号分隔。

- [`-gi` | `--gpu-instance-id`]: 

GPU 实例 ID。当拥有多个 GPU 实例时，使用逗号分隔。

- [`-ci` | `--compute-instance-id`]: 

计算实例 ID。当拥有多个计算实例时使用逗号分隔。

- [`-lgip` | `--list-gpu-instance-profiles`]: 

列出支持的 GPU 实例配置文件。使用 `-i` 可以将此命令重定向到特定的 GPU。

- [`-lgipp` | `--list-gpu-instance-possible-placements`]: 

以下面的格式列出可能的 GPU 实例展示方式，{Start}:Size。使用 `-i` 可以将此命令重定向到特定的 GPU。

- [`-C` | `--default-compute-instance`]: 

当使用 `-cgi` 创建 GPU 实例时使用默认配置文件。

- [`-cgi` | `--create-gpu-instance`]: 

用给定的配置文件元组创建 GPU 实例。一个配置文件元组包含配置文件名称或者 ID 以及一个可选的放置说明符，放置说明符由一个 `:` 和一个放置开始索引组成。使用逗号分隔多个配置文件元组，选项 -i 可用于限制命令在特定 GPU 上运行。

- [`-dgi` | `--destroy-gpu-instance`]: 

摧毁 GPU 实例。使用 `-i` 或者 `-gi` 可以独立或者组合的重定向到特定的 GPU 或者 GPU 实例。

- [`-lgi` | `--list-gpu-instances`]: 

列出 GPU 实例。使用 `-i` 可以将此命令重定向到特定的 GPU。

- [`-lcip` | `--list-compute-instance-profiles`]: 

列出支持的计算实例配置文件。使用 `-i` 或者 `-gi` 可以独立或者组合的重定向到特定的 GPU 或者 GPU 实例。

- [`-cci` | `--create-compute-instance`]: 

为给定的配置文件名称或 ID 创建计算实例。当拥有多个配置文件时使用逗号分隔。使用 `-i` 或者 `-gi` 可以独立或者组合的重定向到特定的 GPU 或者 GPU 实例。

- [`-dci` | `--destroy-compute-instance`]: 

摧毁计算实例。使用 `-i` 或者 `-gi` 或者 `-ci` 可以独立或者组合的重定向到特定的 GPU 或者 GPU 实例或者计算实例。

- [`-lci` | `--list-compute-instances`]: 

列出计算实例。使用 `-i` 或者 `-gi` 可以独立或者组合的重定向到特定的 GPU 或者 GPU 实例。