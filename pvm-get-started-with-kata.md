This document provides an overview on how to run Kata Containers with PVM hypervisor.
# Introduction

---

`PVM` is a software virtualization technology that is purpose-built to support `Kata Containers` without the need for hardware virtualization assistance. It is designed as a vendor for `KVM`, similar to `Intel` and `AMD`, making it compatible with the software stack in `Kata Containers`.

# Pre-requisites

---

This document requires the presence of `Kata Containers` and `Containerd` on your system. If you have the necessary environment set up, you can proceed directly to the [PVM configuration](#configure-pvm).

Since the PVM hypervisor is based on `Linux kernel 6.7-rc6`, if you only want to test it, we also provide a pre-configured VM image with Kata Containers and PVM. You can directly proceed to [verify using the VM image](#verify-kata-containers-with-pvm-using-vm-image).
# Configure Kata Containers and Containerd

---

You can follow the [offical guide](https://github.com/kata-containers/kata-containers/blob/main/docs/install/README.md) to install Kata Containers with Containerd. Here, we will the step-by-step [manual installation](https://github.com/kata-containers/kata-containers/blob/main/docs/install/container-manager/containerd/containerd-install.md) process of Kata Containers with Containerd.
## Install Kata Containers.

---

- **Download a release**

You can get a release from the [offical release url](https://github.com/kata-containers/kata-containers/releases), choose a latest release version (eg: 3.2.0).
```bash
$ wget https://github.com/kata-containers/kata-containers/releases/download/3.2.0/kata-static-3.2.0-amd64.tar.xz
```

- **Unpack the downloaded archive**
```bash
$ sudo tar -C / -xvf kata-static-3.2.0-amd64.tar.xz
```
After unpacking the downloaded archive, you will find the binaries in the /opt/kata/bin directory. It is recommended by Kata Containers to create symbolic links for these binaries, so that Containerd can locate them.
```bash
$ sudo ln -s /opt/kata/bin/kata-collect-data.sh /usr/local/bin/
$ sudo ln -s /opt/kata/bin/kata-runtime	/usr/local/bin/
```

- **Check installation**

Check installation by showing version details:
```bash
$ kata-runtime --version
```
## Install Containerd

---

- **Download a release**

You can get a release from the [offical release url](https://github.com/containerd/containerd/releases), choose a latest release version (eg: 1.7.10).
```bash
$ wget https://github.com/containerd/containerd/releases/download/v1.7.10/containerd-1.7.10-linux-amd64.tar.gz
```

- **Unpack the downloaded archive**
```bash
$ sudo tar -C /usr/local -xvf containerd-1.7.10-linux-amd64.tar.gz
```

- **Configure Containerd**

Firstly,  download the standard systemd(1) service file and install it in the`/etc/systemd/system/`directory.
```bash
$ wget https://raw.githubusercontent.com/containerd/containerd/master/containerd.service
$ sudo mv containerd.service /etc/systemd/system/
$ sudo systemctl daemon-reload
```
Secondly, add the necessary files for runtime and VMM configuration. In this step, we will add configurations for `QEMU` and `Cloud Hypervisor`.
```bash
$ cat <<-EOF | sudo tee -a "/usr/local/bin/containerd-shim-kata-v2"
#!/bin/bash
# QEMU (Default VMM)
KATA_CONF_FILE=/opt/kata/share/defaults/kata-containers/configuration.toml /opt/kata/bin/containerd-shim-kata-v2 \$@
EOF

$ sudo chmod +x /usr/local/bin/containerd-shim-kata-v2

$ cat <<-EOF | sudo tee -a "/usr/local/bin/containerd-shim-kata-clh-v2"
#!/bin/bash
# Cloud Hypervisor
KATA_CONF_FILE=/opt/kata/share/defaults/kata-containers/configuration-clh.toml /opt/kata/bin/containerd-shim-kata-v2 \$@
EOF

$ sudo chmod +x /usr/local/bin/containerd-shim-kata-clh-v2
```
Next, add the Kata Containers configuration to the Containerd configuration file (`/etc/containerd/config.toml`).
> **Note：If you don't have the confiurtation file, you can generate it as follows:**

```bash
$ sudo mkdir -p /etc/containerd
$ sudo containerd config default >> /etc/containerd/config.toml
```
Add the following content into the configuration file under `[plugins."io.containerd.grpc.v1.cri".containerd.runtimes]`section. This will configure Containerd to use the Kata runtime.
```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata]
       runtime_type = "io.containerd.kata.v2"
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata-clh]
       runtime_type = "io.containerd.kata-clh.v2"
```

Finally, start the Containerd service.
```bash
$ sudo systemctl start containerd
$ sudo systemctl enable containerd
```

## Verify the installation with KVM

---

- **Download nerdctl**  
It is recommended to use nerdctl CLI to test containers with containerd.
```bash
$ wget https://github.com/containerd/nerdctl/releases/download/v1.7.5/nerdctl-1.7.5-linux-amd64.tar.gz
$ sudo tar Cxzvvf /usr/local/bin nerdctl-1.7.5-linux-amd64.tar.gz
```

- **Verify**  
You are now ready to run Kata Containers with KVM.
First, ensure the `KVM` and `VSOCK` modules are loaded.
```bash
$ sudo modprobe kvm-intel
$ sudo modprobe vhost_vsock # For QEMU
$ sudo modprobe vmw_vmci  # For cloud hypervisor
```
Then you can perform a simple test by running the following commands:
```bash
$ image="docker.io/library/busybox:latest"
$ sudo nerdctl image pull "$image"
$ sudo nerdctl run --net=none --runtime "io.containerd.kata.v2" --rm -t --name test-kata "$image" date
$ sudo nerdctl run --net=none --runtime "io.containerd.kata-clh.v2" --rm -t --name test-kata "$image" date
```
The last command above will show date information in container.
# Configure PVM

---

The PVM hypervisor is a Linux kernel module based on KVM. Currently, it is maintained privately and is not part of the Linux tree. We are working on merging it upstream as soon as possible. In the meantime, it requires a customized guest kernel as well.
## Install PVM hypervisor

---

- **Download source code**

You can obtain the source code from here, which is base on `Linux kernel 6.7-rc6`.
```
git clone https://github.com/virt-pvm/linux.git -b pvm
```

- **Build kernel and modules**

To build the kernel and module, please refer to the [official kernel build documentation](https://www.kernel.org/doc/html/latest/admin-guide/README.html#documentation). In the menuconfig, select `CONFIG_KVM_PVM`. Then, use the make command to build the kernel and module.
> **Note**: PVM is not currently available with PTI (Page Table Isolation). You can either disable PTI during the building process or disable it during the booting later.

```bash
$ make oldefconfig 				# use old config and set new symbols to their default values
$ make menuconfig 				# select kvm-pvm module, which is under Virtualization menu
$ make -j					# build kernel and modules
$ sudo make modules_install install		# install kernel and modules
```

- **Reboot with new kernel**

Add the additional kernel boot parameter `pti=off` to the  kernel cmdline. Then you can reboot the host.

- **Load pvm module**
> **Note:** Currently, kvm-intel.ko and kvm-amd.ko cannot coexist with kvm-pvm.ko. Therefore, you must unload them first before loading kvm-pvm.ko.

```bash
$ sudo rmmod kvm-intel
$ sudo rmmod kvm
$ sudo modprobe kvm-pvm
```
## Install PVM guest kernel

---

- **Download source code**

You can obtain source code from here, which is base on `Linux kernel 6.7-rc6`.
```
git clone https://github.com/virt-pvm/linux.git -b pvm
```

- **Build kernel**

To build the guest kernel, please refer to the [official kernel build documentation](https://www.kernel.org/doc/html/latest/admin-guide/README.html#documentation).  Additionally, we provide a configuration file based on the default configuration for the guest kernel from Kata Containers.
```bash
$ wget https://raw.githubusercontent.com/virt-pvm/misc/main/pvm-guest.config -O .config
$ make olddefconfig
$ make -j vmlinux     # build kernel
```

- **Install kernel**

You can move the guest kernel to the default path for Kata Containers.
```bash
$ sudo cp vmlinux /opt/kata/share/kata-containers/vmlinux.pvm
```
## Verify Kata Containers with PVM

---

## Configure QEMU for PVM
We only support using qboot as the BIOS for QEMU instead of SeaBIOS, so please change the configuration file (`/opt/kata/share/defaults/kata-containers/configuration.toml`) to use qboot. Additionally, please change the guest kernel path option too.
> kernel = "/opt/kata/share/kata-containers/vmlinux.pvm"  
> firmware = "/opt/kata/share/kata-qemu/qemu/qboot.rom"

In addition, there are some issues when using QEMU with PVM. If you wish to test QEMU with `snapshot` and `migration`, we recommend either picking the [commits](https://github.com/qemu/qemu/compare/master...virt-pvm:qemu:pvm_qemu_migration) from our QEMU repository.


## Configure Cloud Hypervisor for PVM
Due to the [virtio-vsock issue](https://github.com/cloud-hypervisor/cloud-hypervisor/issues/5691), the rust vmm (including Dragonball, Cloud Hypervisor and Firecracker) installed in the Kata Container package cannot support PVM guest. We have only found that Cloud Hypervisor has fixed the issue, so we should use version 35.0 or higher of Cloud Hypervisor.
```bash
$ sudo wget https://github.com/cloud-hypervisor/cloud-hypervisor/releases/download/v37.0/cloud-hypervisor-static -O /opt/kata/bin/cloud-hypervisor
```
Then please change the guest kernel option in configuration file (`/opt/kata/share/defaults/kata-containers/configuration.toml`) .
> kernel = "/opt/kata/share/kata-containers/vmlinux.pvm"

**Note：Due to this [issue](https://github.com/virt-pvm/linux/issues/1), the guest fails to boot successfully when using Cloud Hypervisor with a virtio-pmem rootfs under PVM.
However, the workaround in the issue cannot be utilized by Kata Containers at the moment. Therefore, if you wish to test Cloud Hypervisor for PVM, you must use the modified runtime
binary provided in our [package](https://github.com/virt-pvm/misc/releases/download/test/pvm-kata-vm-img.tar.gz).
And add the following option in configuration file.**

> max_phys_bits=43

## Verify Kata Containers with PVM
You can perform a simple test by running the following commands:
```bash
$ image="docker.io/library/busybox:latest"
$ sudo nerdctl image pull "$image"
$ sudo nerdctl run --net=none --runtime "io.containerd.kata.v2" --rm -t --name test-kata "$image" date
$ sudo nerdctl run --net=none --runtime "io.containerd.kata-clh.v2" --rm -t --name test-kata "$image" date
```
The last command above will show date information in container.
# Verify Kata Containers with PVM using VM image

---

We provide a VM image based on the `Official Ubuntu Cloud Image`, which you can use to test Kata Containers with PVM directly.

- **Download VM image**

You can obtain the VM image from the following url.
```bash
$ wget https://github.com/virt-pvm/misc/releases/download/test/pvm-kata-vm-img.tar.gz -O pvm-kata-vm-img.tar.gz
$ tar -xzvf pvm-kata-vm-img.tar.gz
```

- **Start VM**

You can use QEMU to boot it, and run the previous test command. Then, login with the `root` account and the default password is "`root`".
```bash
qemu-system-x86_64 -machine accel=kvm \
	-cpu host \
	-m 4G \
	-smp cores=2,threads=1 \
	-nographic \
	-device virtio-net-pci,netdev=net0 \
	-netdev user,id=net0,hostfwd=tcp::2222-:22 \
	-drive if=virtio,format=qcow2,file=ubuntu-22.04-pvm-kata.img
```

- **Test running**

You can perform a simple test by running the following commands:
```bash
$ image="docker.io/library/busybox:latest"
$ sudo nerdctl run --net=none --runtime "io.containerd.kata.v2" --rm -t --name test-kata "$image" date
$ sudo nerdctl run --net=none --runtime "io.containerd.kata-clh.v2" --rm -t --name test-kata "$image" date
```
The last command above will show date information in container.
