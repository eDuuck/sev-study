# SEV-Step: A Single-Stepping Framework for AMD-SEV

SEV-Step makes interactive single-stepping, page fault tracking and eviction set-based cache attacks available in a single, reusable framework. For more information, check out the [paper](https://arxiv.org/pdf/2307.14757.pdf).
If you use the framework in your research, please cite it as:

```bibtex
@article{Wilke_Wichelmann_Rabich_Eisenbarth_2023,
    title   = {SEV-Step A Single-Stepping Framework for AMD-SEV},
    author  = {Wilke, Luca and Wichelmann, Jan and Rabich, Anja and Eisenbarth, Thomas},
    journal = {IACR Transactions on Cryptographic Hardware and Embedded Systems},
    volume  = {2024},
    number  = {1},
    url     = {https://tches.iacr.org/index.php/TCHES/article/view/11250},
    DOI     = {10.46586/tches.v2024.i1.180-206},
    year    = {2023},
    month   = {Dec.},
    pages   = {180–206}
}
```

This meta repo tracks a compatible state of all SEV-Step components and contains scripts to install everything required to setup an SEV VM.

This manual was tested on a Dell PowerEdge R6515 with a AMD EPYC 7763 CPU running Ubuntu 22.04.2 LTS.

This repo uses git submodules. Run `git submodule update --init --recursive` to ~~catch~~ fetch them all.

## Build Hypervisor Components

In this section you will install the modified SEV-Step kernel, as well as compatible stock versions of QEMU and OVMF/edk2. There are pre-built .deb files for the kernel in the artifact section of this repo. They were built based on the config of Ubuntu 22.04.2 LTS. If you run into any issues with the pre-built binaries, you can also build the kernel yourself, based on your currently active config (see step 2).

0) First install any missing build dependencies (tested on Ubuntu 22.04):

```
# apt install build-essential ninja-build python-is-python3 flex bison libncurses-dev gawk openssl libssl-dev dkms libelf-dev libudev-dev libpci-dev libiberty-dev autoconf llvm
# sed -i '/deb-src/s/^# //' /etc/apt/sources.list && apt update
# apt build-dep ovmf qemu-system-x86 linux
```

1) Run `./build.sh ovmf`, `./build.sh qemu` to build OVMF and QEMU. All packages are only installed locally in `./local-installation/`.
2) If you don't use the pre-built kernel packages run `./build.sh kernel` to build the kernel based on the config of the currently active kernel. Afterwards install the packages using `dpkg -i ./kernel-packages/*.deb`
   
   *If the kernel build fails to compile, it may be because you downgrade from a newer kernel. A simple fix for this is to downgrade to linux 5.18 by using [Mainline](https://github.com/bkw777/mainline), boot into the 5.18 kernel then run `./build.sh kernel`.*  
4) Create the config file `/etc/modprobe.d/kvm.conf` with content 
```
# Enable SEV Support
options kvm_amd sev-snp=1 sev=1 sev-es=1

# Pagetracking code for SEV-Step does not work if this option is set
# Context: there seems to be a transition in the page table management code.
# Our patch is for the old version that is phased out if this flag is set
options kvm tdp_mmu=0                    
```
4) Boot into the new kernel. It is named `...-sev-step-<git commit>`

## Enable SEV-SNP Hardware Support

The following is based on [the official guidelines from AMD](https://github.com/AMDESE/AMDSEV/tree/snp-latest#prepare-host).

To enable SEV on your system, you might need to change some BIOS options.
Usually, there is a general "SEV Enabled" option as well as a `SEV-ES ASID space limit` option and a `SNP Memory Coverage`. `SEV-ES ASID space limit` should be greater than `1`.

If SEV is enabled, you  should get values similar to the following when running the specified commands on the host system. (Make sure you already rebooted into the new kernel).

```
# dmesg | grep -i -e rmp -e sev
SEV-SNP: RMP table physical address 0x0000000035600000 - 0x0000000075bfffff
ccp 0000:23:00.1: sev enabled
ccp 0000:23:00.1: SEV-SNP API:1.51 build:1
SEV supported: 410 ASIDs
SEV-ES and SEV-SNP supported: 99 ASIDs
# cat /sys/module/kvm_amd/parameters/sev
Y
# cat /sys/module/kvm_amd/parameters/sev_es 
Y
# cat /sys/module/kvm_amd/parameters/sev_snp 
Y
```

## Create and Run an SEV VM

In this section, we will setup an SEV-SNP VM using the toolchain we just built.

1) Create a disk for the VM with `./local-installation/usr/local/bin/qemu-img  create -f qcow2 <VM_DISK.qcow2> 20G`
2) Start VM with `sudo ./launch-qemu.sh -hda <VM_DISK.qcow2> -cdrom <Ubuntu 22.10 iso> -vnc :1` where `<Ubuntu 22.10 iso>` is the path
to the regular Ubuntu installation iso. While you can use other Distros, their SEV support might vary. The most important part is, that they ship at least
kernel version 5.19, as this is the first mainline kernel version that supports running as an SEV-SNP guest.
3) Connect to the VM via a VNC viewer on port `5901` and perform a standard installation. Alternatively, to perform a text-only installation, add `console=ttyS0` to the guest kernel command line when booting in Grub.
4) Once the installation is done, terminate qemu with `ctrl a+x` or use `sudo kill $(pidof qemu-system-x86)`
4) Start the VM again and connect with VNC. Install the OpenSSH server and configure it to autostart. The `./launch-qemu.sh` script already forwards VM port 22 to host port 2222 and VM port 8080 to host port 8080
5) Use `sudo ./launch-qemu.sh -hda <VM_DISK.qcow2> -sev-snp` to start the VM with SEV-SNP protection. You might want to also supply the `-allow-debug`
which enables the SEV debug API. Many **test functions** of the SEV-Step framework require this to get access to the VM's instruction pointer or other register content.

## Use the SEV-Step Library

See the [sev-step-userland](https://github.com/sev-step/sev-step-userland/) submodule to learn how to use the SEV-Step library.

There also is an [experimental version](https://github.com/sev-step/sev-step-rust-userland) of this library that tries to make single-stepping easier by providing an API with high level abstractions. 
