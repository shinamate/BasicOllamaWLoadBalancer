# Basic Ollama with Network Load Balancer
The repository is a basic introduction to setup a local Ollama server with network load balancer. All components are containerized.

The user id is assumed to be 1000 in group 1000. You can get user id and group id by following commands.
```
$ id -u
$ id -g
```

## Install Prerequisites (Need SUDO Permission)
> Tested on Debian-Bookworm

### Install Docker
Follow [https://docs.docker.com/engine/install/]
```
$ sudo usermod -aG docker $(id -un)
```

### Install CUDA and Nvidia Driver
Follow
[https://developer.nvidia.com/cuda-12-9-0-download-archive?target_os=Linux&target_arch=x86_64&Distribution=Debian&target_version=12&target_type=deb_local]

```
$ wget https://developer.download.nvidia.com/compute/cuda/12.9.0/local_installers/cuda-repo-debian12-12-9-local_12.9.0-575.51.03-1_amd64.deb
$ sudo dpkg -i cuda-repo-debian12-12-9-local_12.9.0-575.51.03-1_amd64.deb
$ sudo cp /var/cuda-repo-debian12-12-9-local/cuda-*-keyring.gpg /usr/share/keyrings/
$ sudo apt-get update
$ sudo apt-get -y install cuda-toolkit-12-9
```

Install/Update Host Device Nvidia Driver
```
sudo apt-get install -y cuda-drivers
```

### Install Nvidia Docker Toolkit
 Follow [https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html]
```
$ curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

$ curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
$ apt update 

$ export NVIDIA_CONTAINER_TOOLKIT_VERSION=1.17.8-1

$ sudo apt-get install -y \
      nvidia-container-toolkit=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      nvidia-container-toolkit-base=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      libnvidia-container-tools=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
      libnvidia-container1=${NVIDIA_CONTAINER_TOOLKIT_VERSION}
```

```
$ sudo nvidia-ctk runtime configure --runtime=docker
$ sudo systemctl restart docker docker.socket
```

### Install Other Components
```
$ apt install openssl git tmux
```

## Activate Server
Checking the directory:
```
$ tree ./setup_dockers
./setup_dockers
├── ActivateOllamaServer
├── Dockerfile
├── haproxy
│   └── haproxy.cfg
├── ollama_docker.large
├── ollama_docker.medium
└── ollama_docker.small
```

You can comment out the block of codes that run models if needed. An example of one block of codes for running qwen3:30b-a3b-fp16 in [./setup_dockers/ollama_docker.large].
```
echo 'docker exec -it -u 1000:1000 ollama_server_large_01 bash -ic "ollama run qwen3:30b-a3b-fp16 --keepalive=1m"' > ./run.sh
chmod u+x ./run.sh
tmux new-window -t ollama_server_large:3
tmux send-keys -t ollama_server_large:3 ./run.sh Enter
sleep 1m
```

You can activate the Ollama server using one script in [./setup_dockers].
```
$ cd setup_dockers
$ ./ActivateOllamaServer
```

The Ollama should be available at ports:
- Embedding's API port: 41 (https, load balanced)
- Large model (#param>11b) API port: 51 (https)
- Medium model (11b>#param>3b) API port: 81 (https)
- Small model (#param<3b) API port: 91 (https)

## Configurations
Each model's API are served by separated Docker containers, which can be viewed via Tmux session.

### Frontend and Load Balancer
The configuration of ports for each API is in HAProxy.
```
├── haproxy
│   └── haproxy.cfg
```
If there are users in WAN, correct the IPADDRESS that suitable for Physical Network device before initiate the server.

### Backend
The configurations of Tmux sessions and Docker containers are in these bash scripts.
```
├── ollama_docker.large
├── ollama_docker.medium
└── ollama_docker.small
```

If memory allocation issues occurs on a Ollama server, you can kill the Tmux session and stop the Docker container without interfering other API(s).
```
$ tmux kill-session -t [session_name]
$ docker stop [docker_name]
```

#### If the file is not executable, add user's permission:
```
$ chmod u+x ./ActivateOllamaServer
$ ./ActivateOllamaServer
```

#### Model Storage (Require SUDO Permission)
You can mount Ext4 partitions for storing the Model weights. 
```
$ sudo mount /dev/nvme1n1p1 /home/$(id -un)/Documents/ollama_server_mounted
$ sudo chown -R 1000:1000 /home/$(id -un)/Documents/ollama_server_mounted
```
#### API Verification
Suppose the default port configuration is used.
- Embedding's API port: 41 (https, load balanced)
- Large model (#param>11b) API port: 51 (https)
- Medium model (11b>#param>3b) API port: 81 (https)
- Small model (#param<3b) API port: 91 (https)

You can get all models that stored in the container's OLLAMA environment.
```
$ curl -k https://[ollama host ip address]:51/api/tags
$ curl -k https://[ollama host ip address]:81/api/tags
$ curl -k https://[ollama host ip address]:91/api/tags
```