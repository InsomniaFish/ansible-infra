# GPU 环境手动安装（manual_install.md）

按 `gpuenv` role 的实现手工安装。命令在目标机上以 root 执行。版本以 `vars/gpuenv_repos.yml` 为准：驱动 580.105.08、CUDA 13.0.2、NCCL 2.28.9、OpenMPI 4.1.7、NVSHMEM 3.7.2。

## 前置条件

- 本文件对应 **Ubuntu 22.04**（`22.04` ≤ 版本 `< 22.10`）。`ubuntu24.04` 分支用 `ubuntu2404` 软件源，GPUDirect RDMA 走 DMA-BUF，不要加载 `nvidia_peermem`。
- root。`DEBIAN_FRONTEND=noninteractive`。
- 目录：安装包 `/opt/packages`，编译 `/opt/build`。离线包先放到 `/opt/packages`。
- 在线安装驱动、CUDA、fabricmanager、NCCL、DCGM、container-toolkit 前，先装 cuda-keyring。
- 编译驱动、OFED、OpenMPI、nccl-tests、gpu-burn、GDRCopy 前安装：`build-essential`、`linux-headers-$(uname -r)`、`unzip`、`curl`。GDRCopy 还要 `git`、`dkms`、`debhelper`、`devscripts`、`fakeroot`、`dpkg-dev`、`numactl`。
- 安装顺序：OFED → 驱动 → 重启 → CUDA → fabricmanager → NCCL → OpenMPI → nccl-tests → NVSHMEM → nvidia_peermem → PCI ACS → IOMMU → inotify → PFC/ECN → GDRCopy → gpu-burn → DCGM → container-toolkit。
- 已装且版本相同可跳过。驱动、CUDA、fabricmanager、NCCL 装完后 `apt-mark hold`。`.run` 安装的驱动和 CUDA 不归 dpkg 管，不能 hold。
- 驱动在线默认开源模块（`nvidia-driver-<系列>-open`）。闭源去掉包名里的 `-open`，`.run` 不加 `--kernel-module-type=open`。
- 驱动主系列不低于 CUDA 要求：CUDA 13.0 需要 580 系列。fabricmanager 版本与驱动一致。
- NVSHMEM 写入 `/etc/modprobe.d/nvidia.conf` 后要再重启一次才生效。
- VMware 不要写 `iommu=off`，e1000 和 USB 会起不来。没有 NVIDIA GPU 时不要 `modprobe gdrdrv`。PFC/ECN 只给 RoCE，InfiniBand 不要开。

```bash
export DEBIAN_FRONTEND=noninteractive
mkdir -p /opt/packages /opt/build
apt-get update
apt-get install -y build-essential linux-headers-$(uname -r) unzip curl
```

检查：

```bash
. /etc/os-release && echo "$VERSION_ID"    # 22.04
gcc --version | head -n 1
test -d /opt/packages && test -d /opt/build && echo ok
```

## 1. CUDA 软件源

在线安装驱动、CUDA、fabricmanager、NCCL、DCGM、container-toolkit 时需要。

```bash
curl -fsSL -o /opt/packages/cuda-keyring.deb \
  https://developer.download.nvidia.cn/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
apt-get install -y /opt/packages/cuda-keyring.deb
apt-get update
```

检查：`dpkg-query -W cuda-keyring` 状态为 `ii`。

## 2. OFED

必须在 GPU 驱动之前。`mlnx_ofed` 与 `doca_ofed` 二选一。

`mlnx_ofed` 在线装系统 RDMA 栈，版本号不参与安装：

```bash
apt-get install -y rdma-core ibverbs-providers ibverbs-utils \
  infiniband-diags perftest libibverbs-dev librdmacm-dev
```

`mlnx_ofed` 离线用 Ubuntu 22.04 的 tgz：

```bash
tar -C /opt/packages -xzf /opt/packages/MLNX_OFED_LINUX-24.10-3.2.5.0-ubuntu22.04-x86_64.tgz
/opt/packages/MLNX_OFED_LINUX-24.10-3.2.5.0-ubuntu22.04-x86_64/mlnxofedinstall \
  --add-kernel-support --without-fw-update --force
```

`doca_ofed` 在线按 DOCA 版本拼源（如 3.2.2）：

```bash
curl -fsSL https://linux.mellanox.com/public/repo/doca/GPG-KEY-Mellanox.pub \
  | gpg --dearmor -o /etc/apt/trusted.gpg.d/GPG-KEY-Mellanox.gpg
echo "deb [signed-by=/etc/apt/trusted.gpg.d/GPG-KEY-Mellanox.gpg] https://linux.mellanox.com/public/repo/doca/3.2.2/ubuntu22.04/x86_64/ ./" \
  > /etc/apt/sources.list.d/doca.list
apt-get update
apt-get install -y doca-ofed
```

