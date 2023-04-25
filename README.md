# Firecracker


## Getting Started

To get started with Firecracker, download the latest
[release](https://github.com/firecracker-microvm/firecracker/releases) binaries
or build it from source.

Install docker
```
sudo apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu `lsb_release -cs` test"
sudo apt update
sudo apt install docker-ce
```

You can build Firecracker on any Unix/Linux system that has Docker running
(we use a development container) and `bash` installed, as follows:

```bash
git clone https://github.com/firecracker-microvm/firecracker
cd firecracker
tools/devtool build
toolchain="$(uname -m)-unknown-linux-musl"
```

The Firecracker binary will be placed at
`build/cargo_target/${toolchain}/debug/firecracker`. For more information on
building, testing, and running Firecracker, go to the
[quickstart guide](docs/getting-started.md).

The overall security of Firecracker microVMs, including the ability to meet the
criteria for safe multi-tenant computing, depends on a well configured Linux
host operating system. A configuration that we believe meets this bar is
included in [the production host setup document](docs/prod-host-setup.md).

## Test

To test if firecracker works, in one terminal:

```
rm /tmp/firecracker.socket
firecracker --api-sock /tmp/firecracker.socket
```

In another terminal, run this bash:

```
#!/bin/bash

# get the kernel and rootfs
#
arch=`uname -m`
dest_kernel="hello-vmlinux.bin"
dest_rootfs="hello-rootfs.ext4"
image_bucket_url="https://s3.amazonaws.com/spec.ccfc.min/img"

if [ ${arch} = "x86_64" ]; then
    kernel="${image_bucket_url}/hello/kernel/hello-vmlinux.bin"
    rootfs="${image_bucket_url}/hello/fsfiles/hello-rootfs.ext4"
elif [ ${arch} = "aarch64" ]; then
    kernel="${image_bucket_url}/aarch64/ubuntu_with_ssh/kernel/vmlinux.bin"
    rootfs="${image_bucket_url}/aarch64/ubuntu_with_ssh/fsfiles/xenial.rootfs.ext4"
else
    echo "Cannot run firecracker on $arch architecture!"
    exit 1
fi

echo "Downloading $kernel..."
curl -fsSL -o $dest_kernel $kernel

echo "Downloading $rootfs..."
curl -fsSL -o $dest_rootfs $rootfs

echo "Saved kernel file to $dest_kernel and root block device to $dest_rootfs."

# set the guest kernel
#
arch=`uname -m`
kernel_path=$(pwd)"/hello-vmlinux.bin"

if [ ${arch} = "x86_64" ]; then
    curl --unix-socket /tmp/firecracker.socket -i \
      -X PUT 'http://localhost/boot-source'   \
      -H 'Accept: application/json'           \
      -H 'Content-Type: application/json'     \
      -d "{
            \"kernel_image_path\": \"${kernel_path}\",
            \"boot_args\": \"console=ttyS0 reboot=k panic=1 pci=off\"
       }"
elif [ ${arch} = "aarch64" ]; then
    curl --unix-socket /tmp/firecracker.socket -i \
      -X PUT 'http://localhost/boot-source'   \
      -H 'Accept: application/json'           \
      -H 'Content-Type: application/json'     \
      -d "{
            \"kernel_image_path\": \"${kernel_path}\",
            \"boot_args\": \"keep_bootcon console=ttyS0 reboot=k panic=1 pci=off\"
       }"
else
    echo "Cannot run firecracker on $arch architecture!"
    exit 1
fi

# set the guest rootfs
#
rootfs_path=$(pwd)"/hello-rootfs.ext4"

curl --unix-socket /tmp/firecracker.socket -i \
  -X PUT 'http://localhost/drives/rootfs' \
  -H 'Accept: application/json'           \
  -H 'Content-Type: application/json'     \
  -d "{
        \"drive_id\": \"rootfs\",
        \"path_on_host\": \"${rootfs_path}\",
        \"is_root_device\": true,
        \"is_read_only\": false
   }"

# The default microVM will have 1 vCPU and 128 MiB RAM.
# If you want to customize the VM size:
#
#curl --unix-socket /tmp/firecracker.socket -i  \
#  -X PUT 'http://localhost/machine-config' \
#  -H 'Accept: application/json'            \
#  -H 'Content-Type: application/json'      \
#  -d '{
#      "vcpu_count": 2,
#      "mem_size_mib": 1024,
#      "ht_enabled": false
#  }'

# start the guest machine
#
curl --unix-socket /tmp/firecracker.socket -i \
  -X PUT 'http://localhost/actions'       \
  -H  'Accept: application/json'          \
  -H  'Content-Type: application/json'    \
  -d '{
      "action_type": "InstanceStart"
   }'
```

## Make Image
```
git clone https://github.com/torvalds/linux.git
cd linux
git checkout v5.15.0
cp ../firecracker/resources/guest_configs/microvm-kernel-x86_64-4.14.config .config
make vmlinux 
# make Image 
```


```
# 50 MB
dd if=/dev/zero of=rootfs.ext4 bs=1M count=50 
mkfs.ext4 rootfs.ext4
mkdir /tmp/my-rootfs
sudo mount rootfs.ext4 /tmp/my-rootfs
```


```
# 2GB 22.04 
/data02/firecracker/tools# ./devtool build_rootfs -s 2
cp  /data02/firecracker/build/rootfs/bionic.rootfs.ext4 /data02
mount bionic.rootfs.ext4 /tmp/my-rootfs/
cp -R /root/umk-bench /tmp/my-rootfs/
cp -R /usr/*  /tmp/my-rootfs/usr/
```

## Test new Image


To test if firecracker works, in one terminal:

```
rm /tmp/firecracker.socket
firecracker --api-sock /tmp/firecracker.socket
```

In another terminal, run this bash:

```
# 设置虚拟器内核
set -m 
set -x

kernel_path=/data02/linux.git/vmlinux
# kernel_path=/data02/hello-vmlinux.bin
curl --unix-socket /tmp/firecracker.socket -i \
    -X PUT 'http://localhost/boot-source'   \
    -H 'Accept: application/json'           \
    -H 'Content-Type: application/json'     \
    -d "{
        \"kernel_image_path\": \"${kernel_path}\",
        \"boot_args\": \"console=ttyS0 reboot=k panic=1 pci=off\"
      }"
# 设置虚拟机根文件系统
# rootfs_path=/data02/hello-rootfs.ext4 
rootfs_path=/data02/bionic.rootfs.ext4_firecracker_2G 
# rootfs_path=/data02/rootfs.ext4
curl --unix-socket /tmp/firecracker.socket -i \
  -X PUT 'http://localhost/drives/rootfs' \
  -H 'Accept: application/json'           \
  -H 'Content-Type: application/json'     \
  -d "{
        \"drive_id\": \"rootfs\",
        \"path_on_host\": \"${rootfs_path}\",
        \"is_root_device\": true,
        \"is_read_only\": false
   }"
# 可选的配置虚拟机的资源
curl --unix-socket /tmp/firecracker.socket -i  \
  -X PUT 'http://localhost/machine-config' \
  -H 'Accept: application/json'            \
  -H 'Content-Type: application/json'      \
  -d '{
      "vcpu_count": 1,
      "mem_size_mib": 2048
  }'
# 运行虚拟机
curl --unix-socket /tmp/firecracker.socket -i \
  -X PUT 'http://localhost/actions'       \
  -H  'Accept: application/json'          \
  -H  'Content-Type: application/json'    \
  -d '{
      "action_type": "InstanceStart"
   }'

```
