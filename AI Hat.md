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

### Step 0: Check the PCIe connection to the hailo device

1. `lspci`: This lists all the PCI devices that are connected to your system. E.g. if we were to run `lspci | grep hailo`, we would be asking linux 'Do you see the Hailo chip on the PCIe bus at all?'. When we run it, we should see a line like `0001:01:00.0 Co-processor: Hailo Technologies Ltd. Hailo-8 AI Processor (rev 01)`. This verifies that the raspberry pi can physically see the hailo chip and ensures that it is detected at the hardware level and the PCIe connection is working. This does not mean that a driver has been installed, or that the system can actually use the chip.

<br>

### Step 1: Install the PCIe Driver

1. `sudo apt install ./hailort-pcie-driver_4.23.0_all.deb`
2. `sudo reboot`

<br>

### Step 2: Check PCIe driver installation

1. `lsmod`: This lists all the currently loaded linux kernel modules (drivers). Look for `hailo_pci 131072 0`. It shows the module/driver namee (hailo_pci), how much memory the driver uses in RAM, and how many other modules or processes are currently using it. This confirms that the driver was installed correctly, the kernel succesfully loaded it, and the sytem is ready to talk to the Hailo hardware.
   
2. `dmesg | grep hailo`: This shows the kernel log messages that contain 'hailo'. We should see that the hailo_pci driver is loaded, the pcei device is detected, the memory is mapped correctly, DMA is configured, firmware is uploaded, ai core is booted, and that the device is exposed as /dev/hailo0.
   
3. `ls /dev | grep hailo`. Linux exposes hardware as files inside /dev. This command checks 'did linux create a usable interface for the hailo chip?'. When we run it, we see `hailo0`. Thus, /dev/hailo0 is a special device file that programs can open like a file. HailoRT will use it to send inference jobs. This confirms that the driver is working, firmware is running, and hardware is accessible to applications.

4. `lspci -k | grep -A 3 -i hailo`. This asks 'which driver is controlling the pcie device'? We shouldl see something like `Kernel driver in use: hailo_pci`.  This confirms that the driver didn't just load, it is actually bound to the hardware.

5. `ls -l /dev/hailo*`. This asks if /dev/hailo0 exists, who owns it, and what permission level it has. We should see something like `crw-rw---- 1 root hailo ... /dev/hailo0`.

6. `lspci -tv`. This shows where the hailo chip sits in the PCIe tree.

7. `modinfo hailo_pci`

8. `ls /lib/firmware/hailo/`

<br>

### Step 3: Install the Hailo Runtime

1. `sudo apt install ./hailort_4.23.0_arm64.deb`
2. `sudo reboot`

<br>

### Step 4: Check Hailo Runtime installation

1. `which hailortcli`. We should see `/usr/bin/hailortcli`.
   
2. `hailortcli scan`. This asks 'using the hailo runtime and driver, can i find any connected hailo devices?'. We should see `Hailo Devices: [-] Device: 0001:01:00.0`. This shows that the kernel driver (hailo_pci) is loaded and functioning, and HailoRT can communicate with it.

3. `systemctl status hailort.service`.  This command asks systemd 'what is the current state of the hailoRT background service?'. Basically, it is asking if the background manager for the hailo 8 processor is installed, running, if it is started, and what process it is running. We should see that it is loaded (`/lib/systemd/system/hailort.service; enabled`), and that it is active and running.

4. `hailortcli fw-control identify`. This command queries the hailo device firmware and confirms it is detected. Basically it is asking 'can i talk to the firmware running on the hailo chip, and what exactly is it?'. When we run it we see `No device found`.
   
<br>

### Step 5: Install the HailoRT Python bindings

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

<br>

### Step 6: Troubleshooting

We ran into an issue intially after installation. First run `dmesg | grep hailo`. You will get the following:

```
[ 3.055171] hailo_pci: loading out-of-tree module taints kernel.
[ 3.060775] hailo: Init module. driver version 4.23.0
[ 3.060867] hailo 0001:01:00.0: Probing on: 1e60:2864...
[ 3.060870] hailo 0001:01:00.0: Probing: Allocate memory for device extension, 9072
[ 3.060884] hailo 0001:01:00.0: enabling device (0000 -> 0002)
[ 3.060889] hailo 0001:01:00.0: Probing: Device enabled
[ 3.060904] hailo 0001:01:00.0: Probing: mapped bar 0 - 0000000054225f0c 16384
[ 3.060908] hailo 0001:01:00.0: Probing: mapped bar 2 - 000000006f5a0a16 4096
[ 3.060911] hailo 0001:01:00.0: Probing: mapped bar 4 - 0000000025430d0e 16384
[ 3.060914] hailo 0001:01:00.0: Probing: Setting max_desc_page_size to 16384, (page_size=16384) [ 3.060922] hailo 0001:01:00.0: Probing: Enabled 64 bit dma
[ 3.060924] hailo 0001:01:00.0: Probing: Using userspace allocated vdma buffers
[ 3.060926] hailo 0001:01:00.0: Disabling ASPM L0s
[ 3.060929] hailo 0001:01:00.0: Successfully disabled ASPM L0s
[ 3.061009] hailo 0001:01:00.0: Writing file hailo/hailo8_fw.bin
[ 3.104052] hailo 0001:01:00.0: File hailo/hailo8_fw.bin written successfully
[ 3.104059] hailo 0001:01:00.0: Writing file hailo/hailo8_board_cfg.bin
[ 3.104083] hailo 0001:01:00.0: File hailo/hailo8_board_cfg.bin written successfully [ 3.104085] hailo 0001:01:00.0: Writing file hailo/hailo8_fw_cfg.bin
[ 3.104092] hailo 0001:01:00.0: File hailo/hailo8_fw_cfg.bin written successfully
[ 3.219863] hailo 0001:01:00.0: NNC Firmware loaded successfully
[ 3.219870] hailo 0001:01:00.0: FW loaded, took 158 ms
[ 3.231991] hailo 0001:01:00.0: Probing: Added board 1e60-2864, /dev/hailo0
```

