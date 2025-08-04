FROM nvidia/cuda:12.9.1-cudnn-runtime-ubuntu22.04

RUN apt update && apt install -y git curl wget libglib2.0-0 libgl1 gcc g++ cmake
RUN curl -fsSL https://ollama.com/install.sh | sh
RUN echo "export TERM=xterm-256color" >> /etc/bash.bashrc
RUN adduser user01
