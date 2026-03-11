# iluvatar-quick-start

天数智芯 GPGPU 常见环境配置、信息收集

更新时间：2026-03-11（基于公开资料检索）

## 资料总览

- 官方主页（产品与支持入口）：https://www.iluvatar.com/
- 飞桨硬件支持（天数 GPGPU）：https://www.paddlepaddle.org.cn/documentation/docs/zh/hardware_support/iluvatar_gpu/index_cn.html
- Paddle-iluvatar（Paddle 自定义设备实现）：https://github.com/PaddlePaddle/Paddle-iluvatar
- Ix Container Toolkit（容器运行时）：https://github.com/Deep-Spark/ix-container-toolkit
- IxRT 开源仓库（推理引擎）：https://github.com/Deep-Spark/iluvatar-corex-ixrt
- 社区服务器实践文档（含 PyTorch 环境变量）：https://github.com/LearningInfiniTensor/.github/blob/main/server/iluvatar/doc.md
- Paddle 已知问题样例（run_check、多卡、退出段错误）：https://github.com/PaddlePaddle/Paddle/issues/74961

## 1. PyTorch 环境（天数智芯）

公开资料里，PyTorch 在天数平台通常依赖厂商 SDK 镜像或预装环境。可先用下面方法快速确认：

```bash
export COREX_HOME=/usr/local/corex
export PATH=$PATH:$COREX_HOME/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$COREX_HOME/lib
export PYTHONPATH=/usr/local/corex/lib64/python3/dist-packages
python3 -c "import torch; print(torch.__version__)"
```

说明：

- `/usr/local/corex/lib64/python3/dist-packages` 是社区文档给出的常见 PyTorch 安装路径。
- 如果 `import torch` 失败，优先排查 `PYTHONPATH`、`LD_LIBRARY_PATH` 和当前 CoreX 版本是否匹配。
- 若你们是生产环境，建议直接使用天数官方提供的 SDK 镜像/安装包组合，避免手动拼版本。

## 2. Paddle 环境（天数智芯）

### 2.1 官方入口

先看飞桨硬件支持页（天数 GPGPU）：  
https://www.paddlepaddle.org.cn/documentation/docs/zh/hardware_support/iluvatar_gpu/index_cn.html

### 2.2 从 Paddle-iluvatar 构建/安装

仓库给出的流程要点：

```bash
# 先联系 Iluvatar 客服获取 SDK 镜像（services@iluvatar.com）
git clone https://github.com/PaddlePaddle/Paddle_Iluvatar.git
bash build_paddle.sh
pip install Paddle/build/python/dist/paddlepaddle*
cd tests
bash run_test.sh
```

其中 `build_paddle.sh`、`run_test.sh` 是仓库给出的标准脚本入口。

## 3. 基础环境安装（驱动/SDK/容器）

天数智芯的软件栈通常是“驱动 + SDK/CoreX + 框架包”配套，版本强绑定，建议优先走厂商发布组合。

### 3.1 最小检查

```bash
ixsmi
```

出现 GPU 列表（如 BI-V150）即表示驱动层基本可用。

### 3.2 容器运行时（推荐）

参考 Ix Container Toolkit：

```bash
git clone https://github.com/Deep-Spark/ix-container-toolkit
cd ix-container-toolkit
make all
sudo make install
sudo ix-ctk runtime configure --runtime docker --ix-set-as-default
sudo systemctl daemon-reload
sudo systemctl restart docker
```

验证：

```bash
sudo docker run -it --rm --runtime iluvatar -e IX_VISIBLE_DEVICES=0 corex:4.0.0 ixsmi
```

如果要给 Kubernetes 用，可按仓库文档切换 containerd/crio 配置方式。

## 4. 常见问题排查清单

### 4.1 看不到卡 / 设备异常

1. 先跑 `ixsmi`，确认驱动和设备状态。  
2. 再跑容器内 `ixsmi`，确认 runtime 已生效。  
3. 如果宿主机正常、容器不正常，优先重查 `ix-ctk runtime configure` 和 docker/containerd 重启步骤。

### 4.2 PyTorch 导入失败

优先检查：

- `PYTHONPATH` 是否包含 `/usr/local/corex/lib64/python3/dist-packages`
- `LD_LIBRARY_PATH` 是否包含 `$COREX_HOME/lib`
- Python 版本、CoreX/SDK 版本是否和镜像/whl 对齐

### 4.3 Paddle run_check 多卡卡住或退出段错误

公开 issue 中有类似现象，日志显示设置下面变量后可继续校验：

```bash
export PADDLE_XCCL_BACKEND=iluvatar_gpu
python3 -c "import paddle; paddle.utils.run_check()"
```

另外 `No ccache found` 只是告警，通常不影响功能正确性，但会影响编译速度。

## 5. 建议的落地流程

1. 固化一套“驱动 + CoreX/SDK + PyTorch/Paddle”的版本矩阵。  
2. 每次升级只变更一个层级（先驱动，再框架）。  
3. 把 `ixsmi`、`import torch`、`paddle.utils.run_check()` 做成自动化健康检查。  
4. 优先使用厂商镜像和官方仓库脚本，减少手工安装差异。