Now we have this script:

```
from picamera2 import Picamera2
import cv2
import hailo_platform as hailo
import numpy as np

hef = hailo.HEF("yolov8s.hef")
with hailo.VDevice() as target:

    configure_params = hailo.ConfigureParams.create_from_hef(hef, interface=hailo.HailoStreamInterface.PCIe)
    network_group = target.configure(hef, configure_params)[0]
    network_group_params = network_group.create_params() input_vstream_info = hef.get_input_vstream_infos()[0]
    output_vstream_infos = hef.get_output_vstream_infos()
    input_vstreams_params = hailo.InputVStreamParams.make_from_network_group(network_group, quantized=False, format_type=hailo.FormatType.FLOAT32)
    output_vstreams_params = hailo.OutputVStreamParams.make_from_network_group(network_group, quantized=False, format_type=hailo.FormatType.FLOAT32)

    with network_group.activate(network_group_params):
        with hailo.InferVStreams(network_group, input_vstreams_params, output_vstreams_params) as infer_pipeline:

            picam2 = Picamera2()
            config = picam2.create_preview_configuration(main = {"size": (640, 640), "format": "RGB888"}, controls = {"FrameRate": 1} )
            picam2.configure(config)
            picam2.start()

            while True:

                frame = picam2.capture_array()
                frame_resized = cv2.resize(frame, (640, 640))
                frame_resized_uint8 = np.expand_dims(frame_resized.astype(np.float32), axis=0)
                input_data = {input_vstream_info.name: frame_resized_uint8}
    
                results = infer_pipeline.infer(input_data)
                print("Raw output keys: ", results.keys())
    
                cv2.imshow("Camera", frame)
                if cv2.waitKey(1) & 0xFF == ord('q'):
                    break

            cv2.destroyAllWindows()
            picam2.stop()
```

When we run it we get the following

```
[HailoRT] [error] CHECK failed - max_desc_page_size given 16384 is bigger than hw max desc page size 4096
[HailoRT] [error] CHECK_SUCCESS failed with status=HAILO_INTERNAL_FAILURE(8)
[HailoRT] [error] CHECK_SUCCESS failed with status=HAILO_INTERNAL_FAILURE(8)
[HailoRT] [error] CHECK_SUCCESS failed with status=HAILO_INTERNAL_FAILURE(8)
libhailort failed with error: 8 (HAILO_INTERNAL_FAILURE)
```

The primary error here is a driver/hardware mismatch: `max_desc_page_size given 16384 is bigger than hw max desc page size 4096`

Think of the hailo chip as a mail sorting machine: The driver tells it 'i will send you big envelopees (16kb pages)'. But the hardware replies: 'I only accept small envelopes (4kb max)'. So the system refuses to start. That is what is happening here:

max_desc_page_size controls DMA descriptor buffer sizes. Our driver defaulted to 16384 bytes,  but our hailo8 hardware supports 4096 bytes max. So, HailoRT aborts during configure() with HAILO_INTERNAL_FAILURE (8). 

Next, edit the hailo configuration file and add this line `options hailo_pci force_desc_page_size=4096`

```
rza@rp5-pios:~ $ sudo nano /etc/modprobe.d/hailo.conf
options hailo_pci force_desc_page_size=4096
```

finally, make sure the new configuration is applied every boot, and reboot

```
(hailo_env) rza@rp5-pios:~/Desktop/streamer $ sudo update-initramfs -u
update-initramfs: Generating /boot/initrd.img-6.12.75+rpt-rpi-v8 '/boot/initrd.img-6.12.75+rpt-rpi-v8' -> '/boot/firmware/initramfs8'
update-initramfs: Generating /boot/initrd.img-6.12.75+rpt-rpi-2712 '/boot/initrd.img-6.12.75+rpt-rpi-2712' -> '/boot/firmware/initramfs_2712'

rza@rp5-pios:~ $ sudo reboot
```

