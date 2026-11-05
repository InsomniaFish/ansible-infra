# ansible-infra

[![Ansible](https://img.shields.io/badge/编排-Ansible-EE0000?style=flat&logo=ansible&logoColor=white)](https://www.ansible.com/)
[![Ubuntu](https://img.shields.io/badge/系统-Ubuntu-E95420?style=flat&logo=ubuntu&logoColor=white)](https://ubuntu.com/)
[![NVIDIA](https://img.shields.io/badge/GPU-NVIDIA-76B900?style=flat&logo=nvidia&logoColor=white)](https://www.nvidia.com/)
[![Docker](https://img.shields.io/badge/容器-Docker-2496ED?style=flat&logo=docker&logoColor=white)](https://www.docker.com/)
[![Kubernetes](https://img.shields.io/badge/集群-Kubernetes_1.34-326CE5?style=flat&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Prometheus](https://img.shields.io/badge/监控-Prometheus-E6522C?style=flat&logo=prometheus&logoColor=white)](https://prometheus.io/)

用 Ansible 管理同一批 Ubuntu 机器上的 GPU 环境和 Kubernetes 集群。主机清单是 `inventory/inventory.ini`。

## gpuenv

安装、检测和卸载 NVIDIA GPU 运行环境，包括驱动、CUDA、OFED、NCCL 等。按组件开关选择装什么。

细节见 [gpuenv/README.md](gpuenv/README.md)。不用 Ansible 的手工步骤见 [gpuenv/manual_install.md](gpuenv/manual_install.md)。

## kubernetes

用 kubeadm 搭建集群：节点初始化、控制面、Worker 加入，以及 NFS、Harbor、Prometheus 等可选组件。

细节见 [kubernetes/README.md](kubernetes/README.md)。
