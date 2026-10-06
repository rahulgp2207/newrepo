Vulkan GPU Acceleration Guide: llama.cpp on Intel MacBook Pro (AMD Radeon Pro 5300M)
This runbook covers how to launch, benchmark, and serve quantized local models directly on the AMD Radeon Pro 5300M (4 GB VRAM) using Vulkan (MoltenVK) on macOS x86_64.
1. Hardware & Environment Baseline
Host: 16-inch MacBook Pro (2019, Intel Core i7, 16 GB RAM)
Discrete GPU: AMD Radeon Pro 5300M (4080 MiB VRAM)
Target Device Identifier: Vulkan0
Vulkan ICD Manifest: /usr/local/etc/vulkan/icd.d/MoltenVK_icd.json
llama.cpp Path: ~/tmp/llama.cpp
Python Project Path: ~/projects/qwen-local
2. Rebuilding the Vulkan Binary (If Updated or Cleaned)
If you pull upstream changes or clean the build directory, compile with the official Vulkan loader:
Bash
cd ~/tmp/llama.cpp

cmake -B build-vulkan \
  -DGGML_METAL=OFF \
  -DGGML_VULKAN=ON \
  -DCMAKE_PREFIX_PATH="/usr/local" \
  -DVulkan_INCLUDE_DIR=/usr/local/include \
  -DVulkan_LIBRARY=/usr/local/lib/libvulkan.dylib \
  -DVulkan_GLSLC_EXECUTABLE=/usr/local/bin/glslc

cmake --build build-vulkan --config Release -j6
3. Verify Hardware & Device Visibility
Before running models, confirm that the MoltenVK layer recognizes your discrete GPU:
Bash
cd ~/tmp/llama.cpp
./build-vulkan/bin/llama-cli --list-devices
Expected output:
Plaintext
Available devices:
  Vulkan0: AMD Radeon Pro 5300M (4080 MiB, 4080 MiB free)
  Vulkan1: Intel(R) UHD Graphics 630 (16384 MiB, 1527 MiB free)
  BLAS: Accelerate (0 MiB, 0 MiB free)
Crucial Syntax Note: Modern llama.cpp requires the string device identifier (--device Vulkan0). Passing an integer index like --device 0 will fail.

4. Benchmark Throughput
To test raw tokens/sec processing before serving:
Bash
cd ~/tmp/llama.cpp

# Benchmark Qwen3-4B (~37 t/s expected)
./build-vulkan/bin/llama-bench \
  -hf Qwen/Qwen3-4B-GGUF:Q4_K_M \
  --device Vulkan0 \
  -ngl 99 \
  -p 64 \
  -n 64

# Benchmark Qwen2.5-3B (~47 t/s expected)
./build-vulkan/bin/llama-bench \
  -hf Qwen/Qwen2.5-3B-Instruct-GGUF:Q4_K_M \
  --device Vulkan0 \
  -ngl 99 \
  -p 64 \
  -n 64
5. Running the Background Server (llama-server)
Kill any existing instances and run the server detached with nohup:
Option A: Best Overall (Qwen3-4B)
Bash
pkill -f llama-server

cd ~/tmp/llama.cpp
nohup ./build-vulkan/bin/llama-server \
  -hf Qwen/Qwen3-4B-GGUF:Q4_K_M \
  --device Vulkan0 \
  -ngl 99 \
  --jinja \
  --reasoning off \
  --reasoning-budget 0 \
  -c 2048 \
  -t 6 \
  --host 127.0.0.1 \
  --port 8080 > ~/tmp/llama-server.log 2>&1 &
Option B: Maximum Speed (Qwen2.5-3B)
Bash
pkill -f llama-server

cd ~/tmp/llama.cpp
nohup ./build-vulkan/bin/llama-server \
  -hf Qwen/Qwen2.5-3B-Instruct-GGUF:Q4_K_M \
  --device Vulkan0 \
  -ngl 99 \
  --jinja \
  -c 2048 \
  -t 6 \
  --host 127.0.0.1 \
  --port 8080 > ~/tmp/llama-server.log 2>&1 &
Verify Server Health
Check that the server allocated to Vulkan0 without CPU fallback:
Bash
tail -n 25 ~/tmp/llama-server.log
Look for:
ggml_vulkan: Found 2 Vulkan devices
0 = AMD Radeon Pro 5300M (MoltenVK)
HTTP server listening on [http://127.0.0.1:8080](http://127.0.0.1:8080)
6. Python Client Execution
Navigate to the client environment:
Bash
cd ~/projects/qwen-local
source venv/bin/activate
Confirm .env configuration:
Bash
cat << 'EOF' > .env
OPENAI_BASE_URL=http://127.0.0.1:8080/v1
OPENAI_API_KEY=not-needed
MODEL_NAME=Qwen/Qwen3-4B-GGUF:Q4_K_M
EOF
Query the server:
Bash
python app.py "Explain memory mapping (mmap) vs standard read/write syscalls."
7. Troubleshooting & VRAM Boundaries
Do Not Exceed 4B at Q4_K_M: Models larger than ~4.2B parameters (e.g., 7B/8B/27B) will exceed the 4080 MiB VRAM boundary and force CPU execution or crash with memory allocation errors.
Keep Context Window at ≤ 4096: Large context windows (-c 16384 or -c 32768) expand the KV cache past 1.5 GB, causing out-of-memory errors on 4 GB VRAM cards. Keep -c 2048 or -c 4096.
Portability Warning: If you see VK_KHR_portability_enumeration not found, export the driver manifests manually:
Bash
export VK_ICD_FILENAMES=/usr/local/etc/vulkan/icd.d/MoltenVK_icd.json
export VK_DRIVER_FILES=/usr/local/etc/vulkan/icd.d/MoltenVK_icd.json
export VK_ENABLE_PORTABILITY_ENUMERATION=1
