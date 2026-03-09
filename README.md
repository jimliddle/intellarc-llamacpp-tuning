# intellarc-llamacpp-tuning

The aim of this is to provide a short tutorial on how to setup llama-cpp to be able to take advantage of the Intel ARC GPU.

Stock Ollama releases haven’t consistently shipped Intel Arc GPU support enabled by default, so most “Arc optimizations” are via Intel’s IPEX-LLM–accelerated Ollama builds (SYCL/oneAPI/Level Zero), typically delivered as a Docker image or a portable Windows zip.

I'm going to focus on the portable Windows Zip for this tutorial. Intel/IPEX-LLM provides a portable zip flow where you unzip, run a batch file to start the accelerated Ollama service.

First you need to download the file that intel makes available. You can do that from: https://github.com/ipex-llm/ipex-llm/releases/tag/v2.3.0-nightly

At the time of writing this guide the file that you need is called llama-cpp-ipex-llm-2.3.0b20250612-win.zip.

You will also need a model to test this with. I used Qwen3-4B-Q4_K_M.gguf which you can get from HuggingFace.

You want something that:

- Fits comfortably in Arc VRAM

- Offloads fully to GPU

- Gives realistic tokens/sec

- Doesn’t bottleneck on memory bandwidth

Before we conduct any tests, first lets just confirm what type of ARC GPU we have. Run this command in the terminal:

<code>Get-CimInstance Win32_VideoController | Format-List Name,PNPDeviceID,AdapterRAM,DriverVersion</code>

You should receive something back like:

Name          : Intel(R) Graphics
PNPDeviceID   : PCI\VEN_8086&DEV_7D45&SUBSYS_3D3617AA&REV_08\3&11583659&1&10
AdapterRAM    : 2147479552
DriverVersion : 32.0.101.6881

What this tells us:

- Device ID: VEN_8086&DEV_7D45
- Driver: 32.0.101.6881

The important part is the DEV_7D45, whicb corresponds to the Intel Arc B580. So here my laptop is running on an Intel Arc B580 (Battlemage) which has ~17.7GB “global memory” and is consistent with a 16GB-class card (plus reporting differences).

For an additional confirmation, enter the following in the terminal within the folder where you downloaded the llama intel optimized files and the huggingface image:

<code>.\llama-cli.exe -m "Qwen3-4B-Q4_K_M.gguf" -ngl 99 -p "Write 300 words about Intel Arc B580." -no-cnv</code>

Open Task Manager → Performance → GPU and watch  the dedicated GPU memory rise by a few GB and the compute / 3D activity increase.

<img width="719" height="718" alt="Task Manager Screenshot" src="https://github.com/user-attachments/assets/b3fa0449-176d-4af0-a929-7b32fcc8c674" />

This shows the Arc GPU is actively running inference. The GPU compute engines are fully busy. For LLM inference, this is exactly what we want to see.

Now lets just benchmark this setup.








