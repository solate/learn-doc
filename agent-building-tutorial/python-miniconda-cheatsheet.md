# Python 环境速查表（miniconda）

> 规则就两条：**不在 base 装包**、**一个项目一个 env**。Go 后端不依赖 Python，
> 辅助脚本统一用 `agent` 这一个环境。

## 一次性配置（换机器/重装才要重来）

```bash
conda init zsh                                  # 否则 conda activate 报 "Run 'conda init'..."
conda config --set auto_activate_base false     # 新终端不自动进 base
```

- `auto_activate_base` 弹出改名 `auto_activate` 的警告是正常的，不是错。
- 镜像已配清华源（`~/.condarc`），国内下载不再超时。

## 日常就这 5 条

```bash
conda create -n <名字> python=3.12      # 建环境
conda activate agent                    # 进入环境
pip install <包>                        # 装包
conda deactivate                        # 退出环境
conda env remove -n <名字>              # 彻底删除，磁盘即回收
```

## 速查

| 想做什么 | 命令 |
|---|---|
| 看有哪些环境 | `conda env list` |
| 看环境装了啥 | `conda list -n <名字>` |
| 装重编译包(numpy/torch) | `conda install <包>` |
| 升级 Python 版本 | `conda install -n <名字> python=3.13` |
| 清下载缓存 | `conda clean --all` |

## 本地 VSCode

1. 装 Microsoft 官方 **Python** 扩展
2. `Cmd+Shift+P` → `Python: Select Interpreter` → 选 `agent`

选一次即可：该文件夹终端自动 `conda activate`、F5 调试走同一解释器、`.ipynb` 内核也用它。

## 换机器时：镜像配置（本机已配好，勿改）

```yaml
# ~/.condarc 需要的内容
auto_activate: false
default_channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
custom_channels:
  conda-forge: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
```

改完跑一次 `conda clean -i` 清索引缓存。