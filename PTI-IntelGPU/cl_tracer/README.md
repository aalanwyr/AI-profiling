# OpenCL(TM) Tracer
## Source Code
**This repo is an archive for cl_tracer, you can refer [cl_tracer](https://github.com/intel/pti-gpu/tree/master/tools/cl_tracer) to get source code or use the binary of this repo directly.**

## Quick BKM in Linux
```
cp cl_tracer libclt_tracer.so <your dir>
./cl_tracer [options] <application> <args>
```
## Overview
This tool is an analogue of [Intercept Layer for OpenCL(TM) Applications](https://github.com/intel/opencl-intercept-layer) designed based on internal [tracing mechanism](../../chapters/runtime_api_tracing/OpenCL.md) implemented in Intel runtimes for OpenCL(TM).

The following capabilities are available currently:
```
Usage: ./cl_tracer[.exe] [options] <application> <args>
Options:
--call-logging [-c]            Trace host API calls
--host-timing  [-h]            Report host API execution time
--device-timing [-d]           Report kernels execution time
--kernel-submission [-s]       Report queued, submit and execute intervals for kernels
--device-timeline [-t]         Trace device activities
--chrome-call-logging          Dump host API calls to JSON file
--chrome-device-timeline       Dump device activities to JSON file per command queue
--chrome-kernel-timeline       Dump device activities to JSON file per kernel name
--chrome-device-stages         Dump device activities by stages to JSON file
--verbose [-v]                 Enable verbose mode to show more kernel information
--demangle                     Demangle DPC++ kernel names
--tid                          Print thread ID into host API trace
--pid                          Print process ID into host API and device activity trace
--output [-o] <filename>       Print console logs into the file
--conditional-collection       Enable conditional collection mode
--version                      Print version
```

**Call Logging** mode allows to grab full host API trace, e.g.:
```
...
>>>> [271632470] clCreateBuffer: context = 0x5591dba3f860 flags = 4 size = 4194304 hostPtr = 0 errcodeRet = 0x7ffd334b2f04
<<<< [271640078] clCreateBuffer [7608 ns] result = 0x5591dbaa5760 -> CL_SUCCESS (0)
>>>> [272171119] clEnqueueWriteBuffer: commandQueue = 0x5591dbf4be70 buffer = 0x5591dbaa5760 blockingWrite = 1 offset = 0 cb = 4194304 ptr = 0x5591dc92af90 numEventsInWaitList = 0 eventWaitList = 0 event = 0
<<<< [272698660] clEnqueueWriteBuffer [527541 ns] -> CL_SUCCESS (0)
>>>> [272716922] clSetKernelArg: kernel = 0x5591dc500c60 argIndex = 0 argSize = 8 argValue = 0x7ffd334b2f10
<<<< [272724034] clSetKernelArg [7112 ns] -> CL_SUCCESS (0)
>>>> [272729938] clSetKernelArg: kernel = 0x5591dc500c60 argIndex = 1 argSize = 8 argValue = 0x7ffd334b2f18
<<<< [272733712] clSetKernelArg [3774 ns] -> CL_SUCCESS (0)
...
```
**Chrome Call Logging** mode dumps API calls to JSON format that can be opened in [chrome://tracing](https://www.chromium.org/developers/how-tos/trace-event-profiling-tool) browser tool.

**Host Timing** mode collects duration for each API call and provides the summary for the whole application:
```
=== API Timing Results: ===

          Total Execution Time (ns):    366500174
Total API Time for CPU backend (ns):        16851
Total API Time for GPU backend (ns):    357744252

== CPU Backend: ==

      Function,       Calls,           Time (ns),  Time (%),        Average (ns),            Min (ns),            Max (ns)
clGetDeviceIDs,           1,               16851,    100.00,               16851,               16851,               16851

== GPU Backend: ==

                          Function,       Calls,           Time (ns),  Time (%),        Average (ns),            Min (ns),            Max (ns)
                          clFinish,           4,           174933263,     48.90,            43733315,            42966659,            44067629
                    clBuildProgram,           1,           172466699,     48.21,           172466699,           172466699,           172466699
              clEnqueueWriteBuffer,           8,             3788816,      1.06,              473602,              367912,              593802
            clEnqueueNDRangeKernel,           4,             3238743,      0.91,              809685,              208889,             2562605
...
```
**Device Timing** mode collects duration for each kernel on the device and provides the summary for the whole application:
```
=== Device Timing Results: ===

             Total Execution Time (ns):            375937416
Total Device Time for CPU backend (ns):                    0
Total Device Time for GPU backend (ns):            176740729

== GPU Backend: ==

              Kernel,       Calls,           Time (ns),  Time (%),        Average (ns),            Min (ns),            Max (ns)
                GEMM,           4,           171231415,     96.88,            42807853,            42778416,            42843666
clEnqueueWriteBuffer,           8,             3256330,      1.84,              407041,              287916,              548416
 clEnqueueReadBuffer,           4,             2252984,      1.27,              563246,              558973,              567022
```
**Kernel Submission** mode collects queued, submit and execute intervals for kernels and memory transfers:
```
=== Kernel Submission Results: ===

             Total Execution Time (ns):            424658735
Total Device Time for CPU backend (ns):                    0
Total Device Time for GPU backend (ns):            171493149

== GPU Backend: ==

              Kernel,       Calls,         Queued (ns),  Queued (%),         Submit (ns),  Submit (%),        Execute (ns), Execute (%),
                GEMM,           4,               61975,       20.61,             2076275,       39.43,           166183665,       96.90,
clEnqueueWriteBuffer,           8,              231249,       76.92,             3181001,       60.41,             3032164,        1.77,
 clEnqueueReadBuffer,           4,                7422,        2.47,                8694,        0.17,             2277320,        1.33,
```
**Verbose** mode provides additional information per kernel (SIMD width, global and local sizes) and per transfer (bytes transferred). This option should be used in addition to others, e.g. for **Device Timing** mode one can get:
```
=== Device Timing Results: ===

             Total Execution Time (ns):            377460265
Total Device Time for CPU backend (ns):                    0
Total Device Time for GPU backend (ns):            177372573

== GPU Backend: ==

                                  Kernel,       Calls,           Time (ns),  Time (%),        Average (ns),            Min (ns),            Max (ns)
GEMM[SIMD32, {1024, 1024, 1}, {0, 0, 0}],           4,           171844666,     96.88,            42961166,            42911500,            43056666
     clEnqueueWriteBuffer[4194304 bytes],           8,             3166997,      1.79,              395874,              308666,              551833
      clEnqueueReadBuffer[4194304 bytes],           4,             2360910,      1.33,              590227,              568817,              617947
```

**Device Timeline** mode dumps four timestamps for each device activity - *queued* to the host command queue, *submit* to device queue, *start* and *end* on the device (all the timestamps are in CPU nanoseconds):
```
...
Device Timeline (queue: 0x55a9c7e51e70): clEnqueueWriteBuffer [ns] = 317341082 (queued) 317355010 (submit) 317452332 (start) 317980165 (end)
Device Timeline (queue: 0x55a9c7e51e70): clEnqueueWriteBuffer [ns] = 317789774 (queued) 317814558 (submit) 318160607 (start) 318492690 (end)
Device Timeline (queue: 0x55a9c7e51e70): GEMM [ns] = 318185764 (queued) 318200629 (submit) 318550014 (start) 361260930 (end)
Device Timeline (queue: 0x55a9c7e51e70): clEnqueueReadBuffer [ns] = 361479600 (queued) 361481387 (submit) 361482574 (start) 362155593 (end)
...
```
**Chrome Device Timeline** mode dumps timestamps for device activities per command queue to JSON format that can be opened in [chrome://tracing](https://www.chromium.org/developers/how-tos/trace-event-profiling-tool) browser tool. Can't be used with **Chrome Kernel Timeline** and **Chrome Device Stages**.

**Chrome Kernel Timeline** mode dumps timestamps for device activities per kernel name to JSON format that can be opened in [chrome://tracing](https://www.chromium.org/developers/how-tos/trace-event-profiling-tool) browser tool. Can't be used with **Chrome Device Timeline**.

**Chrome Device Stages** mode provides alternative view for device queue where each kernel invocation is divided into stages: "queued", "sumbitted" and "execution". Can't be used with **Chrome Device Timeline**.

**Conditional Collection** mode allows one to enable data collection for any target interval (by default collection will be disabled) using environment variable `PTI_ENABLE_COLLECTION`, e.g.:
```cpp
// Collection disabled
setenv("PTI_ENABLE_COLLECTION", "1", 1);
// Collection enabled
unsetenv("PTI_ENABLE_COLLECTION");
// Collection disabled
```
All the API calls and kernels, which submission happens while collection disabled interval, will be omitted from final results.

## Supported OS
- Linux
- Windows

## Prerequisites
- [CMake](https://cmake.org/) (version 3.12 and above)
- [Git](https://git-scm.com/) (version 1.8 and above)
- [Python](https://www.python.org/) (version 2.7 and above)
- [OpenCL(TM) ICD Loader](https://github.com/KhronosGroup/OpenCL-ICD-Loader)
- [Intel(R) Graphics Compute Runtime for oneAPI Level Zero and OpenCL(TM) Driver](https://github.com/intel/compute-runtime) to run on GPU
- [Intel(R) Xeon(R) Processor / Intel(R) Core(TM) Processor (CPU) Runtimes](https://software.intel.com/en-us/articles/opencl-drivers#cpu-section) to run on CPU

## Build and Run
### Linux
Run the following commands to build the sample:
```sh
git clone https://github.com/intel/pti-gpu.git pti
cd <pti>/tools/cl_tracer
mkdir build
cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make
```
Use this command line to run the tool:
```sh
./cl_tracer [options] <target_application>
```
One may use [cl_gemm](../../samples/cl_gemm) or [dpc_gemm](../../samples/dpc_gemm) as target application, e.g.:
```sh
./cl_tracer -c -h ../../../samples/cl_gemm/build/cl_gemm
./cl_tracer -c -h ../../../samples/dpc_gemm/build/dpc_gemm cpu
```
### Windows
Use Microsoft* Visual Studio x64 command prompt to run the following commands and build the sample:
```sh
cd <pti>\tools\cl_tracer
mkdir build
cd build
cmake -G "NMake Makefiles" -DCMAKE_BUILD_TYPE=Release -DCMAKE_LIBRARY_PATH=<opencl_icd_lib_path> ..
nmake
```
Use this command line to run the tool:
```sh
cl_tracer.exe [options] <target_application>
```
One may use [cl_gemm](../../samples/cl_gemm) or [dpc_gemm](../../samples/dpc_gemm) as target application, e.g.:
```sh
cl_tracer.exe -c -h ..\..\..\samples\cl_gemm\build\cl_gemm.exe
cl_tracer.exe -c -h ..\..\..\samples\dpc_gemm\build\Release\dpc_gemm.exe cpu
```