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

Run the following to get the benchmark with llama bench:

<code>.\llama-bench.exe -m "Qwen3-4B-Q4_K_M.gguf" -ngl 99</code>

For me this returns:
<code>
| qwen3 4B Q4_K - Medium         |   2.32 GiB |     4.02 B | SYCL       |  99 |         pp512 |        260.52 ± 2.02 |
| qwen3 4B Q4_K - Medium         |   2.32 GiB |     4.02 B | SYCL       |  99 |         tg128 |         12.71 ± 0.55 |
</code>
What that means:

<code>pp512 = 260.52 t/s</code>

This is prompt processing speed for a 512-token prompt.

This is the big number for RAG, long prompts, tool outputs, document-heavy workflows etc and it’s pretty good.

<code>tg128 = 12.71 t/s</code>

This is generation speed for 128 output tokens.

This is the number people usually think of as “chat speed”.

Now we will benchmark this against a local running Ollama instance using the same model but which is not taking advantage of the ARC GPU.

From the terminal lets load the same model in Ollama:

<code>ollama run qwen3:4b --verbose</code>

and the we will use a simple prompt: "Write 300 words about Intel Arc B580 and local AI inference". At the end, Ollama should print verbose timing info.

The output of this is:
<code>
total duration:       2m44.2998187s
load duration:        295.4256ms
prompt eval count:    27 token(s)
prompt eval duration: 1.8975681s
prompt eval rate:     14.23 tokens/s
eval count:           992 token(s)
eval duration:        2m40.1077145s
eval rate:            6.20 tokens/s
</code>
This gives us clean set of comarative numbers:

Prompt ingestion	14.23 tokens/sec
Generation	6.20 tokens/sec

For prompt processing, when taking advantage of the Intel GPU:
*260.52 / 14.23 = ~18.3x faster*

For token generation when taking advantage of the Intel GPU:
*12.71 / 6.20 = ~2.05x faster*

<img width="897" height="665" alt="image" src="https://github.com/user-attachments/assets/97f36ae9-53f1-4425-9f28-4ebd6a36db36" />

Lets look at an example RAG query to understand what this can mean in real usage.

If a system injects:

4,000 tokens of context then:

Ollama = 4000 / 14.23 ≈ 281 seconds
Arc GPU = 4000 / 260 ≈ 15 seconds

This is where GPU acceleration matters most.









