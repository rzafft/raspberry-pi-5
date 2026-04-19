# Understanding the AI Hat

The Raspberry Pi AI HAT+ is a hardware accelerator board that enables ai on the edge (i.e. pi can run ai models locally rather than sending data to the cloud). On the board, you see a Hailo 8 chip with 26 tops - a neural network inference processor that can run about 26 trillion ai operations per second. The board simply connects the Hailo 8 chip to the rasberry py via a PCIe ribbon cable. Inference processors like the Hailo 8 are used to run AI models, not train them - that typically happens on GPUs in data centers. So, in simple terms, the AI HAT+ is a dedicated AI co-processor that sits next to the Raspberry Pi and runs neural networks extremely fast while the Pi handles everything else. It is not a GPU, not a training chip, and not something you program like a CPU. It is a model execution engine for vision AI.

<br>
<br>

# The Hailo Software Stack Overview

1. **Hailo Dataflow Compiler (DFS)**

  * Converts trained AI models into optimized .hef files that can run on the Hailo-8.
  * Builds .hef files

2. **HailoRT (Hailo Runtime)**

  * Executes .hef files on the hardware and manages communication between the Raspberry Pi and the Hailo chip.
  * Runs .hef files

3. **Model Zoo**

  * A collection of precompiled .hef models that are ready to run without needing to compile your own.
  * Download .hef files

4. **TAPPAS**

  * A complete, prebuilt application pipeline that integrates camera input, ai inferences using hailoRT, post processing (drawing bounding boxes/lables), and displaying the output

<br>
<br>

# The Hailo Dataflow Compiler (DFC)

The hailo chip is not like a GPU which runs general purpose code (CUDA, etc.). Instead, it runs a precompiled neural network pipeline. So, you must compile it first. That is where the DFC comes in. The DFC takes a trained model (like TensorFlow/ONNX) (1) optimizes it for the Hailo 8 ai processor, (2) then converts it to a .hef file (Hailo Executable Format). This process involves graph transformations, quantization (float > uint8), and hardware specific opitimzations. 

You can download the DFC for the Hailo 8 at https://hailo.ai/developer-zone/software-downloads/. Just filter on thte Hailo 8 device and search for the dataflow compiler. You should see hailo_dataflow_compiler-3.33.1-py3-none-linux_x86_64.whl.

Important note: The DFC is only available for x86 systems (intel/AMD PCs) because its a heavy compilation tools, not something meant to run on small devices like the rp5. So, the intended design for the DFC is to install it on your x86 machine, convert a model to .hef, then move the file to the raspberry pi where it can run via HailoRT.

<br>
<br>

# The Hailo Runtime (HailoRT)

HailoRT is the Hailo runtime that can run .hef files. Think of it as the driver + api that feeds data into the hailo chip and pulls results out.

You can download HailoRT at https://hailo.ai/developer-zone/software-downloads/. Filter on the Hailo 8 device, ARM64 architecture, Linux OS, and Python Version 3.11 (or run `python3 --version` on your rp5). You should see the following available downloads:

1. HailoRT Kernel-Level PCIe Driver (hailort-pcie-driver_4.23.0_all.deb)
2. HailoRT Core Runtime Library (hailort_4.23.0_arm64.deb)
3. HailoRT Python Bindings (hailort-4.23.0-cp311-cp311-linux_aarch64.whl)

<br>

### Step 0: Check system status before downloading anything

You will need all three downloads, but first, check to see what you currently have setup....

1. Execute the `lspci` command in your terminal. This lists all the PCI devices that are connected to your system. When we run it, we should see a line like `0001:01:00.0 Co-processor: Hailo Technologies Ltd. Hailo-8 AI Processor (rev 01)`. This verifies that the raspberry pi can physically see the hailo chip is detected at the hardware level and the PCIe connection is working. This does not mean that a driver has been installed, or that the system can actually use the chip.
2. Execute the `lsmod` command in your terminal. This lists all the kernel modules (linux kernel drivers) that are currently loading. When we run it, we should see a row for `hailo_pci`. However, since, at this point, no hailo driver is loaded we should not see this. Verify this by running `lsmod | grep hailo`. With this, we can conclude that the PCIe driver needs to be installed.
3. Execute the `which hailortcli` command. We get 'command not found. This concludes that the HailoRT command line tools are not installed. These are installed as part of the Hailort core runtime package.

<br>

### Step 1: Install the HailoRT Core Runtime Library

Download hailort_4.23.0_arm64.deb. Next, execute the command to install: `sudo apt install ./hailort_4.23.0_arm64.deb`. After installation run `sudo reboot`.

Once installed, run the following commands to check the installation:

1. Execute `which hailortcli`. Now you should see `/usr/bin/hailortcli`
2. Execute `hailortcli scan`. This asks 'using the hailo runtiem and driver, can i find any connected hailo devices?'. Before we saw 'no command found'. Now, we see `Hailo devices not found.`. With this, we can verify that the runtime is installed and working, but it cannot reach the hardware through the driver (i.e. the driver still needs to be installed).

Next, we need to install the driver. First lets check a few commands to see its status after installing the runtime;

1. Execute `systemctl status hailort.service`.  This command asks systemd 'what is the current state of the hailoRT background service?'. Basically, it is asking if the background manager for the hailo 8 processor is installed, running, if it is started, and what process it is running. We should see that it is loaded (`/lib/systemd/system/hailort.service; enabled`), and that it is active and running.
2. Execute `hailortcli fw-control identify`. This command queries the hailo device firmware and confirms it is detected. Basically it is asking 'can i talk to the firmware running on the hailo chip, and what exactly is it?'. When we run it we see `No device found`.
3. Execute `ls /dev | grep hailo`. This command lists all the device files in /dev and shows any ones related to hailo (in linux, /dev is where hardware devices appear as files). When we run it we see nothing.

<br>

### Step 2: Install the HailoRT PCIe driver

Download hailort-pcie-driver_4.23.0_all.deb. Next, execute the command to install: `sudo apt install ./hailort-pcie-driver_4.23.0_all.deb`. After installation run `sudo reboot`.

Once installed, run the following commands to check the installation:

1. Execute `hailortcli scan`. Now we see `Hailo Devices: [-] Device: 0001:01:00.0`. Now we see the kernel driver (hailo_pci) is loaded and functioning, and HailoRT can communicate with it.
2. Execute `ls /dev | grep hailo`. Now we see `hailo0`. This confirms that the driver is loaded and active, and it has exposed a usable interface to the user space (o.e. programs can open /dev/hailo0 to talk to the chip)

<br>

### Step 3: Install the HailoRT Python bindings

In order to use HailoRT from python, we need the bindings for our specific python version (3.11). To do this, we should use a virtual environment. This is the safe approach; Create a env, install it there:

```
rza@rp5-pios:~ $ python -c 'import hailo_platform; print("Hailo Python ready")'
ModuleNotFoundError: No module named 'hailo_platform'

rza@rp5-pios:~ $ python3 -m venv hailo_env --system-site-packages
rza@rp5-pios:~ $ source hailo_env/bin/activate

(hailo_env) rza@rp5-pios:~ $ cd Downloads/
(hailo_env) rza@rp5-pios:~/Downloads $ pip install ~/Downloads/hailort-4.23.0-cp311-cp311-linux_aarch64.whl

(hailo_env) rza@rp5-pios:~/Downloads $ python -c 'import hailo_platform; print("Hailo Python ready")'
Hailo Python ready
```