`doca_ofed` 离线把该版本目录下的 deb 全部放进一个目录后 `apt-get install -y ./*.deb`。

检查：

```bash
ofed_info -s || true
dpkg-query -W rdma-core
dpkg-query -W -f='${Status} ${Version}\n' doca-ofed 2>/dev/null || true
ibv_devinfo -l || true
```

## 3. 驱动

在线，开源模块，版本锁定到 580.105.08。闭源把下面包名中的 `-open` 去掉。

```bash
apt-get install -y \
  nvidia-driver-580-open=580.105.08-1ubuntu1 \
  nvidia-dkms-580-open=580.105.08-1ubuntu1 \
  nvidia-kernel-source-580-open=580.105.08-1ubuntu1
reboot
```

源里的修订号不一定是 `-1ubuntu1`，以 `apt-cache madison nvidia-driver-580-open` 为准。用户态子包（`libnvidia-*`、`nvidia-kernel-common` 等）不带 `-open`，有同版本就一起装。

离线 `.run`：

```bash
/opt/packages/NVIDIA-Linux-x86_64-580.105.08.run --silent --dkms \
  --no-questions --no-cc-version-check --kernel-module-type=open
```

装完需要加载内核模块时再重启。`nvidia-smi` 在重启前失败是正常的。

检查（重启后）：

```bash
nvidia-smi --query-gpu=driver_version --format=csv,noheader
modinfo nvidia | awk '/^version:|^license:/{print}'
# 开源模块 license 为 Dual MIT/GPL
```

## 4. CUDA

只装 Toolkit，不装 run 包里捆绑的驱动。

在线：

```bash
apt-get install -y cuda-toolkit-13-0
```

离线：

```bash
sh /opt/packages/cuda_13.0.2_580.95.05_linux.run --silent --toolkit
```

```bash
cat > /etc/profile.d/cuda.sh << 'EOF'
export PATH=/usr/local/cuda/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:/usr/local/cuda/targets/x86_64-linux/lib:$LD_LIBRARY_PATH
EOF
source /etc/profile.d/cuda.sh
```

检查：

```bash
dpkg-query -W cuda-toolkit-13-0 2>/dev/null || true
/usr/local/cuda/bin/nvcc -V
```

## 5. fabricmanager

包名是 `nvidia-fabricmanager`，不要用 Ubuntu multiverse 里带系列号的 server 包。版本与驱动一致。

```bash
apt-get install -y nvidia-fabricmanager=580.105.08-1ubuntu1
systemctl enable --now nvidia-fabricmanager
```

离线：`apt-get install -y /opt/packages/nvidia-fabricmanager_<版本>_amd64.deb`。

检查：

```bash
dpkg-query -W nvidia-fabricmanager
systemctl is-active nvidia-fabricmanager
```

## 6. NCCL

与 OpenMPI、nccl-tests 分开装。

```bash
apt-get install -y libnccl2=2.28.9-1+cuda13.0 libnccl-dev=2.28.9-1+cuda13.0
```

离线：把 `libnccl2` 和 `libnccl-dev` 的 deb 放进一个目录，`apt-get install -y ./*.deb`。

检查：`dpkg-query -W libnccl2 libnccl-dev`。

## 7. OpenMPI

装到 `/usr/local/openmpi`。

```bash
curl -fL -o /opt/packages/openmpi-4.1.7.tar.gz \
  https://download.open-mpi.org/release/open-mpi/v4.1/openmpi-4.1.7.tar.gz
tar -C /opt/build -xzf /opt/packages/openmpi-4.1.7.tar.gz
cd /opt/build/openmpi-4.1.7
./configure --prefix=/usr/local/openmpi
make -j"$(nproc)"
make install
cat > /etc/profile.d/openmpi.sh << 'EOF'
export PATH=/usr/local/openmpi/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/openmpi/lib:$LD_LIBRARY_PATH
EOF
```

检查：`/usr/local/openmpi/bin/mpirun --version`。

## 8. nccl-tests

依赖已安装的 OpenMPI 和 CUDA。装到 `/opt/build/nccl-tests`。

```bash
curl -fL -o /opt/packages/nccl-tests-src.zip \
  https://ghproxy.net/https://github.com/NVIDIA/nccl-tests/archive/refs/heads/master.zip
rm -rf /opt/build/nccl-tests /tmp/nccl-tests-extract
mkdir -p /tmp/nccl-tests-extract
unzip -q /opt/packages/nccl-tests-src.zip -d /tmp/nccl-tests-extract
mv /tmp/nccl-tests-extract/* /opt/build/nccl-tests
cd /opt/build/nccl-tests
make -j MPI=1 MPI_HOME=/usr/local/openmpi CUDA_HOME=/usr/local/cuda
```

