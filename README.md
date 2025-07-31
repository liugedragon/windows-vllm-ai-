# windows-vllm-ai-
windows系统通过vllm部署ai大模型
## **终极部署流程：在 Windows 11 + WSL 上部署多卡 vLLM 服务**

**目标硬件**: 8 x NVIDIA GeForce RTX 4090 D
**目标模型**: Qwen/QwQ-32B (32B 规模)
**目标系统**: Windows 11 + WSL 2 (Ubuntu)
**最终优化配置**: 4 卡并行

------



### **第一阶段：WSL 系统级准备（打好地基）**

#### **第 1 步：在 D 盘安装并启动 WSL**

1. 以管理员身份打开 Windows PowerShell。

2. 安装 WSL 并指定 Ubuntu 发行版：

   Generated powershell

   ```
   wsl --install -d Ubuntu
   ```

   Use code [with caution](https://support.google.com/legal/answer/13505487).Powershell

3. **【关键】将安装好的 Ubuntu 系统迁移到 D 盘**，以避免占用 C 盘空间。

   Generated powershell

   ```
   # 停止 WSL 服务
   wsl --shutdown
   
   # 导出当前的 Ubuntu 系统到 D 盘的一个 tar 文件
   # (请确保 D:\wsl_export 文件夹已存在)
   wsl --export Ubuntu D:\wsl_export\ubuntu.tar
   
   # 注销当前的 Ubuntu (这会删除 C 盘的版本)
   wsl --unregister Ubuntu
   
   # 从 D 盘的 tar 文件重新导入 WSL，并指定安装到 D 盘的新位置
   # (请确保 D:\wsl_system 文件夹已存在)
   wsl --import Ubuntu D:\wsl_system D:\wsl_export\ubuntu.tar --version 2
   ```

4. 设置默认登录用户为 root（或您创建的用户名）：

   Generated powershell

   ```
   ubuntu config --default-user root
   ```

5. 启动您的 WSL 终端。

   ~~~
   wsl
   ~~~

   

#### **第 2 步：【关键系统优化】解决底层环境依赖**

这是我们调试过程中遇到的核心问题，提前解决可以避免所有后续的错误。

1. **更新软件源列表**:

   Generated bash

   ```
   apt update
   ```

2. **安装核心编译工具**: 为了解决 找不到 C 编译器 和 找不到 Python.h 的问题。

   Generated bash

   ```
   apt install -y build-essential python3.12-dev
   ```

3. **【最关键】扩大 WSL 共享内存**: 为了解决多卡通信时的 NCCL error: Cannot allocate memory /dev/shm 问题。

   - 在 **Windows** 中，打开文件资源管理器，在地址栏输入 %USERPROFILE% 并回车。

   - 创建一个名为 .wslconfig 的文件。

   - 用记事本打开它，并粘贴以下内容：

     Generated ini

     ```
     [wsl2]
     shmSize=16G
     ```

   - 回到 **Windows PowerShell**，**必须执行**以下命令来让配置生效：

     Generated powershell

     ```
     wsl --shutdown
     ```

------



### **第二阶段：项目与 Python 环境配置**

1. **重新启动 WSL 终端**。

2. **验证共享内存配置** (可选但推荐):

   Generated bash

   ```
   df -h | grep shm
   ```

   Use code [with caution](https://support.google.com/legal/answer/13505487).Bash

   您应该能看到共享内存大小至少为 16G。

3. **创建项目和模型目录**:

   ```
   # 在 D 盘创建项目、模型文件夹
   mkdir -p /mnt/d/aiworkspace/vllm_project
   mkdir -p /mnt/d/aiworkspace/models
   ```

4. **创建并激活 Python 虚拟环境**:

   ```
   # 进入项目目录
   cd /mnt/d/aiworkspace/vllm_project
   
   # 创建虚拟环境
   python3 -m venv venv
   
   # 激活虚拟环境
   source venv/bin/activate
   ```

   Use code [with caution](https://support.google.com/legal/answer/13505487).Bash

   您的命令行提示符前应该会出现 (venv) 字样。

------



### **第三阶段：安装 vLLM 并配置国内镜像**

1. **配置 Pip 使用清华大学镜像** :

   ```
   pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
   ```

2. **安装 vLLM 和 Flash-Attention**:

   Generated bash

   ```
   pip install vllm
   pip install flash-attn --no-build-isolation
   ```

3. **配置 Hugging Face 使用镜像和 D 盘缓存** (永久生效):

   Generated bash

   ```
   echo 'export HF_HOME="/mnt/d/aiworkspace/models"' >> ~/.bashrc
   echo 'export HF_ENDPOINT="https://hf-mirror.com"' >> ~/.bashrc
   source ~/.bashrc
   ```

------



### **第四阶段：模型下载与部署**

1. **下载模型** (如果您还没有下载):

   - 如果您是从零开始，vLLM 会在启动时自动下载。由于您已经下载好了，我们可以跳过这一步。您的模型应该位于我们之前确定的正确路径：/mnt/d/aiworkspace/models/models/Qwen/QwQ-32B。

2. **执行最终的、性能最优的 4 卡部署命令**:

   - 我们已经通过实验最终确认，在您的 WSL 环境下，4 卡并行是**最稳定、性能最好的配置**。

     #低于4卡显存不足 高于4卡由于通信的限制也会报错 而且成功了速度也不一定快

   ```
   vllm serve /mnt/d/aiworkspace/models/models/Qwen/QwQ-32B \
   --tensor-parallel-size 4 \
   --gpu-memory-utilization 0.8 \
   --max-model-len 8192 \
   --dtype bfloat16 \
   --host 0.0.0.0 \
   --port 6006
   ```

3. **耐心等待**: 等待模型加载和编译，直到您在终端看到 Application startup complete. 的成功信息。

------



### **第五阶段：调用并测试您的模型服务**

1. **保持 vLLM 服务终端运行**。

2. **获取 WSL 的 IP 地址**:

   - 打开一个**新的 Windows PowerShell** 窗口，运行 wsl hostname -I，记下 IP 地址（例如 192.30.76.138）。

3. **使用 Python 脚本进行性能测试**:

   - 在您的 Windows 电脑上，创建一个 test_speed.py 文件。
   - 将下面**已为您配置好**的代码粘贴进去：

   Generated python

   ```
   import sys
   import time
   from openai import OpenAI
   
   # --- 已为您配置好 ---
   WSL_IP_ADDRESS = "172.30.76.138" # 请替换为您自己的真实 IP
   MODEL_PATH = "/mnt/d/aiworkspace/models/models/Qwen/QwQ-32B"
   user_question = "请给我详细介绍一下人工智能的历史，从图リングテスト讲起，直到今天的 Transformer 模型。"
   # --------------------
   
   def run_performance_test():
       api_base_url = f"http://{WSL_IP_ADDRESS}:6006/v1"
       client = OpenAI(base_url=api_base_url, api_key="NOT_USED")
   
       print("="*70 + "\n VLLM 性能测试脚本\n" + "="*70)
       print(f"连接目标: {api_base_url}")
       print(f"测试模型: {MODEL_PATH}\n")
       print("--- 开始请求，请稍候 ---")
   
       try:
           start_time = time.monotonic()
           response = client.chat.completions.create(
               model=MODEL_PATH,
               messages=[{"role": "user", "content": user_question}],
               max_tokens=1024,
               temperature=0.7
           )
           end_time = time.monotonic()
           
           model_reply = response.choices[0].message.content
           completion_tokens = response.usage.completion_tokens
           elapsed_time = end_time - start_time
           tokens_per_second = completion_tokens / elapsed_time if elapsed_time > 0 else 0
   
           print("\n--- 模型回复 ---\n" + model_reply)
           print("\n" + "="*70 + "\n 性能分析报告\n" + "="*70)
           print(f"生成 Token 数量   : {completion_tokens} tokens")
           print(f"总计响应时间      : {elapsed_time:.2f} 秒")
           print(f"平均生成速度      : {tokens_per_second:.2f} tokens/秒")
           print("=" * 70)
   
       except Exception as e:
           print(f"\n[错误] 调用 API 时出错: {e}", file=sys.stderr)
   
   if __name__ == "__main__":
       run_performance_test()
   ```

4. **运行测试**:

   - 在 PowerShell 中，确保已安装 openai 库 (pip install openai)。
   - 运行脚本：python test_speed.py。
   - **连续运行 2-3 次**，您将看到稳定的 **~35 tokens/秒** 的性能表现。
