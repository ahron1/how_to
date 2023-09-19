# How to Install NVIDIA CUDA Toolkit on Ubuntu 22.04

## Introduction

CUDA stands for Compute Unified Device Architecture. It is a platform to perform parallel computing on NVIDIA GPUs. Machine Learning programs use the GPU to parallelize and speed up tensor operations. Thus, the CUDA Toolkit is a prerequisite for developing or using modern ML/AI applications, such as Stable Diffusion and Large Language Models (LLMs).

> CUDA comprises two APIs - the [CUDA driver API](https://docs.nvidia.com/cuda/cuda-driver-api/index.html) and the [CUDA runtime API](https://docs.nvidia.com/cuda/cuda-runtime-api/index.html). The runtime API allows applications to plug into the CUDA platform and use GPU computing features in their programs. 

 * On Vultr Cloud GPU servers, the instance creation process installs the NVIDIA GPU drivers and the driver API
 * The CUDA Toolkit installs the CUDA runtime API. Application programmers use the runtime API to take advantage of the GPU for parallel computations.

This guide explains how to install the CUDA Toolkit on a Ubuntu 22.04 server. 

### Prerequisites

To follow this guide and install the CUDA Toolkit, you need to:

 * [Generate SSH keys](https://www.vultr.com/docs/how-do-i-generate-ssh-keys/) and [add the keys to your account](https://www.vultr.com/docs/deploy-a-new-server-with-an-ssh-key/#Add_an_SSH_key_to_your_Vultr_Account), if you haven't already done so
 * Deploy a Vultr instance running Ubuntu 22.04 LTS   
 * [connect to the server over SSH](https://www.vultr.com/docs/connect-to-a-server-using-an-ssh-key/). 

 * [Create a new user with sudo rights](https://www.vultr.com/docs/how-to-use-sudo-on-a-vultr-cloud-server/#Create_a_Sudo_User_on_Debian___Ubuntu). 
 * [Log into the server over SSH as the non-root user](https://www.vultr.com/docs/using-your-ssh-key-to-login-to-non-root-users/)
 * This guide assumes you log in as the user `pythonuser`.

### Installation Methods

You can install the CUDA Toolkit in two ways:

 * natively - directly on the OS
 * using Conda. 

This guide shows both of these methods. To install the CUDA Toolkit using Conda, you also need to:

 * [install Anaconda](google.com) or
 * [install Miniconda](google.com) on the server
 * be comfortable with [using Conda](google.com).

## Install CUDA Toolkit

This section shows how to install CUDA Toolkit version `12.0.1`.

### Install Natively 

In principle, there are two ways of installing the CUDA Toolkit natively: 

 * Using a distribution-independent runfile. This is a standalone installer packaged as a script. It contains only the CUDA Toolkit executable. It does not install any other dependencies, such as drivers. 
 * Using a distribution-specific package. Distribution-specific packages also install the specific driver versions they were compiled against. However, installing a specific driver version overwrites the driver files already on the system. On cloud GPU servers, this is problematic. 

 > To avoid overwriting the installed NVIDIA driver files, install the CUDA Toolkit using the runfile and not the Ubuntu specific installer. 

#### Download the Installer Runfile

On the [CUDA Toolkit Archive page](https://developer.nvidia.com/cuda-toolkit-archive), find the link to the [installer page for CUDA Toolkit 12.0.1](https://developer.nvidia.com/cuda-12-0-1-download-archive). This page interactively guides you to choose the right installation method depending on the OS, architecture, and your needs.

The following steps show how to download the runfile:

 * Choose Linux as the OS.
 * It asks you to choose the architecture. Choose `x86_64`. 
 * Now you get a list of OS Distributions to choose from. Select Ubuntu. 
 * Select the right version of Ubuntu, `22.04`.
 * It asks to choose one of three installer types - `deb(local)`, `deb(network)`, and `runfile(local)`. Choose `runfile(local)`.

After choosing `runfile(local)`, the page shows the installation instructions for the specific version.

#### Install

Download the runfile for version `12.0.1`:

    $ wget https://developer.download.nvidia.com/compute/cuda/12.0.1/local_installers/cuda_12.0.1_525.85.12_linux.run

The runfile is a large file, a few Gigabytes in size. 

Execute the runfile:

    # sh cuda_12.0.1_525.85.12_linux.run

The installer presents an interactive interface to choose the installation options explained below:

 * The installer shows the End User License Agreement (EULA). Use the `Up` and `Down` arrow keys to browse through the EULA. To accept the agreement, type `accept` at the prompt and press `Enter`.
 * It shows the installation options. The top-level options are `Driver`, `CUDA Toolkit 12.0`,  `CUDA Demo Suite 12.0`,   `CUDA Documentation 12.0`, and `Kernel Objects`.
 * Use the `Up` and `Down` arrow keys to navigate through these options. Use the `Left` and `Right` arrow keys to expand and collapse the nested options. Press the `Enter` key to select or unselect an option. The selected options are marked with an `X` at the left. 
 * **Unselect the driver**. Vultr Cloud GPU servers already have the drivers pre-installed.
 * Select CUDA Toolkit. Optionally, if you are developing new programs, also select CUDA Demo Suite and CUDA Documentation. 
 * After selecting the software to be installed, navigate down to the line that reads `Install`
 * Press `Enter` to start the installation.

The CUDA Toolkit is installed in the location `/usr/local/cuda-12.0/`. In general, the installation directory is `/usr/local/cuda-X.Y`. After the installer finishes, it shows a summary report of the installation. The report contains a message about the executable path:

    Please make sure that
    -   PATH includes /usr/local/cuda-12.0/bin
    -   LD_LIBRARY_PATH includes /usr/local/cuda-12.0/lib64, or, add /usr/local/cuda-12.0/lib64 to /etc/ld.so.conf and run ldconfig as root

This is explained in the next section. The report also presents a warning that it did not install the CUDA Driver:

    ***WARNING: Incomplete installation! This installation did not install the CUDA Driver. 
    A driver of version at least 525.00 is required for CUDA 12.0 functionality to work.

This is expected, as you explicitly unselected the option to install the driver. Because Vultr Cloud GPU servers come pre-installed with the right drivers.

To configure your system to work with the new software, update the environment variables to include the newly added libraries. To learn how to do this, go to the section *Configure the System*.

#### Uninstall

To uninstall the CUDA Toolkit, run the file `/usr/local/cuda-12.0/bin/cuda-uninstaller`. 

The uninstaller lets you choose which software to uninstall. Use the arrow keys to navigate and the `Enter` key to mark the options.

 * Mark the software you want to uninstall: `CUDA_Toolkit_12.0`, `CUDA_Demo_Suite_12.0`, and `CUDA_Documentation_12.0`
 * Navigate down to the line that reads `Done`.
 * Press `Enter` to uninstall. 

### Install with Conda

This section shows how to install CUDA Toolkit version `12.0.1` using Conda. The sample commands below assume you are using Conda as the user `pythonuser` in the Conda environment `env1`.

Use the [webpage of the `cuda` package on the `nvidia` channel of Conda](https://anaconda.org/nvidia/cuda). It contains the list of all available versions. Packages corresponding to each version are also organized in individual channels. Each version has a separate channel. Get the command to install version `12.0.1`:

    $ conda install -c "nvidia/label/cuda-12.0.1" cuda -y

This installs the `cuda` package version `12.0.1`. It includes the CUDA Toolkit.

After installing, check if the CUDA Toolkit is installed:

    $ conda list | grep "cuda-toolkit"

The list of installed programs now contains `cuda-toolkit`. The output of the above command is:

    cuda-toolkit              12.0.1                        0    nvidia/label/cuda-12.0.1

Update the environment variables so that your computer recognizes the newly installed software. The next section, *Configure the System* explains how to do this.

### Uninstall

To uninstall the CUDA Toolkit, use the Conda `remove` command:

    $ conda remove cuda -y

## Configure the System 

Application programs use a dynamic link loader to load shared libraries they need to run. The CUDA Toolkit and compiler are dynamically linked libraries. The dynamic link loader uses the environment variable `LD_LIBRARY_PATH` to know where to load these binaries from. Hence, update the `LD_LIBRARY_PATH` environment variable to include the paths of the newly installed CUDA Toolkit libraries. 

The following sections show how to update the `LD_LIBRARY_PATH` and other environment variables.

### Configure CUDA Toolkit Installed Natively 

The [Post-installation Actions section of the Linux Installation Guide](https://docs.nvidia.com/cuda/archive/12.0.1/cuda-installation-guide-linux/index.html#post-installation-actions) shows how to update the environment. You need to update the values of two environment variables:

 * Add the CUDA path to the system `PATH` 
 * Add the CUDA Toolkit library's path to the `LD_LIBRARY_PATH`. 

These changes also need to persist across system reboots. Append the path modification commands to the `.bashrc` file:

    $ echo "export PATH=/usr/local/cuda-12.0/bin${PATH:+:${PATH}}" >> /home/pythonuser/.bashrc
    $ echo "export LD_LIBRARY_PATH=/usr/local/cuda-12.0/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}" >> /home/pythonuser/.bashrc

Close and reopen the terminal session for the PATH and environment variable changes to take effect. Jump to the section *Verify the Installation* to check that CUDA Toolkit is installed and working correctly on your system.

### Configure CUDA Toolkit Installed in Conda

To store environment variables, Conda uses a file `env_vars.sh` in the activation directory, `activate.d`. Each Conda session loads these variables while starting. 

The activation directory doesn't exist by default. Create it:

    $ mkdir -p $CONDA_PREFIX/etc/conda/activate.d

Append the command to update `LD_LIBRARY_PATH` to the `env_vars.sh` file:

    $ echo "export LD_LIBRARY_PATH=$CONDA_PREFIX/lib${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}" >> $CONDA_PREFIX/etc/conda/activate.d/env_vars.sh

Some programs, including the NVIDIA sample programs used to test CUDA installations, use an environment variable `CUDA_PATH` to get the location of the CUDA Toolkit executables. In the absence of this environment variable, they fall back on a default hard-coded path. This hard-coded path is `/usr/local/cuda/bin/` and it is based on native OS installations. It doesn't work for Conda installations because Conda uses environment-specific paths to install binaries. 

Update the `env_vars.sh` file to add the path of the installed CUDA packages to the `CUDA_PATH` environment variable. 

    $ echo "export CUDA_PATH=$CONDA_PREFIX${CUDA_PATH:+:${CUDA_PATH}}" >> $CONDA_PREFIX/etc/conda/activate.d/env_vars.sh

Exit the session and log back into the `env1` environment to activate the new environment variables.

## Verify the installation

Run `nvidia-smi` again:

    $ nvidia-smi

The output should be the same as before. If you (mistakenly) overwrite the default drivers, NVIDIA-SMI gives an error.

The `nvcc` command runs the NVIDIA CUDA Compiler (NVCC). If the CUDA Toolkit is installed correctly, `nvcc` is also installed. Check the installed version of the compiler:

    $ nvcc --version

The output resembles the text below:

    nvcc: NVIDIA (R) Cuda compiler driver 
    Copyright (c) 2005-2023 NVIDIA Corporation 
    Built on Tue_Jul_11_02:20:44_PDT_2023 
    Cuda compilation tools, release 12.0, V12.0.140
    Build cuda_12.0.r12.0/compiler.32267302_0

Check if CUDA-based programs run on the system. NVIDIA provides a set of sample programs to help developers test their CUDA installations. Download the CUDA Samples Git repository:

    $ git clone https://github.com/NVIDIA/cuda-samples.git

Navigate to the `deviceQuery` sample program's directory:

    $ cd cuda-samples/Samples/1_Utilities/deviceQuery

Compile the sample program:

    $ make

Run the compiled executable:

    $ ./deviceQuery

The output resembles the excerpt below:

     CUDA Device Query (Runtime API) version (CUDART static linking)

    Detected 1 CUDA Capable device(s)

    Device 0: "NVIDIA A40-1Q"
        CUDA Driver Version / Runtime Version          12.0 / 12.0
        CUDA Capability Major/Minor version number:    8.6
        .
        .
        .
        Result = PASS

CUDA Toolkit is now successfully installed on the system.

## Check System Compatibility for Other CUDA Toolkit Versions

The version of the CUDA Toolkit that you can install depends on the configuration of your server. The examples in this guide are based on a test server with the following configuration:

 * GCC Version `11.3.0` 
 * NVIDIA graphics driver version `525.125.06`
 * CUDA Driver API Version `12.0`
 * Linux kernel version `5.15.0-75`

Based on the above system configuration, the installation examples show how to install CUDA Toolkit version `12.0.1`. As of September 2023, this is the latest CUDA Toolkit version that you can install on Vultr servers running Ubuntu 22.04 LTS.

Before installing an arbitrary version of the CUDA Toolkit, check if the system can run the software. The following sections show how to do this.

### Check the Hardware 

Peripheral Component Interconnect (PCI) and PCI Express (PCIe) are technology standards used to connect hardware devices, such as GPU cards, to the computer. The `lspci` command shows a list of all PCI devices on the system. Check the list for NVIDIA devices:

    $ lspci | grep -i nvidia

If the system has a CUDA-compatible graphic card, the output of the above command resembles:

    6:00.0 VGA compatible controller: NVIDIA Corporation GA102GL [A40] (rev a1)

The example output above is for a system with an A40 GPU. 

### Check the GCC Version

For developing or compiling CUDA applications, the computer needs to have the GNU Compiler Collection (GCC). On recent Operating Systems, the installed version of GCC is enough. Check if GCC is installed:

    $ gcc --version

If GCC is installed, the output resembles the example below:

    gcc (Ubuntu 11.3.0-1ubuntu1~22.04.1) 11.3.0 
    Copyright (C) 2021 Free Software Foundation, Inc. 
    This is free software; see the source for copying conditions.  
    There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

### Check the NVIDIA Drivers

To check if the system has the right NVIDIA GPU drivers, test if it can access the GPU:

    $ nvidia-smi

`nvidia-smi` shows the details of the GPU. NVIDIA SMI is NVIDIA's System Management Interface. The GPU driver installer installs NVIDIA-SMI during system initialization. Vultr Cloud GPU servers come pre-installed with the right drivers. 

    +-----------------------------------------------------------------------------+
    | NVIDIA-SMI 525.125.06   Driver Version: 525.125.06   CUDA Version: 12.0     |
    |-------------------------------+----------------------+----------------------+
    | GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
    | Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
    |                               |                      |               MIG M. |
    |===============================+======================+======================|
    |   0  NVIDIA A40-1Q       On   | 00000000:06:00.0 Off |                    0 |
    | N/A   N/A    P8    N/A /  N/A |      0MiB /  1024MiB |      0%      Default |
    |                               |                      |             Disabled |
    +-------------------------------+----------------------+----------------------+

The example output above shows in the top row: 

 * The NVIDIA Accelerated Graphics Driver Version is `525.125.06`. 
 * The installed CUDA Driver API version is `12.0`. 

#### Check Driver Version Compatibility

The table [CUDA Toolkit and Corresponding Driver Versions](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html#id5) on the [CUDA Toolkit Release Notes](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html) page shows the minimum driver version needed for each CUDA Toolkit version. Newer drivers also support older versions of the CUDA Toolkit. 

 * Based on [table](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html#id5) mentioned above, CUDA Toolkit version `12.2 Update 1` needs a driver version that is higher than `535.86.10`. 
 * The same table also shows CUDA Toolkit version `12.0.x` needs a driver version that is higher than `525.60.13`. 
 * The example output of NVIDIA SMI (shown in the previous section) shows the driver version as `525.125.06`. 

Thus, on this test server: 

 * The installed drivers are **insufficient** for CUDA Toolkit version `12.2 Update 1`. 
 * The test server configuration supports version `12.0.1`.

### Check the Kernel

CUDA Toolkit needs a relatively recent version of the Linux kernel. The kernel version installed on the computer must be higher than the minimum required kernel version for the specific version of the CUDA Toolkit.

Check the kernel version of your OS:

    $ uname -r

The output resembles the example below:

    5.15.0-75-generic 

Based on the sample output above, the Linux kernel version is `5.15.0-75`. 

The previous section showed that CUDA Toolkit `12.0 Update 1` is the latest version supported by the installed drivers (`525.125.06`). Confirm that the installed kernel also supports this CUDA Toolkit version (`12.0.1`). 

 * Check the table [Native Linux Distribution Support in CUDA 12.0.1](https://docs.nvidia.com/cuda/archive/12.0.1/cuda-installation-guide-linux/index.html#id13) on the [CUDA Installation Guide for Linux for version `12.0.1`](https://docs.nvidia.com/cuda/archive/12.0.1/cuda-installation-guide-linux/index.html).
 * The [table](https://docs.nvidia.com/cuda/archive/12.0.1/cuda-installation-guide-linux/index.html#id13) shows that for CUDA Toolkit `12.0.1`, for Ubuntu 22.04 LTS, the minimum required kernel version is `5.15.0-43`. 
 * The installed kernel version on the test server is `5.15.0-75`. 

Hence, CUDA Toolkit version `12.0.1` is compatible with the kernel on this server.

#### Check the Kernel Headers 

Kernel headers are the interface for libraries to interact with the Linux kernel. The kernel headers must correspond to the installed kernel. If the system does not have the right kernel headers, manually install them. Check if kernel headers corresponding to the system's kernel version are also installed:

    $ apt list linux-headers-$(uname -r)

If the right headers are installed for kernel `5.15.0-75`, the output of the above command resembles the example below:
    
    linux-headers-5.15.0-75-generic/jammy-updates,jammy-updates,jammy-security,jammy-security,now 5.15.0-75.82 amd64 [installed]

If the right version of the headers is not installed, install it:

    # apt-get install linux-headers-$(uname -r)

### Choose the right version

To decide which version of the CUDA Toolkit to install on Vultr Cloud GPU servers, consider the following:

 * The CUDA Toolkit version that the NVIDIA Graphics Drivers support
 * The CUDA Toolkit version that the kernel supports
 * The requirements of the application(s) you want to run

The [CUDA Toolkit Archive](https://developer.nvidia.com/cuda-toolkit-archive) page has links to the download page and documentation page of all CUDA Toolkit releases, including the latest version. 

## Conclusion

This guide gave an overview of the installation process for the CUDA Toolkit. The sample commands in the previous sections are based on installing a specific version (`12.0.1`) of the CUDA Toolkit. To install an arbitrary version of the CUDA Toolkit, follow the same instructions to get the respective documentation page, the downloads page, and the installation commands. 

Because Machine Learning is a dynamic and rapidly evolving field, tools are frequently updated and incompatibilities are common. It is important to check which version of the CUDA Toolkit you need, given the constraints of your system and the requirements of your application.

### More Information

 * The [NVIDIA CUDA Installation Guide for Linux](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html) discusses the installation process and compatibility checks. 
 * To understand compatibility issues better, refer to NVIDIA's documentation on [CUDA Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/). 
 * To understand better the difference between the [runtime API](https://docs.nvidia.com/cuda/cuda-runtime-api/index.html) and the [driver API](https://docs.nvidia.com/cuda/cuda-driver-api/index.html), refer to the [official documentation comparing them](https://docs.nvidia.com/cuda/cuda-runtime-api/driver-vs-runtime-api.html).