now, run `dmesg | grep hailo`, you should see `Force setting max_desc_page_size to 4096`:

```
[ 3.160062] hailo_pci: loading out-of-tree module taints kernel.
[ 3.167468] hailo: Init module. driver version 4.23.0
[ 3.168684] hailo 0001:01:00.0: Probing on: 1e60:2864...
[ 3.168688] hailo 0001:01:00.0: Probing: Allocate memory for device extension, 9072
[ 3.168701] hailo 0001:01:00.0: enabling device (0000 -> 0002) [ 3.168705] hailo 0001:01:00.0: Probing: Device enabled
[ 3.168719] hailo 0001:01:00.0: Probing: mapped bar 0 - 00000000ba755f05 16384
[ 3.168723] hailo 0001:01:00.0: Probing: mapped bar 2 - 00000000e1576ddb 4096
[ 3.168725] hailo 0001:01:00.0: Probing: mapped bar 4 - 00000000b05f013b 16384
[ 3.168728] hailo 0001:01:00.0: Probing: Force setting max_desc_page_size to 4096 (recommended value is 16384)
[ 3.168735] hailo 0001:01:00.0: Probing: Enabled 64 bit dma
[ 3.168738] hailo 0001:01:00.0: Probing: Using userspace allocated vdma buffers
[ 3.168740] hailo 0001:01:00.0: Disabling ASPM L0s
[ 3.168743] hailo 0001:01:00.0: Successfully disabled ASPM L0s
[ 3.168817] hailo 0001:01:00.0: Writing file hailo/hailo8_fw.bin
[ 3.213421] hailo 0001:01:00.0: File hailo/hailo8_fw.bin written successfully [ 3.213429] hailo 0001:01:00.0: Writing file hailo/hailo8_board_cfg.bin
[ 3.213453] hailo 0001:01:00.0: File hailo/hailo8_board_cfg.bin written successfully [ 3.213455] hailo 0001:01:00.0: Writing file hailo/hailo8_fw_cfg.bin
[ 3.213463] hailo 0001:01:00.0: File hailo/hailo8_fw_cfg.bin written successfully
[ 3.329233] hailo 0001:01:00.0: NNC Firmware loaded successfully
[ 3.329240] hailo 0001:01:00.0: FW loaded, took 160 ms
[ 3.341398] hailo 0001:01:00.0: Probing: Added board 1e60-2864, /dev/hailo0
```

Great, now try running again...

```
(hailo_env) rza@rp5-pios:~/Desktop/streamer $ python camtest.py
[HailoRT] [error] CHECK failed - The given output format type UINT8 is not supported, should be HAILO_FORMAT_TYPE_FLOAT32
[HailoRT] [error] CHECK_SUCCESS failed with status=HAILO_INVALID_ARGUMENT(2)
[HailoRT] [error] CHECK_SUCCESS failed with status=HAILO_INVALID_ARGUMENT(2)
[HailoRT] [error] CHECK_SUCCESS failed with status=HAILO_INVALID_ARGUMENT(2)
```

Make the final changes to fix tehre reminaing error 

```
from picamera2 import Picamera2
import cv2
import hailo_platform as hailo
import numpy as np

hef = hailo.HEF("yolov8s.hef")
with hailo.VDevice() as target:

    configure_params = hailo.ConfigureParams.create_from_hef(hef, interface=hailo.HailoStreamInterface.PCIe)
    network_group = target.configure(hef, configure_params)[0]
    network_group_params = network_group.create_params() input_vstream_info = hef.get_input_vstream_infos()[0]
    output_vstream_infos = hef.get_output_vstream_infos()
    input_vstreams_params = hailo.InputVStreamParams.make_from_network_group(
        network_group,
        quantized=False,
        format_type=hailo.FormatType.FLOAT32
    )
    output_vstreams_params = hailo.OutputVStreamParams.make_from_network_group(
        network_group,
        quantized=False,
        format_type=hailo.FormatType.FLOAT32
    )
    with network_group.activate(network_group_params):
        with hailo.InferVStreams(network_group, input_vstreams_params, output_vstreams_params) as infer_pipeline:

            picam2 = Picamera2()
            config = picam2.create_preview_configuration(main = {"size": (640, 640), "format": "RGB888"}, controls = {"FrameRate": 1} )
            picam2.configure(config)
            picam2.start()

            while True:

                frame = picam2.capture_array()
                frame = cv2.resize(frame, (640, 640))
                frame = frame.astype(np.float32)
                frame = frame / 255.0
                input_data = {input_vstream_info.name: np.expand_dims(frame, axis=0)}
                    
                results = infer_pipeline.infer(input_data)
                print("Raw output keys: ", results.keys())
    
                cv2.imshow("Camera", frame)
                if cv2.waitKey(1) & 0xFF == ord('q'):
                    break

            cv2.destroyAllWindows()
            picam2.stop()
```

now when we run it we get no immediate error! we can see video!




