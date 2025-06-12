# Run Llama.cpp with Deepseek on Risc-V(Licheepi 4A)
Author: Ruixi Date:25/6/12

## 环境准备
- LicheePi 4A (16G)

- 良好的网络环境

- RevyOS系统镜像

- LLM 模型

	> 笔者测试使用的是 DeepSeek-R1-Distill-Qwen-1.5B-Q2_K 与 DeepSeek-R1-Distill-Qwen-1.5B-Q4_K_M
	>
	> 下载：[https://huggingface.co/unsloth/DeepSeek-R1-Distill-Qwen-1.5B-GGUF](https://huggingface.co/unsloth/DeepSeek-R1-Distill-Qwen-1.5B-GGUF)
### 镜像安装

参考 [https://docs.revyos.dev/docs/Installation/licheepi4a/](https://docs.revyos.dev/docs/Installation/licheepi4a/)

### 依赖安装

```bash
sudo apt update && sudo apt upgrade
sudo apt install git pkg-config libcurl4-openssl-dev cmake make gcc g++
sudo apt install ccache #加速重复编译
```

## 编译 Llama.cpp

Llama.cpp 的 PR [ggml : riscv: add xtheadvector support #13720p](https://github.com/ggml-org/llama.cpp/pull/13720) 为 XTheadVector 扩展指令集提供了支持，可显著提升运行 LLM 模型时的性能。

```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp
mkdir build
cd build
cmake .. -DGGML_RVV=1 -DGGML_XTHEADVECTOR=1 -DGGML_RV_ZFH=0
make -j$(nproc)
```

编译结果于文件夹 ` build/bin `中：

```bash
debian@revyos-lpi4a:~/llama.cpp/build$ tree bin/
bin/
├── libggml-base.so
├── ...
├── llama-cli
├── ...
└── test-tokenizer-1-spm
```

其中 `llama-cli`是我们所需的可执行文件。

## 运行

在终端中交互式运行：

```bash
cd ~
llama.cpp/build/bin/llama-cli -m ./DeepSeek-R1-Distill-Qwen-1.5B-Q2_K.gguf -t 4 
```

 在`-m`后指定实际模型的位置。`-t 4`代表使用4个线程，笔者所用的`LicheePi 4A`只有四个核心。

```bash
debian@revyos-lpi4a:~$ llama.cpp/build/bin/llama-cli -m ./DeepSeek-R1-Distill-Qwen-1.5B-Q2_K.gguf -t 4
build: 5640 (2e89f76b) with cc (Debian 14.2.0-11revyos1) 14.2.0 for riscv64-linux-gnu
main: llama backend init
main: load the model and apply lora adapter, if any
llama_model_loader: loaded meta data with 38 key-value pairs and 339 tensors from ./DeepSeek-R1-Distill-Qwen-1.5B-Q2_K.gguf (version GGUF V3 (latest))

...
== Running in interactive mode. ==

 - Press Ctrl+C to interject at any time.
 - Press Return to return control to the AI.
 - To return control without starting a new line, end your input with '/'.
 - If you want to submit another line, end your input with '\'.
 - Not using system message. To change it, set a different value via -sys PROMPT


> Hello! Who's there?                   
<think>

</think>

Hello! I'm just a placeholder, software. How can I assist you today? Whether you're seeking information, having questions, or just need a smile, I'm here to help! What's your next goal?

> 
llama_perf_sampler_print:    sampling time =      24.33 ms /    57 runs   (    0.43 ms per token,  2342.40 tokens per second)
llama_perf_context_print:        load time =    2712.47 ms
llama_perf_context_print: prompt eval time =    3405.18 ms /     9 tokens (  378.35 ms per token,     2.64 tokens per second)
llama_perf_context_print:        eval time =   24542.83 ms /    48 runs   (  511.31 ms per token,     1.96 tokens per second)
llama_perf_context_print:       total time =   35294.17 ms /    57 tokens
Interrupted by user
```

## 性能

输入词均为: **Hello! Who's there?**

| 模型                                 | 构建选项                                          | ms per token | Token/s |
| :----------------------------------- | :------------------------------------------------ | -----------: | ------: |
| DeepSeek-R1-Distill-Qwen-1.5B-Q2_K   | DGGML_RVV=1 -DGGML_XTHEADVECTOR=1 -DGGML_RV_ZFH=0 |       511.31 |    1.96 |
| DeepSeek-R1-Distill-Qwen-1.5B-Q4_K_M | DGGML_RVV=1 -DGGML_XTHEADVECTOR=1 -DGGML_RV_ZFH=0 |       610.19 |    1.64 |
| DeepSeek-R1-Distill-Qwen-1.5B-Q2_K   | -DGGML_RVV=0 -DGGML_XTHEADVECTOR=0 ( 无向量优化 ) |      1545.64 |    0.65 |
| DeepSeek-R1-Distill-Qwen-1.5B-Q4_K_M | -DGGML_RVV=0 -DGGML_XTHEADVECTOR=0 ( 无向量优化 ) |      1314.45 |    0.76 |

