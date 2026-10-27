本操作是基于h200-18，注意各个指令的参数，nmve盘符和/data实际使用的nvme盘或者分区名称。

---
1. lvm重建前准备
1.1 预装软件包
#安装性能测试和存储管理工具
#rpm系
dnf install -y fio mdadm lvm2 util-linux nvme-cli parted
#dpkg系
apt install -y fio mdadm lvm2 util-linux nvme-cli parted

# 确认工具可用
fio --version
mdadm --version
lvm version
nvme list
parted --version
1.2 检查挂载占用+卸载挂载
#1、检查本地挂载情况
lsblk
df -Th
#看全部挂载点
findmnt
#看挂载点和递归查看下级目录挂载情况，nfs的也能看到
findmnt -R /data
#输出示例
# [root@i9i5iuwot ~]# findmnt -R /data
# TARGET                 SOURCE                   FSTYPE OPTIONS
# /data                  /dev/nvme0n1             ext4   rw,relatime
# ├─/data/slpan          /dev/nvme1n1             ext4   rw,relatime
# └─/data/test-mount-nfs 1.117.70.113:/data/slpan nfs4   rw,relatime,vers=4.2,rsize=262144,wsize=262144,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=10.12.192.7,local_lock=none,addr=1.117.70.113


#2、再次检查是否还有进程使用/data，fuser可能识别不到k8s进程
fuser -vm /data
#检查LVM 是否仍被其他挂载命名空间引用，可以检查到k8s容器进程
#open不为0就表示还有引用
dmsetup info -c -o name,open,major,minor,attr
#找具体占用进程，253:0替换为lsblk看到的实际lsblk看到的MAJ:MIN（内核用于识别设备的编号）
#可以查找各进程挂载命名空间中是否存在该设备的挂载记录，这样就能找到其他命名空间的容器进程了
grep -H '253:0' /proc/[0-9]*/mountinfo 2>/dev/null | head -n 30


#3、/data数据不做保留，可以先尝试卸载挂载
#如果 umount /data 报 target is busy，不要强制卸载
umount /data
#输出/data is not a mountpoint，说明卸载成功
mountpoint /data
为什么fuser识别不到k8s容器的挂载？
对于在其他 mount namespace 中挂载的块设备，fuser 可能无法识别相应进程，因为进程文件描述符中的设备号与 fuser 用来匹配的设备号可能不同。
https://www.man7.org/linux/man-pages/man1/fuser.1.html
[图片]
1.3 卸载挂载后再检查挂载占用，定位进程
#1、再次检查 LVM 是否仍被其他挂载命名空间引用，大概率是k8s容器进程
#卸载挂载后，检查是否还有进程用lv，open不为0就表示还有引用
dmsetup info -c -o name,open,major,minor,attr

#2、找占用进程，253:0替换为lsblk看到的实际lsblk看到的MAJ:MIN（内核用于识别设备的编号）
grep -H '253:0' /proc/[0-9]*/mountinfo 2>/dev/null | head -n 30
# 示例输出：
# /proc/1000/mountinfo:44 40 259:0 / /data/slpan rw,relatime shared:171 - ext4 /dev/nvme1n1 rw
# /proc/1001866/mountinfo:44 40 259:0 / /data/slpan rw,relatime shared:171 - ext4 /dev/nvme1n1 rw

#3、如果还有进程占用的话根据pid定位进程，如果是k8容器进程，查containerd-》pod
cat /proc/<pid>/cgroup
#示例输出
# root@h200-18:~# cat /proc/2319764/cgroup 
# 0::/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod0e596601_8cfd_4bef_a09a_752bb82710a3.slice/cri-containerd-df29f3b555dd666d8c831bd84954dcf902ab6467b788d973e3f23b496c0dffb5.scope

#下面容器id取自看cgroup输出的cri-containerd-后面的数字
ctr --address /run/k3s/containerd/containerd.sock -n k8s.io containers info f9c959fd137ad9eeb36fee7581dd05d5a830559f07f9e7084d9f71ee899066c6 |
jq -r '.Labels | to_entries[] | select(.key == "io.kubernetes.pod.name" or .key == "io.kubernetes.pod.namespace" or .key == "io.kubernetes.container.name" or .key == "io.kubernetes.pod.uid") | "\(.key)=\(.value)"'
2. 重建条带化lv
2.1 只删原有lv，不动pv、vg、part
2.1.1 清理原本lvm
#只删lv，不动vg、pv，反正重做lv也要格式化文件系统的
#如果要pv、vg、part全清理，看本文档下一节
lvremove /dev/vg_nvme/lv_data
2.1.2 创建新的条带化lv
#条带大小128k，vg空间全分给lv
#以免vg里面包含数个pv，建议在做lv1的时候尾部指定pv名（pvdisplay可看到pv名）
lvcreate --type striped -i 2 -I 128k -l 100%FREE -n lv_data vg_nvme /dev/nvme0n1p1 /dev/nvme1n1p1
#lv要做的刚好是vg包含的全部pv，尾部就不用接pv名
lvcreate --type striped -i 2 -I 128k -l 100%FREE -n lv_data vg_nvme

