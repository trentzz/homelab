# Benchmarking

Benchmarking the homelab helps characterise each node's capabilities before assigning workloads, compare nodes, and catch regressions after config changes.

## CPU

### sysbench

General-purpose benchmark for CPU and memory.

```bash
sudo apt install sysbench   # Debian/Ubuntu
sudo dnf install sysbench   # Fedora
```

**CPU benchmark (prime number calculation):**

```bash
# Single-threaded
sysbench cpu --cpu-max-prime=20000 run

# Multi-threaded (use all cores)
sysbench cpu --cpu-max-prime=20000 --threads=$(nproc) run
```

Key output: `events per second` — higher is better. Compare across nodes to understand relative CPU performance.

### stress-ng

More comprehensive stress testing. Useful for confirming a system is stable under sustained load and for checking thermals.

```bash
sudo apt install stress-ng
```

```bash
# Stress all CPU cores for 60 seconds
stress-ng --cpu $(nproc) --timeout 60s --metrics-brief

# Stress with multiple methods to exercise different CPU paths
stress-ng --cpu $(nproc) --cpu-method all --timeout 60s
```

### 7-zip benchmark

The built-in 7-zip benchmark is a practical CPU test that reflects real compression workloads.

```bash
sudo apt install p7zip-full
7z b
```

Reports MIPS for compression and decompression. Useful for comparing nodes that will run compression-heavy jobs.

## Memory

### sysbench memory

```bash
# Sequential memory throughput
sysbench memory --memory-block-size=1M --memory-total-size=10G run

# Multi-threaded
sysbench memory --memory-block-size=1M --memory-total-size=10G --threads=$(nproc) run
```

Key output: `transferred (MB/s)` — reflects memory bandwidth.

### STREAM

STREAM is the standard benchmark for sustained memory bandwidth. It requires compilation:

```bash
sudo apt install gcc
wget https://www.cs.virginia.edu/stream/FTP/Code/stream.c
gcc -O2 -fopenmp -DSTREAM_ARRAY_SIZE=80000000 -o stream stream.c
OMP_NUM_THREADS=$(nproc) ./stream
```

Reports bandwidth in MB/s for Copy, Scale, Add, and Triad operations. Triad is typically used as the summary number.

## Disk / IO

### fio

`fio` is the most capable disk benchmark tool.

```bash
sudo apt install fio
```

**Sequential read:**

```bash
fio --name=seq-read --rw=read --bs=1M --size=4G --numjobs=1 \
  --iodepth=8 --runtime=30 --time_based --group_reporting \
  --filename=/tmp/fio-test
```

**Sequential write:**

```bash
fio --name=seq-write --rw=write --bs=1M --size=4G --numjobs=1 \
  --iodepth=8 --runtime=30 --time_based --group_reporting \
  --filename=/tmp/fio-test
```

**Random 4K read (IOPS):**

```bash
fio --name=rand-read --rw=randread --bs=4K --size=4G --numjobs=4 \
  --iodepth=32 --runtime=30 --time_based --group_reporting \
  --filename=/tmp/fio-test
```

Key outputs: `bw` (bandwidth in KB/s) and `iops`.

Clean up after testing:

```bash
rm /tmp/fio-test
```

> To benchmark an NFS share, set `--filename` to a path on the mounted share.

### hdparm (quick check)

For a quick read speed estimate on a local disk:

```bash
sudo hdparm -tT /dev/sda
```

This is a rough indicator — use fio for serious benchmarking.

## Network

### iperf3

Measures throughput between two nodes. Run a server on one machine and a client on another.

```bash
sudo apt install iperf3
```

**Server (target node):**

```bash
iperf3 -s
```

**Client (source node):**

```bash
# TCP throughput
iperf3 -c <server-ip> -t 30

# UDP throughput
iperf3 -c <server-ip> -u -b 1G -t 30

# Bidirectional
iperf3 -c <server-ip> --bidir -t 30
```

Key output: `sender` and `receiver` bandwidth in Gbits/sec.

Use this to check:
- LAN throughput between Proxmox server and mini PCs
- NFS throughput (compare with disk benchmark to see if network is the bottleneck)
- Tailscale throughput (will be lower than LAN — expected)

### ping / latency

```bash
ping -c 100 <node-ip>
```

Report: average and max RTT. Useful baseline when debugging slow cross-node job performance.

## GPU

### gpu-burn (Nvidia)

Stress test for CUDA GPUs. Confirms the GPU is stable under sustained load.

```bash
git clone https://github.com/wilicc/gpu-burn
cd gpu-burn
make
./gpu_burn 60   # run for 60 seconds
```

### nvtop / nvidia-smi

Real-time GPU utilisation:

```bash
sudo apt install nvtop
nvtop
```

Or with nvidia-smi:

```bash
watch -n 1 nvidia-smi
```

### CUDA bandwidth test

From the CUDA samples:

```bash
# Debian/Ubuntu with CUDA toolkit installed
/usr/local/cuda/extras/demo_suite/bandwidthTest
```

Reports host-to-device and device-to-host transfer rates.

## Recording Results

Keep a simple record of benchmark results per node so you can compare over time:

```
# node: minipc-1
# date: 2026-03-13
# cpu: sysbench, 8 threads, 20000 primes
events/s: 4231

# memory: sysbench, 1M block, 10G total, 8 threads
MB/s: 18420

# disk: fio sequential read, 1M block
bw: 520 MB/s

# network: iperf3 to proxmox-server
throughput: 940 Mbits/sec
```

A plain markdown file per node in `docs/compute/nodes/` works well for this.