检查：`test -x /opt/build/nccl-tests/build/all_reduce_perf && echo ok`。

## 9. NVSHMEM

依赖驱动和 CUDA。引导包按系统拼 `ubuntu2204`。

```bash
curl -fL -o /opt/packages/nvshmem-local-repo.deb \
  https://developer.download.nvidia.cn/compute/nvshmem/3.7.2/local_installers/nvshmem-local-repo-ubuntu2204-3.7.2_3.7.2-1_amd64.deb
dpkg -i /opt/packages/nvshmem-local-repo.deb
cp /var/nvshmem-local-repo-ubuntu2204-3.7.2/nvshmem-*-keyring.gpg /usr/share/keyrings/
apt-get update
apt-get install -y nvshmem-cuda-13
cat > /etc/modprobe.d/nvidia.conf << 'EOF'
options nvidia NVreg_EnableStreamMemOPs=1 NVreg_RegistryDwords="PeerMappingOverride=1;"
EOF
update-initramfs -u
```

然后重启。

检查：

```bash
dpkg-query -W nvshmem-cuda-13
grep EnableStreamMemOPs /etc/modprobe.d/nvidia.conf
# 重启后：
grep EnableStreamMemOPs /proc/driver/nvidia/params    # 应为 1
```

## 10. nvidia_peermem

仅 Ubuntu 22.04。依赖驱动已经加载。

```bash
modprobe nvidia_peermem
printf 'nvidia_peermem\n' > /etc/modules-load.d/nvidia-peermem.conf
```

加载失败时，按已安装的包重装后再 `modprobe`。开源：

```bash
ver=$(dpkg-query -W -f='${Version}' nvidia-dkms-580-open)
apt-get install --reinstall -y "nvidia-dkms-580-open=${ver}"
modprobe nvidia_peermem
```

闭源把包名换成 `nvidia-dkms-580`。不要改成别的版本号。

检查：

```bash
lsmod | grep '^nvidia_peermem'
grep -qx nvidia_peermem /etc/modules-load.d/nvidia-peermem.conf && echo ok
```

## 11. PCI ACS

```bash
apt-get install -y pciutils
install -d /usr/local/sbin
cat > /usr/local/sbin/disable-pcie-acs.sh << 'EOF'
#!/usr/bin/env bash
set -u
for BDF in $(lspci -D | awk '{print $1}'); do
  if setpci -s "$BDF" ECAP_ACS+0x6.w >/dev/null 2>&1; then
    before=$(setpci -s "$BDF" ECAP_ACS+0x6.w 2>/dev/null || echo unknown)
    setpci -s "$BDF" ECAP_ACS+0x6.w=0000 >/dev/null 2>&1 || true
    after=$(setpci -s "$BDF" ECAP_ACS+0x6.w 2>/dev/null || echo unknown)
    echo "$BDF ACSCtl $before -> $after"
  fi
done
EOF
chmod 755 /usr/local/sbin/disable-pcie-acs.sh
/usr/local/sbin/disable-pcie-acs.sh
cat > /etc/systemd/system/disable-pcie-acs.service << 'EOF'
[Unit]
Description=Disable PCIe ACS for GPUDirect RDMA
DefaultDependencies=no
After=local-fs.target
Before=network-pre.target
[Service]
Type=oneshot
ExecStart=/usr/local/sbin/disable-pcie-acs.sh
RemainAfterExit=yes
[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable disable-pcie-acs.service
```

检查：`lspci -vvv | grep ACSCtl | grep SrcValid+` 应无输出。`systemctl is-enabled disable-pcie-acs` 为 `enabled`。

## 12. IOMMU

`off` 与 `pt` 二选一，`off` 必须写成带引号的字符串。按 CPU 厂商改 `/etc/default/grub` 的 `GRUB_CMDLINE_LINUX`，先去掉已有的 `intel_iommu`、`amd_iommu`、`iommu`，再 `update-grub`。重启后 `/proc/cmdline` 才变。

- Intel `off`：`intel_iommu=off iommu=off`
- AMD `off`：`amd_iommu=off iommu=off`
- Intel `pt`：`intel_iommu=on iommu=pt`
- AMD `pt`：`amd_iommu=on iommu=pt`

VMware 客户机不要写 `iommu=off`。示例（Intel，`off`）：

