# Step Realtime 安装指南

## 安装与运行

我们提供了多种安装和部署 Step Realtime API Server 的方式，包含预编译二进制文件、本地源码编译以及 Docker 部署等，用户可以根据自身需求选择合适的方式进行安装。

部分服务（如 vLLM Server、Token2Audio Server）依赖 GPU 加速，建议提前准备满足需求的硬件环境。

| 服务 | 启动方式 |
|------|----------|
| **vLLM Server** | [通过源码运行](#vllm-source-compilation)、[通过 Docker 运行](#vllm-docker) |
| **Token2Audio Server** | [通过 pip 安装运行](#token2audio-pip)、[通过源码运行](#token2audio-source-compilation)、[通过 Docker 运行](#token2audio-docker) |
| **Webrtcvad Server** | [通过二进制运行](#webrtcvad-binary)、[通过源码运行](#webrtcvad-source-compilation)、[通过 Docker 运行](#webrtcvad-docker) |
| **Whisper Server** | [通过二进制运行](#whisper-binary)、[通过源码运行](#whisper-source-compilation)、[通过 Docker 运行](#whisper-docker) |
| **Realtime API** | [通过二进制运行](#realtime-binary)、[通过源码运行](#realtime-source-compilation)、[通过 Docker 运行](#realtime-docker) |

## 1. vLLM Chat Server 部署

vLLM Chat Server 是对话交互核心组件，依赖 Step-Audio-2 系列模型，支持源码编译和 Docker 两种部署方式。

### 1.1 模型下载（前置步骤）

Step-Audio-2 模型已发布至 Hugging Face 和 ModelScope，下载后用于组件启动。

从 Hugging Face 下载（推荐，需先安装 huggingface-hub）

```bash
pip install huggingface-hub
hf download stepfun-ai/Step-Audio-2-mini --local-dir ./chat-model

# 或者从 ModelScope 下载（国内环境推荐，需先安装 modelscope）
# pip install modelscope
# modelscope download --model stepfun-ai/Step-Audio-2-mini --local_dir ./chat-model
```

下载完成后，./chat-model 目录包含模型权重、配置文件及 Token2Audio 辅助模型。

### <a id="vllm-source-compilation"></a>1.2 源码编译部署

```bash
# 1. 克隆定制化 vLLM 源码（适配 Step-Audio-2 模型）
git clone https://github.com/stepfun-ai/vllm.git
cd vllm

# 2. 安装依赖（editable 模式，支持代码调试）
pip install -e .

# 3. 切换至模型适配分支
git checkout step-audio-2-mini

# 4. 启动服务（参数需根据硬件调整）
python3 -m vllm.entrypoints.openai.api_server \
    --model ./chat-model  # 模型下载目录
    --served-model-name step-audio-2-mini \
    --port 7781 \
    --host 0.0.0.0 \
    --max-model-len 65536  # 模型最大序列长度（影响显存占用）
    --max-num-seqs 128      # 最大并发序列数（影响吞吐量）
    --tensor-parallel-size 4  # 张量并行 GPU 数量（单卡设为 1）
    --gpu-memory-utilization 0.85  # 显存利用率阈值
    --trust-remote-code
```


| 参数 | 作用 | 推荐值 |
|------|------|--------|
| `--max-model-len` | 控制模型支持的最大对话长度 | 长对话：65536，短对话：8192 |
| `--max-num-seqs` | 最大并发请求数 | 显存充足：128，显存紧张：32 |
| `--tensor-parallel-size` | 占用 GPU 数量 | 与实际 GPU 数量一致（单卡设为 1） |


### <a id="vllm-docker"></a>1.3 Docker 部署

```bash
# 1. 拉取预编译镜像
docker pull stepfun2025/vllm:step-audio-2-v20250909

# 2. 启动容器（映射模型目录和端口）
docker run --rm -ti --gpus all \
    -v $(pwd)/chat-model:/step-audio-2-mini  # 本地模型目录映射到容器
    -p 7781:7781 \
    stepfun2025/vllm:step-audio-2-v20250909 \
    -- vllm serve /step-audio-2-mini \
    --served-model-name step-audio-2-mini \
    --port 7781 \
    --max-model-len 65536 \
    --max-num-seqs 128 \
    --tensor-parallel-size 4 \
    --gpu-memory-utilization 0.85 \
    --trust-remote-code
```

## 2. Whisper Server 部署

Whisper Server 提供语音转文字（ASR）功能，支持预编译二进制、源码编译和 Docker 三种部署方式。

### <a id="whisper-binary"></a>2.1 预编译二进制部署

预编译的 Whisper Server 默认有 ggml-base 模型，可通过 API 下载其他模型。

```bash
# 1. 下载对应平台的预编译包（比如 macos）
wget https://github.com/stepfun-ai/step-realtime/releases/download/0.0.1/whisper-server-linux-arm64.tar.gz

# 2. 解压并进入目录
tar -xzf whisper-server-linux-amd64-vX.X.X.tar.gz
cd whisper-server-linux-amd64

# 3. 启动服务
./whisper-server -addr=:7779 -model_dir=models  # model_dir 为模型存储目录
```

### <a id="whisper-source-compilation"></a>2.2 源码编译部署

```bash
# 1. 克隆项目源码
git clone https://github.com/stepfun-ai/step-realtime.git
cd step-realtime

# 2. 编译 Whisper Server（含 whisper.cpp 依赖库）
chmod +x scripts/build.sh
./scripts/build.sh whisper

# 3. 启动服务（编译产物位于 bin/ 目录）
./bin/whisper-server -addr=:7779 -model_dir=models
```

### <a id="whisper-docker"></a>2.3 Docker 部署

```bash
# 启动容器（内置 ggml-base 模型，映射模型存储卷）
docker run -d \
  --name step-whisper-server \
  -p 7779:7779 \
  stepfun2025/whisper-server:latest
```

### 2.4 验证

浏览器打开端口 7779，可以上传 wav 测试 asr 功能。

<img width="500" alt="image" src="https://github.com/user-attachments/assets/6467547c-567b-4251-a301-b39306f719e9" />


## 3. Webrtcvad Server 部署

Webrtcvad Server 提供语音活动检测（VAD）功能，支持三种部署方式。

### <a id="webrtcvad-binary"></a>3.1 预编译二进制部署（推荐）

```bash
# 1. 下载预编译包
wget https://github.com/stepfun-ai/step-realtime/releases/download/0.0.1/webrtcvad-server-linux-amd64.tar.gz

# 2. 解压并启动
tar -xzf webrtcvad-server-linux-amd64-vX.X.X.tar.gz
cd webrtcvad-server-linux-amd64
./webrtcvad-server -addr=:7778
```

### <a id="webrtcvad-source-compilation"></a>3.2 源码编译部署

```bash
# 1. 克隆项目源码
git clone https://github.com/stepfun-ai/step-realtime.git
cd step-realtime

# 2. 编译 Webrtcvad Server
chmod +x scripts/build.sh
./scripts/build.sh webrtcvad

# 3. 启动服务
./bin/webrtcvad-server -addr=:7778
```

### <a id="webrtcvad-docker"></a>3.3 Docker 部署

```bash
# 启动容器
docker run -d \
  --name step-webrtcvad-server \
  -p 7778:7778 \
  stepfun2025/webrtcvad-server:latest
```

### 2.4 验证

浏览器打开端口 7778，可以上传 wav 测试 vad 功能。

<img width="500" alt="image" src="https://github.com/user-attachments/assets/83062792-df21-4a2b-8004-02d24b7efb2c" />


## 4. Token2Audio Server 部署

Token2Audio Server 提供 Token 转语音功能，支持 Pip 安装、源码编译和 Docker 三种部署方式。

### <a id="token2audio-pip"></a>4.1 Pip 安装部署（推荐，快速启动）

```bash
# 1. 安装 Token2Audio 包
# 方式 1：基础安装（CPU 支持）
pip install git+https://github.com/stepfun/step-realtime.git#subdirectory=pytoken2audio

# 方式 2：GPU 加速安装（需提前安装 CUDA）
pip install git+https://github.com/stepfun/step-realtime.git#subdirectory=pytoken2audio[gpu]

# 方式 3：开发模式安装（支持代码修改）
git clone <repo-url>
cd step-realtime/pytoken2audio
pip install -e ".[gpu]"  # 含 GPU 依赖

# 2. 下载模型
python -m token2audio.download /path/to/token2audio-model  # 模型存储路径

# 3. 启动服务
python -m token2audio.server --model-dir /path/to/token2audio-model --port 7780
```

### <a id="token2audio-source-compilation"></a>4.2 源码编译部署

```bash
# 1. 克隆项目源码
git clone <repo-url>
cd step-realtime

# 2. 编译 Token2Audio Server
chmod +x scripts/build.sh
./scripts/build.sh token2audio

# 3. 启动服务
./bin/token2audio-server --model-dir /path/to/token2audio-model --port 7780
```

### <a id="token2audio-docker"></a>4.3 Docker 部署

```bash
# 启动容器（内置默认模型，GPU 加速）
docker run -d \
  --name step-token2audio-server \
  --gpus all \
  -p 7780:7780 \
  stepfun2025/token2audio-server:latest
```

### 4.4 验证

浏览器打开 7780，可以输入 token 测试语音生成功能。

<img width="500" alt="image" src="https://github.com/user-attachments/assets/38c6003c-9b28-4adc-97cb-0bbc230aceb0" />

## 5. Realtime API Server 部署

Realtime API Server 是系统入口组件，聚合所有子服务功能，支持三种部署方式。

### <a id="realtime-binary"></a>5.1 预编译二进制部署（推荐）

```bash
# 1. 下载预编译包

wget https://github.com/stepfun-ai/step-realtime/releases/download/0.0.1/realtime-server-linux-amd64.tar.gz

# 2. 解压并进入目录

tar -xzf realtime-server-linux-amd64.tar.gz
cd server-linux-amd64

# 3. 启动服务（指定配置文件，需提前配置子服务地址）
./server -addr=:7777 -config=conf.yaml
```

### <a id="realtime-source-compilation"></a>5.2 源码编译部署

```bash
# 1. 克隆项目源码
git clone https://github.com/stepfun-ai/step-realtime
cd step-realtime

# 2. 编译所有组件（含 Realtime Server 及子服务）
chmod +x scripts/build.sh
./scripts/build.sh all  # 编译产物位于 bin/ 目录

# 3. 启动服务（需先启动所有子服务）
./bin/server -addr=:7777 -config=conf.yaml
```

### <a id="realtime-docker"></a>5.3 Docker 部署

#### 5.3.1 单独启动容器

```bash
# 启动 Realtime API Server（需提前启动子服务并配置 conf.yaml）
docker run -d \
  --name step-realtime-server \
  -p 7777:7777 \
  -v $(pwd)/conf.yaml:/app/conf.yaml:ro \
  stepfun2025/step-realtime:latest \
  /app/server -addr=:7777 -config=/app/conf.yaml
``` 

## 6. 配置文件说明

Realtime API Server 通过 conf.yaml 配置子服务连接信息、模型参数等，以下为完整示例及关键配置说明。

示例配置如下：

```yaml
# ASR 服务（Whisper Server）配置
asr:
  - type: whisper-server
    model: ggml-base  # 与 Whisper Server 加载的模型一致
    language: null    # 自动检测语言（可指定 zh/en 等）
    url: http://localhost:7779/v1/audio/transcriptions  # Whisper Server 地址

# 对话服务（vLLM Chat Server）配置
chat:
  - model: step-audio-2-mini  # 与 vLLM Server 配置的模型名称一致
    url: http://localhost:7781/v1/chat/completions  # vLLM Server 地址
    instructions: 你是一个 AI 机器人  # 系统提示词
    top_p: 0.7  # 采样 Top P 参数
    temperature: 0.7  # 生成多样性控制（0~1）
    repetition_penalty: 1.07  # 重复惩罚参数
    audio_out:
      type: token2audio
      url: ws://localhost:7780/v1/token2audio  # Token2Audio Server WebSocket 地址
      voices:  # 支持的音色列表
        - name: female
        - name: male

# VAD 服务（Webrtcvad Server）配置
vad:
  type: webrtcvad-server
  url: ws://localhost:7778/ws/v1/vad  # Webrtcvad Server WebSocket 地址
  threshold: 0.8  # 激活阈值（0~1，值越高越严格）
  prefix_padding_ms: 300  # 前缀填充时间（毫秒）
  silence_duration: 300  # 沉默判定时间（毫秒）
  create_response: true  # 是否生成响应
  interrupt_response: true  # 是否允许中断响应

# 追踪日志配置
trace:
  enable: true  # 启用追踪日志
  type: local  # 日志存储类型（本地文件）
  path: ./trace  # 日志存储路径
  compress: true  # 压缩日志文件
  archive: true  # 自动归档日志
```
