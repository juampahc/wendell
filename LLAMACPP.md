# Building llamacpp
The goal of this guide is to illustrate the process of building llamacpp for two platforms CPU and GPU.

### Why building from source?

In my case, I create my VMs with Proxmox enabling GPU-passthrough with PCI (when needed). I don't see the gain in using docker and passing the gpu to the container daemon. Plus, I would like to use OpenBLAS for cpu inference.

## Requirements

First we install what we will be using (install everything even if you are building for just one platform).

```bash
sudo apt update
sudo apt install -y git build-essential cmake libopenblas-dev pkg-config ccache libcurl4-openssl-dev
```

Now clone the repository:
```bash
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
```

## CUDA

I assume you already have the drivers and the CUDA toolkit installed.

If you are planning to make inference with CUDA you have two options. You can go straight to compile:
```bash
cmake -B build -DGGML_CUDA=ON
cmake --build build --config Release
```
Or, in case you see some issues, you can also specify the architecture by giving the compute capability (in case the compile cannot se nvcc). This is the best way to build if you have old hardware:
```bash
cmake -B build -DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES="61"
cmake --build build --config Release
```

I have personally used the last one, specifying my architecture. It should be noted that (at the time of writing) the release `b6381` already has some warnings relative to support gpu architures below 75. For a future note.

Now that we have llama.cpp we can launch the server with the following command:

```bash
# You need to cd into llama.cpp/build/
./llama-server -m /mnt/model-hub/Qwen2.5-VL-3B-Instruct-GGUF/Qwen2.5-VL-3B-Instruct-Q6_K.gguf --host 0.0.0.0 --port 8080 -c 4096 -ngl 0
```

## CPU

In order to build llamacpp for based-cpu inference, the command is very similar. We built it with OpenBLAS for cpu:

```bash
cmake -B build -DGGML_BLAS=ON -DGGML_BLAS_VENDOR=OpenBLAS
cmake --build build -j$(nproc) --config Release
```

And again, we launch with the same command:
```bash
# You need to cd into llama.cpp/build/
./llama-server -m /mnt/model-hub/Qwen2.5-VL-3B-Instruct-GGUF/Qwen2.5-VL-3B-Instruct-Q6_K.gguf --host 0.0.0.0 --port 8080 -c 4096 -ngl 0
```

## Making a service

At this moment my homelab is not using kubernetes, so I thought that the best approach at this moment is to use `systemctl` to make services. This way, we can have the server up and runnning no matter the times we reboot our VM. What do we need? The most important thing is to locate both the model-hub or directory where you have GGUF files and the directory where the binaries of llama.cpp can be found. Once this is clear, the process is really straighforward:

- First create the file for defining the service:

```bash
sudo nano /etc/systemd/system/llama-server.service
```

- If you don't have `nano` installed, use your preferred editor. Once the editor shows up fill the file like this:

```bash
[Unit]
Description=Llama Server
After=network.target

[Service]
ExecStart=/path/to/llama-server -m /mnt/model-hub/Qwen2.5-VL-3B-Instruct-GGUF/Qwen2.5-VL-3B-Instruct-Q6_K.gguf --host 0.0.0.0 --port 8080 -c 4096 -ngl 0
User=jp
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target

```

- For the system to recognize the service we need to restart the daemon:

```bash
sudo systemctl daemon-reload
```
- And now the pretty well known wombo-combo of commands for systemctl, i.e. `status,start,restart,enable,disable`. In this case we perform enable, and start:

```bash
sudo systemctl enable llama-server
sudo systemctl start llama-server
```

Remember that for checking logs:
```bash
sudo journalctl -u llama-server -f
```

There are some cons of this approach: the command is static, changing it requires us modifying a file. Plus, we cannot change the model on the fly, we need to stop the service, modify the file, reload the daemon and start again.