```bash
cp /etc/default/grub /etc/default/grub.bak.$(date +%F_%H%M%S)
sed -i 's/^GRUB_CMDLINE_LINUX="\(.*\)"/GRUB_CMDLINE_LINUX="intel_iommu=off iommu=off \1"/' /etc/default/grub
update-grub
```

先确认原行里没有旧的 `iommu=`。检查：`grep '^GRUB_CMDLINE_LINUX=' /etc/default/grub`。重启后再看 `cat /proc/cmdline`。

## 13. inotify

```bash
cat > /etc/sysctl.d/99-ai-inference-fsnotify.conf << 'EOF'
fs.inotify.max_user_watches = 1048576
fs.inotify.max_user_instances = 8192
fs.inotify.max_queued_events = 65536
EOF
sysctl --system
```

检查：

```bash
sysctl fs.inotify.max_user_watches fs.inotify.max_user_instances fs.inotify.max_queued_events
```

## 14. PFC / ECN

只用于 RoCE，InfiniBand 不要做。下面把优先级 3 打开，网口改成实际的 mlx5 以太网口。

```bash
iface=eth0
mlnx_qos -i "$iface" --trust dscp --pfc 0,0,0,1,0,0,0,0
echo 1 > /sys/class/net/$iface/ecn/roce_np/enable/3
echo 1 > /sys/class/net/$iface/ecn/roce_rp/enable/3
```

检查：

```bash
mlnx_qos -i "$iface"
cat /sys/class/net/$iface/ecn/roce_np/enable/3
cat /sys/class/net/$iface/ecn/roce_rp/enable/3
```

## 15. GDRCopy

CUDA 13.0 不能改用 DMA-BUF mmap，继续用 gdrdrv。没有 NVIDIA GPU（`lspci -n` 无 `10de:`）就不要加载。

```bash
apt-get install -y git dkms debhelper devscripts fakeroot dpkg-dev numactl
git clone --branch v2.6 --depth 1 https://github.com/NVIDIA/gdrcopy.git /usr/local/src/gdrcopy
cd /usr/local/src/gdrcopy/packages
CUDA=/usr/local/cuda-13.0 ./build-deb-packages.sh
dpkg -i gdrdrv-dkms_*_amd64*.deb libgdrapi_*_amd64*.deb gdrcopy-tests_*_amd64*.deb gdrcopy_*_amd64*.deb
apt-get -f install -y
modprobe gdrdrv
printf 'gdrdrv\n' > /etc/modules-load.d/gdrdrv.conf
```

检查：

```bash
lsmod | grep '^gdrdrv'
grep -qx gdrdrv /etc/modules-load.d/gdrdrv.conf && echo ok
gdrcopy_sanity
```

## 16. gpu-burn

装到 `/opt/build/gpu-burn`。

```bash
curl -fL -o /opt/packages/gpu-burn-src.zip \
  https://ghproxy.net/https://github.com/wilicc/gpu-burn/archive/refs/heads/master.zip
rm -rf /opt/build/gpu-burn
mkdir -p /tmp/gpu-burn-extract
unzip -q /opt/packages/gpu-burn-src.zip -d /tmp/gpu-burn-extract
mv /tmp/gpu-burn-extract/* /opt/build/gpu-burn
make -C /opt/build/gpu-burn
```

检查：`test -x /opt/build/gpu-burn/gpu_burn && echo ok`。

## 17. DCGM

```bash
apt-get install -y datacenter-gpu-manager
systemctl enable --now nvidia-dcgm
```

离线用本地 deb：`apt-get install -y /opt/packages/datacenter-gpu-manager_*.deb`。

检查：`dpkg-query -W datacenter-gpu-manager`，`systemctl is-active nvidia-dcgm`。

## 18. nvidia-container-toolkit

会改 docker 运行时并重启 docker。K8s 控制节点慎用。

```bash
apt-get install -y nvidia-container-toolkit
nvidia-ctk runtime configure --runtime=docker
systemctl restart docker
```

有 containerd 时再执行 `nvidia-ctk runtime configure --runtime=containerd` 并重启 containerd。

检查：`dpkg-query -W nvidia-container-toolkit`，`nvidia-ctk --version`。

## 19. 锁定版本

```bash
apt-mark hold nvidia-driver-580-open nvidia-dkms-580-open nvidia-kernel-source-580-open \
  cuda-toolkit-13-0 nvidia-fabricmanager libnccl2 libnccl-dev
```

装了 `doca-ofed` 或 `nvshmem-cuda-13` 时一并 hold。`.run` 安装的驱动和 CUDA 不要 hold。

检查：`apt-mark showhold`。
