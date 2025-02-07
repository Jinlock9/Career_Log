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