#检查lv，确认Stripe列是128k
lvs -a -o vg_name,lv_name,lv_size,segtype,stripes,stripesize,devices vg_nvme
lvdisplay
2.2 如果需要pv、vg、part全清
2.2.1 第一种情况，保留原本主分区记录，从分区上建立lvm
#删lv，vg，pv
lvremove /dev/vg_nvme/lv_data
vgremove vg_nvme
pvremove /dev/nvme0n1p1 /dev/nvme1n1p1
#检查，如果此处操作报错，或者原来的 VG 仍然存在，不要直接继续分区
pvs -o pv_name,vg_name,pv_size,pv_free
--------------------------------
#重建分区
#检查块设备签名
wipefs -n /dev/nvme0n1 /dev/nvme1n1
#nvme0n1，在/dev/nvme0n1上创建新的 GPT
#mklabel gpt 会改写分区表（替换原有gpt分区表），需确认数据不需要保留了再操作
parted -s /dev/nvme0n1 mklabel gpt
#创建一个占用整块磁盘的分区，执行后生成/dev/nvme0n1p1
parted -s /dev/nvme0n1 mkpart primary 1MiB 100%
#含义是将第 1 个分区的 LVM 标志设为开启，让分区表标记它用于Linux LVM。
#设置 lvm on 并不等于创建了 LVM 物理卷（PV）
parted -s /dev/nvme0n1 set 1 lvm on

#nvme1n1
parted -s /dev/nvme1n1 mklabel gpt
parted -s /dev/nvme1n1 mkpart primary 1MiB 100%
parted -s /dev/nvme1n1 set 1 lvm on

#通知内核重读分区表
partprobe /dev/nvme0n1 /dev/nvme1n1
udevadm settle
#分区后再次检查块设备签名
wipefs -n /dev/nvme0n1 /dev/nvme1n1
#检查新分区
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS /dev/nvme0n1 /dev/nvme1n1
-------------------------------------
#在分区上创建条带lv
pvcreate /dev/nvme0n1p1 /dev/nvme1n1p1
pvs -o pv_name,vg_name,pv_size,pv_free
vgcreate vg_nvme /dev/nvme0n1p1 /dev/nvme1n1p1
vgs -o vg_name,vg_size,vg_free

lvcreate --type striped -i 2 -I 128k -l 100%FREE -n lv_data vg_nvme /dev/nvme0n1p1 /dev/nvme1n1p1

#检查lv
lvs -a -o vg_name,lv_name,lv_size,segtype,stripes,stripesize,devices vg_nvme
2.2.2 第二种情况，不保留原本主分区记录，从裸盘上建立lvm
#删lv，vg，pv
lvremove /dev/vg_nvme/lv_data
vgremove vg_nvme
pvremove /dev/nvme0n1p1 /dev/nvme1n1p1
#检查，如果此处操作报错，或者原来的 VG 仍然存在，不要直接继续分区
pvs -o pv_name,vg_name,pv_size,pv_free
--------------------------------
#单纯只是清除分区，后续创lvm用裸盘
#检查块设备是否有签名，有输出谨慎操作
wipefs -n /dev/nvme0n1 /dev/nvme1n1

#删除分区两种情况，我们这里用第二种直接给nvme恢复裸盘状态：
#1、从分区表中删除第 1 个分区条目，保留 GPT 分区表
#只想删除某个分区
#nvme0n1清除分区
parted -s /dev/nvme0n1 rm 1
#nvme1n1清除分区
parted -s /dev/nvme1n1 rm 1
#2、直接抹除整个/dev/nvme0n1 /dev/nvme1n1块设备的签名，恢复裸盘
#想清除磁盘上的分区表签名/文件系统签名，恢复为系统无法识别分区的状态就做这个，谨慎操作
#但是如果分区/dev/nvme1n1p1是已经做了mkfs.ext4等已有文件系统的，此次对/dev/nvme1n1的签名抹除并不会抹除实际的分区的签名，但是分区路径确实是找不到了。重建分区后，wipefs -n /dev/nvme1n1p1又会有文件系统的显示，这时还得执行一遍wipefs -a /dev/nvme1n1p1才算清理干净
wipefs -a /dev/nvme0n1 /dev/nvme1n1

#通知内核重读分区表
partprobe /dev/nvme0n1 /dev/nvme1n1
udevadm settle

#检查磁盘划分
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS /dev/nvme0n1 /dev/nvme1n1
-------------------------------------
#在裸盘上创建条带lv
pvcreate /dev/nvme0n1 /dev/nvme1n1
pvs -o pv_name,vg_name,pv_size,pv_free
vgcreate vg_nvme /dev/nvme0n1 /dev/nvme1n1
vgs -o vg_name,vg_size,vg_free

lvcreate --type striped -i 2 -I 128k -l 100%FREE -n lv_data vg_nvme /dev/nvme0n1 /dev/nvme1n1

#检查lv
lvs -a -o vg_name,lv_name,lv_size,segtype,stripes,stripesize,devices vg_nvme
3. 挂载/data
3.1 持久化挂载
#格式化lv为文件系统，ext4/xfs这些看需求定，大文件xfs
mkfs.ext4 -F /dev/vg_nvme/lv_data
#获取uuid
blkid /dev/vg_nvme/lv_data

#临时挂载
mkdir -p /data
mount /data

#挂载加持久化
vi /etc/fstab
UUID=<新UUID> /data ext4 defaults 0 2
#刷新挂载验证持久化
systemctl daemon-reload
umount /data
mount -a
lsblk
3.2 验证挂载可用
#检查
lsblk -f
df -hT /data
lvs -a -o vg_name,lv_name,lv_size,segtype,stripes,stripesize,devices vg_nvme

#写文件测试
echo "a">/data/a
cat /data/a
rm /data/a
