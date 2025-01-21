# Setup

## Developing Environment

### Setup Docker
```sh
# Setup
docker run --name heigui-dev --net host -v /wkspc/heigui/:/wkspc/heigui --security-opt seccomp=unconfined --privileged=true -it ubuntu

# Refresh
newgrp docker

# Check Running Process
docker ps -a
```

### Connect to Docker Container
- `Ctrl + Shift + P`
- `Dev-Containers: Attach to Running Container...`

### LLVM BUILD
```sh
# Preparation
apt-get update
apt update

# Install Libraries
apt install python3 python3-pip
apt install lld
pip install cmake
pip install Ninja

# Compile
# in LLVM repo
mkdir build && cd build
cmake -G Ninja -DCMAKE_BUILD_TYPE=Debug -DLLVM_USE_LINKER=lld -DLLVM_ENABLE_PROJECTS="clang;lld" -DLLVM_TARGETS_TO_BUILD="X86" -DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD="DLC" ../llvm
ninja
```

### Setup `dlc-thunk`, `DLCsim`, `DLCSynapse`, and `DLC_Custom_Kernel`:
```sh
# dlc-thunk
cd dlc-thunk
./compile.sh

# DLCsim
apt-get install libelf-dev
cd DLCsim
mkdir build && cd build
cmake -G Ninja ../ -DCMAKE_CXX_FLAGS="-O3"
ninja install

# DLCSynapse
apt-get install -y libhugetlbfs-dev
cd DLCSynapse
mkdir build && cd build
cmake -G Ninja ../ -DCMAKE_CXX_FLAGS="-O3"
ninja install

# DLC_Custom_Kernel
vim ~/.bashrc
# add env
export LD_LIBRARY_PATH=/usr/local/chipltech/thunk/lib/:/usr/local/chipltech/synapse/lib/:/usr/local/chipltech/simulator/lib/:/usr/local/chipltech/kernel/lib/
export SYNAPSE_MEM_MANAGE_BY_DEV=1
export SUITE_TEST_J=1

export LLVM_PATH=/wkspc/heigui/LLVM
export HL_DEBUG_LEVEL=NONE

export WANDB_MODE=disabled

export ACCELERATE_TORCH_DEVICE=dlc
export HF_ENDPOINT=https://hf-mirror.com
export DLC_SIM_USE_ELF2HEX_POLYFILL=1
export DLC_VISIBLE_DEVICES=1

export DLC_SYN_VERBOSE=4
export DLC_SYN_DEBUG=1
export DLC_SYN_USE_SIM=1
# 
source ~/.bashrc
mkdir build && cd build
cmake -G Ninja ../
ninja syntests install
```