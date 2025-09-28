# Nvidia Driver

We will first install the drivers in the VM where PCI Passthrought has already been enabled.

The first step is doing a manual research in NVIDIA website to locate the URL of the latest driver, In my case for the GTX 1050 the actual url is:

```
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/580.82.07/NVIDIA-Linux-x86_64-580.82.07.run
```

And we made it auto exec:

```
sudo chmod +x NVIDIA-Linux-x86_64-580.82.07.run
```

Now, by default ubuntu uses nouveau, but we want to use drivers nvc we just downloaded. We put it in a blacklist so out vm is not going to use it:

```
sudo bash -c "echo blacklist nouveau > /etc/modprobe.d/blacklist-nvidia-nouveau.conf"
sudo bash -c "echo options nouveau modeset=0 >> /etc/modprobe.d/blacklist-nvidia-nouveau.conf"
```

After that we make a refresh:

```
sudo update-initramfs -u
```

And we restart the VM manually! Once the VM has reboot we login again with ssh. it is possible to make a small check with `lspci -v` look for the device and see that no driver should being used.

Before installing the actual drivers we need to install some libraries:

```
sudo apt update && sudo apt install build-essential libglvnd-dev pkg-config
```

Once it's done we proceed to the installation:

```
sudo ./NVIDIA-Linux-x86_64-580.82.07.run
```

Some prompts will appear to accept license and other stuff, if you want to omit them you can use:

```bash
# Prompt information
sudo ./NVIDIA-Linux-x86_64-580.82.07.run --no-questions
# No prompts, nothing
sudo ./NVIDIA-Linux-x86_64-580.82.07.run --silent
```

We can confirm that everything works by running `nvida-smi` and seeing our NVIDIA card with CUDA version. We already know the CUDA version that comes with the driver we just installed, CUDA 13.0

# CUDA toolkit

Before we proceed we need to install the CUDA toolkit, later on when trying to make inference we will need this software to be present. After going to the NVIDIA website and doing the appropiate selections the steps to install cuda toolkit are pretty straightforward:

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-ubuntu2404.pin
sudo mv cuda-ubuntu2404.pin /etc/apt/preferences.d/cuda-repository-pin-600
wget https://developer.download.nvidia.com/compute/cuda/13.0.0/local_installers/cuda-repo-ubuntu2404-13-0-local_13.0.0-580.65.06-1_amd64.deb
sudo dpkg -i cuda-repo-ubuntu2404-13-0-local_13.0.0-580.65.06-1_amd64.deb
sudo cp /var/cuda-repo-ubuntu2404-13-0-local/cuda-*-keyring.gpg /usr/share/keyrings/
sudo apt-get update
sudo apt-get -y install cuda-toolkit-13-0
```

In the website there are some commands to install NVIDIA Driver Instructions. The command is really simple `sudo apt-get install -y cuda-drivers` but at this moment I am not sure if its needed or not. Wether it is the same we did earlier or we can skip. I will try to update this guide in the future. For this moment we skip that mentioned command.

Now add cuda to the path, so the system knows where to find it 

```bash
echo 'export PATH=/usr/local/cuda-13.0/bin${PATH:+:${PATH}}' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda-13.0/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}' >> ~/.bashrc
source ~/.bashrc
```

Again, we can check with `nvcc --version`, if we are getting the right ouput, then everything should work fine.

Please not that installation of cuda toolkit is crutial for other components to work right, for example, PyTorch: if later on you want to install python, with say conda or uv, and you want to use the gpu you will need to build or install a compatible version. Let's say PyTorch accepts only 12.1 CUDA version:

```
YOUR CUDA VERSION >= PyTorch Version : Everything is fine, there is backward compatibility
YOUR CUDA VERSION < PyTorch Version : You need to downgrade Pytorch, otherwise there will be errors with set of instructions.
```

Once cuda is installed we will select the inference provider (for example, llama.cpp) but thats part of the future. As a side note, compatibility for Pascal architectures is removed in CUDA 13/