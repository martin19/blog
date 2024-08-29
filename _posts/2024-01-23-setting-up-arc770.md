## setup hardware, document specs.

test

## install ubuntu
version 22.04

## update to latest package versions
skip this.

## install some tools

```sh
sudo apt-get install intel-gpu-tools 
sudo apt-get clinfo
```

## install latest Arc770 drivers

We'll install the Arc770 drivers following this [driver installation guide](https://dgpu-docs.intel.com/driver/client/overview.html#client-install-options)!.
As of today, the Arc770 drivers are under heavy development and installation is not straightforward. 

Add the package repository to apt sources:

```sh
wget -qO - https://repositories.intel.com/gpu/intel-graphics.key | \
  sudo gpg --dearmor --output /usr/share/keyrings/intel-graphics.gpg
echo "deb [arch=amd64,i386 signed-by=/usr/share/keyrings/intel-graphics.gpg] https://repositories.intel.com/gpu/ubuntu jammy client" | \
  sudo tee /etc/apt/sources.list.d/intel-gpu-jammy.list
sudo apt update
```

Install Compute, Media, and Display runtimes

```sh
sudo apt install -y \
  intel-opencl-icd intel-level-zero-gpu level-zero \
  intel-media-va-driver-non-free libmfx1 libmfxgen1 libvpl2 \
  libegl-mesa0 libegl1-mesa libegl1-mesa-dev libgbm1 libgl1-mesa-dev libgl1-mesa-dri \
  libglapi-mesa libgles2-mesa-dev libglx-mesa0 libigdgmm12 libxatracker2 mesa-va-drivers \
  mesa-vdpau-drivers mesa-vulkan-drivers va-driver-all vainfo hwinfo clinfo
```

# make sure the A770 is available as pci device

```sh
martin@mlbox:~/projects/arc770setup$ lspci -v | grep -A 10 VGA
```

```
00:02.0 VGA compatible controller: Intel Corporation Device 4692 (rev 0c) (prog-if 00 [VGA controller])
        DeviceName: Onboard - Video
        Subsystem: Gigabyte Technology Co., Ltd Device d000
        Flags: bus master, fast devsel, latency 0, IRQ 137
        Memory at 6401000000 (64-bit, non-prefetchable) [size=16M]
        Memory at 4000000000 (64-bit, prefetchable) [size=256M]
        I/O ports at 5000 [size=64]
        Expansion ROM at 000c0000 [virtual] [disabled] [size=128K]
        Capabilities: <access denied>
        Kernel driver in use: i915
        Kernel modules: i915
--
03:00.0 VGA compatible controller: Intel Corporation Device 56a0 (rev 08) (prog-if 00 [VGA controller])
        Subsystem: Intel Corporation Device 1020
        Flags: bus master, fast devsel, latency 0, IRQ 138
        Memory at 41000000 (64-bit, non-prefetchable) [size=16M]
        Memory at 6000000000 (64-bit, prefetchable) [size=16G]
        Expansion ROM at 42000000 [disabled] [size=2M]
        Capabilities: <access denied>
        Kernel driver in use: i915
        Kernel modules: i915
```

Second must be our Arc770, memory reports to 16M. The kernel driver in use is `i915` that we have just installed.

```sh
martin@mlbox:~/projects/arc770setup$ clinfo -l
 ```

 ```
 Platform #0: Intel(R) OpenCL Graphics
 `-- Device #0: Intel(R) Arc(TM) A770 Graphics
Platform #1: Intel(R) OpenCL Graphics
 `-- Device #0: Intel(R) UHD Graphics 730

 ```

When we run gputop we see this:

```sh
martin@mlbox:~/projects/arc770setup$ sudo intel_gpu_top
```

```
intel-gpu-top: 8086:56a0 @ /dev/dri/card1 -    0/   0 MHz; 100% RC6;        0 irqs/s

         ENGINES     BUSY                                                                               MI_SEMA MI_WAIT
       Render/3D    0.00% |                                                                           |      0%      0%
         Blitter    0.00% |                                                                           |      0%      0%
           Video    0.00% |                                                                           |      0%      0%
    VideoEnhance    0.00% |                                                                           |      0%      0%
       [unknown]    0.00% |                                                                           |      0%      0%
```

Obviously our gpu is at `/dev/dri/card1` which is what we want. Following command from the OpenCL framework
tells us the device is available as opencl device.

### setup permissions to access gpu

We'll need to setup permissions for our current user to use the gpu.

```sh
stat -c "%G" /dev/dri/render*
groups ${USER}
sudo gpasswd -a ${USER} render
stat -c "%G" /dev/dri/render*
```
# Intels Arc770 software stack

Intel is developing a driver layer and api called OneApi. It can be used with pytorch through an extension mechanism.
We'll install oneApi and make sure it runs properly with pytorch.

## Install oneApi Toolkits

https://www.intel.com/content/www/us/en/developer/tools/oneapi/base-toolkit-download.html?operatingsystem=linux&distributions=aptpackagemanager

```sh
wget -O- https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB \ | gpg --dearmor | sudo tee /usr/share/keyrings/oneapi-archive-keyring.gpg > /dev/null
```

```sh
echo "deb [signed-by=/usr/share/keyrings/oneapi-archive-keyring.gpg] https://apt.repos.intel.com/oneapi all main" | sudo tee /etc/apt/sources.list.d/oneAPI.list
```

```sh
sudo apt install intel-basekit
```

This installs about 14GB of packages. 

# Install python, pytorch and ipex 

We all know it: you never have the correct python version installed for a certain project. There are some solutions to it like virtual environmens (venv).
I'm not a fan of this as it adds even more complexity, so I'll try python version `3.10` which is supposed to work with the intel extensions for pytorch.
Let's try that:

```
martin@mlbox:/opt$ python3 --version
Python 3.10.12
```

We'll need to install pytorch and ipex, the [ipex github repo](https://github.com/intel/intel-extension-for-pytorch) says 

>"Note: Intel® Extension for PyTorch* v2.1.10+xpu requires PyTorch*/libtorch v2.1.* (patches needed) to be installed."

so we'll make sure we have the appropriate version of pytorch:

```
sudo apt install -y intel-oneapi-dpcpp-cpp-2024.0 intel-oneapi-mkl-devel=2024.0.0-49656
```

```
python -m pip install torch==2.1.0a0 torchvision==0.16.0a0 torchaudio==2.1.0a0 intel-extension-for-pytorch==2.1.10+xpu --extra-index-url https://pytorch-extension.intel.com/release-whl/stable/xpu/us/
```

Now lets try to run the sanity test given on the page:

```sh
source {DPCPPROOT}/env/vars.sh
source {MKLROOT}/env/vars.sh
python -c "import torch; import intel_extension_for_pytorch as ipex; print(torch.__version__); print(ipex.__version__); [print(f'[{i}]: {torch.xpu.get_device_properties(i)}') for i in range(torch.xpu.device_count())];"
```

We get following error:
```
Traceback (most recent call last):
  File "<string>", line 1, in <module>
  File "/home/martin/.local/lib/python3.10/site-packages/intel_extension_for_pytorch/__init__.py", line 93, in <module>
    from .utils._proxy_module import *
  File "/home/martin/.local/lib/python3.10/site-packages/intel_extension_for_pytorch/utils/_proxy_module.py", line 2, in <module>
    import intel_extension_for_pytorch._C
ImportError: "libmkl_intel_lp64.so.2": cannot open shared object file: No such file or directory
```

Obviously somewhere in the stack a library `libmkl_intel_lp64.so.2` cannot be found, lets see if it is installed:

```
martin@mlbox:~/projects/arc770setup$ find / -name "libmkl_intel_lp64.so.2" 2>/dev/null
/opt/intel/oneapi/mkl/2024.0/lib/libmkl_intel_lp64.so.2
```

It is not clear why the file is not found by the python module, but it can be resolved by setting the 
`LD_LIBRARY_PATH` environment variable. This is the path where the system searches for native shared object libraries.

Lets do this and run command above again:

```
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/intel/oneapi/mkl/2024.0/lib/
```

We get another similar error, this time a library `libsvml.so` is not found. We append this location to the `LD_LIBRARY_PATH` as well and run sanity test again:

```
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/intel/oneapi/compiler/2024.0/lib/
```

Next error coming up...

```
ImportError: /home/martin/.local/lib/python3.10/site-packages/intel_extension_for_pytorch/lib/libintel-ext-pt-gpu.so: undefined symbol: _ZNK5torch8autograd4Node4nameB5cxx11Ev
```

A symbol is not found in the shared library. This tells us some (binary) library does not find a symbol which should be defined in another (binary) library. 
There is an open gitbug issue with an unconfirmed resolution, great: `https://github.com/intel/intel-extension-for-pytorch/issues/457`. Looks as if we need to 
solve that ourselves:

We've installed `intel-extension-for-pytorch==2.1.10+xpu`, it probably imports a symbol from another shared library. We could either make sure the dependency 
exports the symbol or recompile ipex. We'll recompile ipex. It can be tough to succeed quickly but let's try that.

I'll not list every step here, but this is what we need to do:

- checkout ipex source code at tag `2.1.10+xpu`
- checkout pytorch repo at matching version 
- apply intel patches to pytorch
- build pytorch 

```
python setup install
```

- compile ipex as described [here](https://intel.github.io/intel-extension-for-pytorch/#installation)

Check your gcc version to be 12 with following command:

```
ls -larth `which gcc` 
```

If version of the links is not 12, you can adjust the links like this:

```
sudo ln -sf /usr/bin/x86_64-linux-gnu-gcc-nm-12 /usr/bin/gcc-nm
sudo ln -sf /usr/bin/x86_64-linux-gnu-gcc-ar-12 /usr/bin/gcc-ar
sudo ln -sf /usr/bin/x86_64-linux-gnu-gcc-12 /usr/bin/gcc
sudo ln -sf /usr/bin/x86_64-linux-gnu-gcc-ranlib-12 /usr/bin/gcc-ranlib
```

```
install cudatoolkit in conda environment
(base) martin@mlbox:~/projects/arc770setup/ipex$ sudo chown --recursive martin:martin ~/miniconda3/
(base) martin@mlbox:~/projects/arc770setup/ipex$ conda install anaconda::cudatoolkit


wget https://github.com/intel/intel-extension-for-pytorch/raw/v2.1.10%2Bxpu/scripts/compile_bundle.sh 

chmod +x compile_bundle.sh  

sudo ./compile_bundle.sh /opt/intel/oneapi/compiler/2024.0 /opt/intel/oneapi/mkl/2024.0 USE_AOT_DEVLIST='ats-m150'
 ```




# compile ipex without conda
# source the oneapi environment
# call compile script
# setup ccache
# wait


# Setup oneApi on ubuntu 22.04 6.2.0-37-generic
wget -O- https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB | gpg --dearmor | sudo tee /usr/share/keyrings/oneapi-archive-keyring.gpg > /dev/null

# Links

[1] https://dgpu-docs.intel.com/driver/installation.html  
[2] https://www.intel.com/content/www/us/en/docs/oneapi/  installation-guide-linux/2024-0/overview.html  
[3] https://github.com/intel/intel-extension-for-pytorch  
[4] https://intel.github.io/intel-extension-for-pytorch/#installation  