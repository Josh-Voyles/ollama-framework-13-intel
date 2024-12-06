# Running Ollama on Framework 13 Intel Ultra Series 1

WARNING: Ubuntu Only.
I tested this ad nauseam on Fedora trying to get it to work with no luck.

# Installing (To be installed in 'home' folder)

WARNING: There is no need to install Ollama on your system first.
In fact, having Ollama already running on your machine will stop the new server using the Intel driver from working.

However, you can stop the Ollama if it's currently running on your system.
```bash
systemctl stop ollama
```

## Install IPEX-LLM

### Enable gpu driver support (may not need)
```bash
export FORCE_PROBE_VALUE=$(sudo dmesg  | grep i915 | grep -o 'i915\.force_probe=[a-zA-Z0-9]\{4\}')

sudo sed -i "/^GRUB_CMDLINE_LINUX_DEFAULT=/ s/\"\(.*\)\"/\"\1 $FORCE_PROBE_VALUE\"/" /etc/default/grub
```

### Install compute packages
#### Ubuntu 24.10
```bash
apt-get update
apt-get install -y software-properties-common

# Add the intel-graphics PPA for 24.10
add-apt-repository -y ppa:kobuk-team/intel-graphics

# Install the compute-related packages
apt-get install -y libze-intel-gpu1 libze1 intel-ocloc intel-opencl-icd clinfo

# Install the media-related packages
apt-get install -y intel-media-va-driver-non-free libmfx1 libmfx-gen1.2 libvpl2 libvpl-tools libva-glx2 va-driver-all vainfo
```

#### Ubuntu 24.04 LTS
```bash
# Install the Intel graphics GPG public key
wget -qO - https://repositories.intel.com/gpu/intel-graphics.key | \
  sudo gpg --yes --dearmor --output /usr/share/keyrings/intel-graphics.gpg

# Configure the repositories.intel.com package repository
echo "deb [arch=amd64,i386 signed-by=/usr/share/keyrings/intel-graphics.gpg] https://repositories.intel.com/gpu/ubuntu noble client" | \
  sudo tee /etc/apt/sources.list.d/intel-gpu-noble.list

# Update the package repository meta-data
sudo apt update

# Install the compute-related packages
apt-get install -y libze1 intel-level-zero-gpu intel-opencl-icd clinfo
```

### Verify drivers are installed and functional
Should see arc gpu listed.
```bash
clinfo | grep "Device Name"
```

If you need to add permissions (you can also run clinfo as root)
```bash
sudo gpasswd -a ${USER} render
newgrp render
```

## Install oneAPI

```bash
wget -O- https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB | gpg --dearmor | sudo tee /usr/share/keyrings/oneapi-archive-keyring.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/oneapi-archive-keyring.gpg] https://apt.repos.intel.com/oneapi all main" | sudo tee /etc/apt/sources.list.d/oneAPI.list

sudo apt update

sudo apt install intel-oneapi-common-vars=2024.0.0-49406 \
  intel-oneapi-common-oneapi-vars=2024.0.0-49406 \
  intel-oneapi-diagnostics-utility=2024.0.0-49093 \
  intel-oneapi-compiler-dpcpp-cpp=2024.0.2-49895 \
  intel-oneapi-dpcpp-ct=2024.0.0-49381 \
  intel-oneapi-mkl=2024.0.0-49656 \
  intel-oneapi-mkl-devel=2024.0.0-49656 \
  intel-oneapi-mpi=2021.11.0-49493 \
  intel-oneapi-mpi-devel=2021.11.0-49493 \
  intel-oneapi-dal=2024.0.1-25 \
  intel-oneapi-dal-devel=2024.0.1-25 \
  intel-oneapi-ippcp=2021.9.1-5 \
  intel-oneapi-ippcp-devel=2021.9.1-5 \
  intel-oneapi-ipp=2021.10.1-13 \
  intel-oneapi-ipp-devel=2021.10.1-13 \
  intel-oneapi-tlt=2024.0.0-352 \
  intel-oneapi-ccl=2021.11.2-5 \
  intel-oneapi-ccl-devel=2021.11.2-5 \
  intel-oneapi-dnnl-devel=2024.0.0-49521 \
  intel-oneapi-dnnl=2024.0.0-49521 \
  intel-oneapi-tcm-1.0=1.0.0-435
```
### Reboot machine
```bash
sudo reboot
```

### Install Conda
Accepts agreements and choose 'no' to auto activate
```bash
wget https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh
bash Miniforge3-Linux-x86_64.sh
source ~/.bashrc
```

### Activate Conda Manually
```bash
source miniforge3/bin/activate
```

## Install IPEX-LLM
```bash
conda create -n llm python
conda activate llm
pip install --pre --upgrade ipex-llm[xpu] --extra-index-url https://pytorch-extension.intel.com/release-whl/stable/xpu/us/
```


## Install IPEX-LLM for llama-cpp

```bash
conda create -n llm-cpp python
conda activate llm-cpp
pip install --pre --upgrade ipex-llm[cpp]


mkdir llama-cpp
cd llama-cpp

init-llama-cpp
init-ollama
```


## Set environment variables
```bash
export USE_XETLA=OFF
export OLLAMA_NUM_GPU=999
export no_proxy=localhost,127.0.0.1
export ZES_ENABLE_SYSMAN=1
source /opt/intel/oneapi/setvars.sh
export SYCL_CACHE_PERSISTENT=1
# [optional] if you want to run on single GPU, use below command to limit GPU may improve performance
export ONEAPI_DEVICE_SELECTOR=level_zero:0

./ollama serve
```

# Running
In a separate terminal window, run the model from the same llama-cpp folder.
```bash
./ollama run llama3.2 # running 3.2 model as an example.
```

```bash
# need to check performance without setting. With set, 0 is better than 1.
export SYCL_PI_LEVEL_ZERO_USE_IMMEDIATE_COMMANDLISTS=0
```
