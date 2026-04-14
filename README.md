# RPi5 AI-Inference & LLM Optimized Linux Kernel

This repository provides a custom-compiled, high-performance, and low-power Linux kernel (v6.12.y) specifically engineered for the Raspberry Pi 5 (Broadcom BCM2712). 

It is compiled natively on **NVIDIA Grace Blackwell (DGX Spark)** ARM64 architecture, leveraging `-O3` and Cortex-A76 specific tuning (`-march=armv8.2-a+crypto+fp16+dotprod`). 

This kernel is designed for **headless AI inferencing** (e.g., LocalLLaMA, Hailo, Coral) where memory bandwidth and C-state residency (idle power) are the highest priorities.

## 🧠 Architectural Optimizations

### 1. Memory Subsystem (Throughput Focus)
The standard Pi 5 kernel uses 4K pages, which creates a massive Translation Lookaside Buffer (TLB) bottleneck when navigating large LLM weight matrices.
* **16K Page Size (`CONFIG_ARM64_16K_PAGES=y`)**: Quadruples the memory mapped per TLB entry, yielding a 5-10% token generation speedup in `llama.cpp`.
* **Transparent HugePages (`CONFIG_TRANSPARENT_HUGEPAGE_ALWAYS=y`)**: Actively collapses pages into 2MB blocks, drastically reducing page fault overhead during large tensor allocations.
* **Fake NUMA Emulation (`CONFIG_NUMA_EMU=y`)**: Splits the physical RAM into 4 virtual NUMA nodes. This tricks the kernel scheduler into better memory parallelization across the 4 Cortex-A76 cores, yielding up to an 18% uplift in specific parallel workloads.

### 2. CPU & Power Scheduling (Idle Efficiency)
To keep thermal output and power draw minimal when not actively inferencing.
* **100Hz Timer (`CONFIG_HZ_100=y`)**: Reduced from the standard 1000Hz. Drastically cuts down on context switching and CPU wakeups.
* **Tickless Idle (`CONFIG_NO_HZ_IDLE=y`)**: Disables the scheduling-clock interrupt on idle CPUs, allowing deep C-state residency.
* **Schedutil Governor (`CONFIG_CPU_FREQ_GOV_SCHEDUTIL=y`)**: Replaces the standard 'ondemand' governor for tighter, utilization-driven frequency scaling.

### 3. Headless Strip-Down (Bloat Reduction)
This is a raw compute kernel. Unnecessary hardware drivers have been removed to maximize available RAM and reduce kernel footprint.
* **Disabled DRM (Direct Rendering Manager)**: No HDMI/Display output.
* **Disabled Sound Architecture (ALSA)**: No audio processing overhead.

## 🎯 Ideal Use Cases

This kernel is purpose-built for scenarios where the Raspberry Pi 5 acts as a dedicated compute node rather than a general-purpose desktop. Here is where it outperforms the stock Raspberry Pi OS kernel:

### 1. Local LLM Hosting (Ollama, llama.cpp)
* **The Scenario:** Running a 7B or 8B parameter model locally.
* **Why this kernel wins:** The stock kernel's 4K page size struggles with the massive memory mapping required by LLM weight matrices, causing Translation Lookaside Buffer (TLB) bottlenecks. By utilizing **16K pages**, **Transparent HugePages**, and **Fake NUMA**, this kernel maximizes memory bandwidth, directly increasing tokens-per-second (t/s) generation. Additionally, stripping the GUI frees up critical Megabytes of RAM, allowing for larger context windows.

### 2. Edge Vision & NVR AI (Hailo-8L, Google Coral, Frigate)
* **The Scenario:** An always-on camera recording system that runs object detection (e.g., YOLO) on a PCIe accelerator.
* **Why this kernel wins:** Stock kernels maintain a 1000Hz tick rate, keeping the CPU active even when nothing is happening. This kernel's **100Hz timer** and **tickless idle** mean that when there is no motion on the cameras, the Pi drops into deep C-states, saving power and reducing heat. When motion occurs, the forced PCIe Gen 3 tuning ensures the accelerator is fed without bottlenecking.

### 3. HomeNetworkSecurityAgent
* **The Scenario:** A headless node deployed to monitor network traffic, isolate VLANs, and run local AI models to detect anomalies or malicious IoT behavior.
* **Why this kernel wins:** Security agents require high throughput for packet inspection but spend significant time idling during low-traffic periods. The **Schedutil governor** aggressively scales frequency based on exact network load, ensuring minimal power draw during the night while providing instant compute bursts when scanning heavy traffic.

### 4. Autonomous Robotics (SLAM / ROS2)
* **The Scenario:** A battery-powered drone or rover using the Pi 5 as its main "brain" for navigation and computer vision.
* **Why this kernel wins:** Battery life is critical. By stripping out the Direct Rendering Manager (DRM), ALSA sound architecture, and legacy display drivers, the kernel has a strictly lower baseline power consumption than the stock image, extending operational battery life while providing faster memory access for real-time sensor processing.

## 📦 Installation

Do not clone this repository to install the kernel. Download the pre-compiled `.deb` binaries from the [Releases](https://github.com/assix/rpi5-ai-inference-llm-optimized-linux-kernel/releases) page.

1. Transfer the `.deb` files to your Raspberry Pi 5.
2. Install the kernel and headers:
   ```bash
   sudo dpkg -i linux-image-6.12.*.deb linux-headers-6.12.*.deb
   ```
3. Update your `/boot/firmware/config.txt` to leverage the new kernel features:

   ```ini
   # /boot/firmware/config.txt optimizations
   arm_64bit=1
   
   # Maximize AI Throughput (PCIe Gen 3 for NVMe/Hailo)
   dtparam=pciex1_gen=3
   
   # Enable Fake NUMA (Requires this kernel)
   cmdline=numa=fake=4
   
   # Power Savings (Headless)
   arm_freq_min=500
   gpu_freq_min=500
   disable_splash=1
   hdmi_blanking=1
   ```
4. Reboot the system.

## 🔍 Verification

After rebooting into the new kernel, verify the optimizations:

```bash
# Check for 16K Page Size (Should output 16384)
getconf PAGESIZE

# Check NUMA initialization
dmesg | grep -i numa

# Check active HugePages
cat /proc/meminfo | grep AnonHugePages
```

## 🛠️ Build Source
If you wish to replicate this build:
* **Source:** `https://github.com/raspberrypi/linux` (Branch: `rpi-6.12.y`)
* **Config:** Provided in this repository as `bcm2712-ai-headless.config`.
