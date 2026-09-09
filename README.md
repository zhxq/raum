# DS-RAID and RAUM

This is the README repository for the paper [https://dl.acm.org/doi/abs/10.1145/3793230.3839383](Revisiting Block-Interface ZNS RAID Systems with Dynamically-Sized RAID (DS-RAID) and Random Accessible Unified Memory (RAUM)).

If you have any questions, you can find me at xzhang84@syr.edu. Have fun hacking!

# Setting up FEMU


Go to https://github.com/zhxq/raum-femu and clone the repository. Follow the vanilla FEMU guide (available in that repository) to set up. After copying scripts from `femu-scripts` to `build-femu`, you can set up the parameters in the `run-zns.sh` under `build-femu`.

IMPORTANT: Parameters in `run-zns.sh`:
```
MAX_ACTIVE=14     # Max active zones
MAX_OPEN=14       # Max open zones 
ZRWA_SZ=1048576   # ZRWA size per open zone (in KB). For RAUM, the total RAUM area will be ZRWA_SZ * NUM_ZRWA (defined below).
ZRWA_FG=-1        # Default=-1, which sets it to ZNS_PAGE_SIZE.
NUM_ZRWA=14       # Max number number of ZRWA. MAX_ACTIVE, MAX_OPEN, and NUM_ZRWA should be set to the same number.
```

The parameter below is in `FEMU_OPTIONS` (also very important):

```
namespaces=2      # Set to 1: one namespace (ZNS only, ZRWA enabled). Set to 2: two namespaces (ZNS and RAUM namespaces, ZRWA disabled).
```

After setting these parameters, don't forget to set up the total number of SSDs.

```
sudo ./qemu-system-x86_64 \
    -name "FEMU-ZNSSD-VM" \
    -enable-kvm \
    -cpu host \
    -smp 4 \
    -m 4G \
    -device virtio-scsi-pci,id=scsi0 \
    -device scsi-hd,drive=hd0 \
    -drive file=$OSIMGF,if=none,aio=native,cache=none,format=qcow2,id=hd0 \
    ${FEMU_OPTIONS} \
    -net user,hostfwd=tcp::6666-:22 \
    -net nic,model=virtio \
    -nographic \
    -virtfs local,path=$(mkdir -p ../hostshare && cd ../hostshare && pwd),mount_tag=host0,security_model=passthrough,id=host0 \
    -device virtio-9p-pci,fsdev=host0,mount_tag=hostshare  \
    -qmp unix:./qmp-sock,server,nowait 2>&1 | tee log
```

The default script initiates only one ZNS SSD. You can spawn multiple SSDs by having multiple lines of `${FEMU_OPTIONS} \`. E.g., this spawns 4 SSDs:
```
sudo ./qemu-system-x86_64 \
    -name "FEMU-ZNSSD-VM" \
    -enable-kvm \
    -cpu host \
    -smp 4 \
    -m 4G \
    -device virtio-scsi-pci,id=scsi0 \
    -device scsi-hd,drive=hd0 \
    -drive file=$OSIMGF,if=none,aio=native,cache=none,format=qcow2,id=hd0 \
    ${FEMU_OPTIONS} \
    ${FEMU_OPTIONS} \
    ${FEMU_OPTIONS} \
    ${FEMU_OPTIONS} \
    -net user,hostfwd=tcp::6666-:22 \
    -net nic,model=virtio \
    -nographic \
    -virtfs local,path=$(mkdir -p ../hostshare && cd ../hostshare && pwd),mount_tag=host0,security_model=passthrough,id=host0 \
    -device virtio-9p-pci,fsdev=host0,mount_tag=hostshare  \
    -qmp unix:./qmp-sock,server,nowait 2>&1 | tee log
```

You should also consider assigning larger DRAM and more CPUs by setting up appropriate numbers for `-m` and `-smp`.


# Setting up Linux Kernel

Our kernel code is based on BIZA.
```
mkdir biza && cd biza
mkdir build
git clone https://github.com/ChaseLab-PKU/BIZA.git
mv BIZA codes && cd codes
```

You might want to disable the code to compile BIZA from BIZA's repository. We have a BIZA branch in our kernel repository that fixes some bugs that prevents BIZA from running correctly. Comment out the following lines in `BIZA/drivers/md/Makefile`.

```
# dm biza added
dm-biza-y	+= dm-biza-target.o dm-biza-map.o dm-biza-gc.o dm-biza-pred.o
# raizn added
raizn-y	+= raizn.o
```

Install kernel compilation toolchain.
```
sudo apt update && sudo apt upgrade
sudo apt-get install git fakeroot build-essential ncurses-dev xz-utils libssl-dev bc flex libelf-dev bison libc6-dev liblz4-tool gcc-11 dwarves
```


Some kernel flags should be turned on or off to pass compilation.

```
scripts/config --set-str SYSTEM_TRUSTED_KEYS ""
scripts/config --disable SYSTEM_TRUSTED_KEYS
scripts/config --disable SYSTEM_REVOCATION_KEYS
scripts/config --disable CONFIG_DEBUG_INFO_BTF
scripts/config --disable CONFIG_PAHOLE_HAS_SPLIT_BTF
scripts/config --disable CONFIG_DEBUG_INFO_BTF_MODULES
scripts/config --enable CONFIG_XOR_BLOCKS
scripts/config --enable CONFIG_RAID6_PQ
scripts/config --enable CONFIG_MD
scripts/config --enable CONFIG_BLK_DEV_MD
scripts/config --enable CONFIG_MD_AUTODETECT
scripts/config --enable CONFIG_MD_RAID456
scripts/config --disable CONFIG_MODULE_SIG
scripts/config --disable CONFIG_MODULE_SIG_ALL
scripts/config --enable CONFIG_MODULES
make olddefconfig
```

Because the kernel is relatively old (version 5.15), you might need to use an older GCC (e.g., GCC 11) to compile the kernel source:

```
make -j$(nproc) HOSTCC=gcc-11 CC=gcc-11 bindeb-pkg
```


After compiling the kernel, install the kernel to your evaluation environment (e.g., the FEMU virtual machine). You also need to install the headers `deb` in your local environment or the virtual machine to compile the kernel module.

Now you should have the environment to compile the kernel module. Clone the kernel module from `https://github.com/zhxq/raum-kernel`. The default branch is `ds_raid`, which contains the driver for DS-RAID and RAUM. If you want to try BIZA, you can use the `biza` branch from our repo (as discussed previously, we fixed some bugs that prevents BIZA from running correctly). The `main` repo, on the other hand, is from the original BIZA repository but in Linux Kernel code/indent style. You can compare the `biza` and the `main` repos to see our changes to BIZA. 

To compile and install the kernel module, run `make -j$(nproc) && make install`. Then you should be able to `insmod` the module. After the `insmod`:

To run with BIZA:
```
echo "0 409600 biza 4 1 64 /dev/nvme0n1 /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1" | sudo dmsetup create biza0
```

To run with RAUM:
```
echo "0 409600 raum 4 1 64 /dev/nvme0n1 /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1 /dev/nvme0n2 /dev/nvme1n2 /dev/nvme2n2 /dev/nvme3n2" | sudo dmsetup create raum0
```

# Notes


Technically, I should use git submodules here and include the two repositories (FEMU and kernel), but I found it to be error prone (especially for users), and the two repos are not dependent on each other. Therefore, I am using this repo for documentation purposes only.