# TorchRec Inference Library (**Experimental** Release)

## Overview
---
TorchRec Inference is a C++ library that supports **multi-gpu inference**. The Torchrec library is used to shard models written and packaged in Python via [torch.package](https://pytorch.org/docs/stable/package.html) (an alternative to TorchScript). The [torch.deploy](https://pytorch.org/docs/stable/deploy.html) library is used to serve inference from C++ by launching multiple Python interpreters carrying the packaged model, thus subverting the GIL.

Follow the instructions below to package a DLRM model in Python, run a C++ inference server with the model on a GPU and send requests to said server via a python client.

## Example
---

C++ 17 is a requirement.

<br>

### **1. Install Dependencies**

Create a conda environment: `conda create --name inference`

### **Pytorch**

The inference library uses torch deploy which is a library in pytorch that's only accessible if built from source. **Ensure that the pytorch version installed is compatible**
**with your CUDA toolkit and driver**. Run `nvidia-smi` to check driver version. Run `nvcc --version` or `conda list` to check toolkit version.
- Follow this link: https://github.com/pytorch/pytorch/tree/master/torch/csrc/deploy to initially install all the necessary CPython dependencies
- Follow this link: https://github.com/pytorch/pytorch/#from-source, to install pytorch from source. However, once at the installation step, run these commands:
```
export CMAKE_PREFIX_PATH=${CONDA_PREFIX:-"$(dirname $(which conda))/../"}
export USE_DEPLOY=1
python setup.py develop
```

### **TorchRec**
```
pip install torchrec-nightly
```

### **Folly**
The inference library relies on folly for performance optimzations. Follow: https://github.com/facebook/folly#build-notes to install folly from source.
Ensure to provide a path to the `--scratch-path` option (e.g. `~/folly-build/`). Folly will be installed at the location of the scratch path. Folly will also install fmt
(along with other libraries) for you.

### **gRPC**
Install gRPC for both C++ (server) and Python (client).

**C++**

gRPC is used by the server and client to communicate via RPC. It is highly recommended to install gRPC from source. Follow: https://grpc.io/docs/languages/cpp/quickstart/#install-grpce to do so. Note that they strongly encourage local installation ("using an appropriately set `CMAKE_INSTALL_PREFIX`") as opposed to a global installation as it'll be difficult to uninstall.

**Python**

```
pip install grpcio
pip install grpcio-tools # for protoc to generate Python code for from .protobuf files
```


### **fbgemm_gpu**
TorchRec relies on fbgemm_gpu for high-performance CUDA GPU operations. This is needed
for GPU inference. Install fbgemm_gpu here: https://github.com/pytorch/FBGEMM/tree/main/fbgemm_gpu.

If pytorch was built from source with `GLIBCXX_USE_CXX11_ABI=0`, then you can use `pip install fbgemm-gpu-nightly`.

This can be checked via: `torch._C._GLIBCXX_USE_CXX11_ABI`. If the value is `1`, then install fbgemm_gpu from source using the instructions in the link above. This is necessary to ensure the ABIs between dependencies match (else you'll run into `undefined symbol` issues).

In order to use the fbgemm_gpu C++ operators, we need to link to the shared lib: `libfbgemm_gpu_py.so`. After installing fbgemm_gpu, search for the file `fbgemm_gpu_py.so` in your system:

```
sudo find / -type f -name fbgemm_gpu_py.so
```

This file will likely be located in the `site-packages` of your conda environment (e.g. `~anaconda3/envs/inference/lib/python3.8/site-packages/fbgemm_gpu-0.1.0-py3.8-linux-x86_64.egg/fbgemm_gpu/`). Create a copy of this library named: `libfbgemm_gpu_py.so`.

```
cd location_of_fbgemm_gpu_py.so
cp fbgemm_gpu_py.so libfbgemm_gpu_py.so
```

This is necessary to make it easier to link to the fbgemm_gpu C++ library. This step will be removed in the near future.

### **2. Set variables**

Replace these variables with the relevant paths in your system. Check `CMakeLists.txt` and `server.cpp` to see how they're used throughout the build and runtime.

```
# provide the cmake prefix path of pytorch, folly, and fmt.
# fmt is pulled from folly's installation in this example.

export FOLLY_CMAKE_DIR="~/folly-build/installed/folly/lib/cmake/folly"
export FMT_CMAKE_DIR="~/folly-build/installed/fmt-dGmDTkdcPS1pyvm65J7UcKzxzLonWCKaaxWmgYpScUw/lib/cmake/fmt"
export INFERENCE_CMAKE_PREFIX_PATH=
"$(python -c 'import torch.utils; print(torch.utils.cmake_prefix_path)');$FOLLY_CMAKE_DIR;"

#  provide fmt from pytorch for torch deploy
export PYTORCH_FMT_INCLUDE_PATH="~/repos/pytorch/third_party/fmt/include/"
export PYTORCH_LIB_FMT="~/repos/pytorch/build/lib/libfmt.a"

#  provide necessary info to link to torch deploy
export DEPLOY_INTERPRETER_PATH="/repos/pytorch/build/torch/csrc/deploy/libtorch_deployinterpreter.o"
export DEPLOY_SRC_PATH="~/repos/pytorch/torch/csrc/deploy/"

#  provide common.cmake from grpc/examples, makes linking to grpc easier
export GRPC_COMMON_CMAKE_PATH="~/grpc/examples/cpp/cmake/common.cmake"
export GRPC_HEADER_INCLUDE_PATH="~/.local/include/"

# provide libfbgemm_gpu_py.so to enable fbgemm_gpu c++ operators
export FBGEMM_LIB="~/anaconda3/envs/inference/lib/python3.8/site-packages/fbgemm_gpu-0.1.0-py3.8-linux-x86_64.egg/fbgemm_gpu/libfbgemm_gpu_py.so"

# provide path to python packages for torch deploy runtime
export PYTHON_PACKAGES_PATH="~/anaconda3/envs/inference/lib/python3.8/site-packages/"
```

Update `$LD_LIBRARY_PATH` and `$LIBRARY_PATH` to make it easier for linker to locate certain shared libs.

```
# double-conversion, fmt and gflags are pulled from folly's installation in this example
export DOUBLE_CONVERSION_LIB_PATH="~/folly-build/installed/double-conversion-skGL6pOaPHjtDwdXY-agzdwT1gvTXP0bD-7P4gKJD9I/lib"
export FMT_LIB_PATH="~/folly-build/installed/fmt-dGmDTkdcPS1pyvm65J7UcKzxzLonWCKaaxWmgYpScUw/lib"
export GFLAGS_LIB_PATH="~/folly-build/installed/gflags-KheHQBqQ3_iL3yJBFwWe5M5f8Syd-LKAX352cxkhQMc/lib"
export PYTORCH_LIB_PATH="~/repos/pytorch/build/lib/"

export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:$DOUBLE_CONVERSION_LIB_PATH:$FMT_LIB_PATH:$GFLAGS_LIB_PATH:$PYTORCH_LIB_PATH"
export LIBRARY_PATH="$PYTORCH_LIB_PATH"
```

### **3. Package DLRM model**

The `PredictFactoryPackager` class in `model_packager.py` can be used to implement your own packager class. Implement
`set_extern_modules` to specify the dependencies of your predict module that should be accessed from the system and
implement `set_mocked_modules` to specify dependencies that should be mocked (necessary to import but not use). Read
more about extern and mock modules in the `torch.package` documentation: https://pytorch.org/docs/stable/package.html.

`/torchrec/examples/dlrm/inference/dlrm_package.py` provides an example of packaging a module for inference (`/torchrec/examples/dlrm/inference/dlrm_predict.py`).
`DLRMPredictModule` is packaged for inference in the following example.

```
git clone https://github.com/pytorch/torchrec.git

cd ~/torchrec/examples/dlrm/inference/
python dlrm_packager.py --output_path /tmp/model_package.zip
```



### **4. Build inference library and example server**

Generate protobuf C++ and Python code from protobuf

```
cd ~/torchrec/inference/
mkdir -p gen/torchrec/inference

# C++ (server)
protoc -I protos/ --grpc_out=gen/torchrec/inference --plugin=protoc-gen-grpc=~/.local/bin/grpc_cpp_plugin protos/predictor.proto
protoc -I protos/ --cpp_out=gen/torchrec/inference protos/predictor.proto


# Python (client)
python -m grpc_tools.protoc -I protos --python_out=gen/torchrec/inference --grpc_python_out=gen/torchrec/inference protos/predictor.proto
```


Build inference library and example server
```
cmake -S . -B build/ \
-DCMAKE_PREFIX_PATH="$INFERENCE_CMAKE_PREFIX_PATH" \
-DPYTORCH_FMT_INCLUDE_PATH="$PYTORCH_FMT_INCLUDE_PATH" \
-DPYTORCH_LIB_FMT="$PYTORCH_LIB_FMT" \
-DDEPLOY_INTERPRETER_PATH="$DEPLOY_INTERPRETER_PATH" \
-DDEPLOY_SRC_PATH="$DEPLOY_INTERPRETER_PATH" \
-DGRPC_COMMON_CMAKE_PATH="$GRPC_COMMON_CMAKE_PATH" \
-DGRPC_HEADER_INCLUDE_PATH="$GRPC_HEADER_INCLUDE_PATH"
-DFBGEMM_LIB="$FBGEMM_LIB"

cd build
make -j
```


### **5. Run server and client**

Run server. Update `CUDA_VISABLE_DEVICES` depending on the world size.
```
CUDA_VISABLE_DEVICES="0" ./example --package_path="/tmp/model_package.zip"
```

**output**

In the logs, a plan should be outputted by the Torchrec planner:

```
INFO:<torch_package_0>.torchrec.distributed.planner.stats:#                                   --- Planner Statistics ---                                    #
INFO:<torch_package_0>.torchrec.distributed.planner.stats:#                   --- Evalulated 1 proposal(s), found 1 possible plan(s) ---                    #
INFO:<torch_package_0>.torchrec.distributed.planner.stats:# ----------------------------------------------------------------------------------------------- #
INFO:<torch_package_0>.torchrec.distributed.planner.stats:#      Rank     HBM (GB)     DDR (GB)     Perf (ms)     Input (MB)     Output (MB)     Shards     #
INFO:<torch_package_0>.torchrec.distributed.planner.stats:#    ------   ----------   ----------   -----------   ------------   -------------   --------     #
INFO:<torch_package_0>.torchrec.distributed.planner.stats:#         0     0.2 (1%)     0.0 (0%)          0.08            0.1            1.02     TW: 26     #
INFO:<torch_package_0>.torchrec.distributed.planner.stats:#                                                                                                 #
INFO:<torch_package_0>.torchrec.distributed.planner.stats:# Input: MB/iteration, Output: MB/iteration, Shards: number of tables                             #
INFO:<torch_package_0>.torchrec.distributed.planner.stats:# HBM: est. peak memory usage for shards - parameter, comms, optimizer, and gradients             #
INFO:<torch_package_0>.torchrec.distributed.planner.stats:#                                                                                                 #
INFO:<torch_package_0>.torchrec.distributed.planner.stats:# Compute Kernels:                                                                                #
INFO:<torch_package_0>.torchrec.distributed.planner.stats:#   batched_quant: 26                                                                             #
````

`nvidia-smi` output should also show allocation of the model onto the gpu:

```
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|    0   N/A  N/A     86668      C   ./example                        1357MiB |
+-----------------------------------------------------------------------------+
```

Make a request to the server via the client:

```
python client.py
```

**output**

```
Response:  [0.13199582695960999, -0.1048036441206932, -0.06022112816572189, -0.08765199035406113, -0.12735335528850555, -0.1004377081990242, 0.05509107559919357, -0.10504599660634995, 0.1350800096988678, -0.09468207508325577, 0.24013587832450867, -0.09682435542345047, 0.0025023818016052246, -0.09786031395196915, -0.26396819949150085, -0.09670191258192062, 0.2691854238510132, -0.10246685892343521, -0.2019493579864502, -0.09904996305704117, 0.3894067406654358, ...]
```

<br>

## Planned work
---
- Provide benchmarks for torch deploy vs TorchScript and cpu, single gpu and multi-gpu inference
- In-code documentation
- Simplify installation process

<br>

## Potential issues and solutions
---
Skip this section if you had no issues with installation or running the example.

**Missing header files during pytorch installation**

If your environment is missing a speicfic set of header files such as `nvml.h` and `cuda_profiler_api.h`, the pytorch installation will fail with error messages similar to the code snippet below:

```
~/nvml_lib.h:13:10: fatal error: nvml.h: No such file or directory
 #include <nvml.h>
          ^~~~~~~~
compilation terminated.
[80/2643] Building CXX object third_party/ideep/mkl-dnn/third_party/oneDNN/src/cpu/CMakeFiles/dnnl_cpu.dir/cpu_convolution_list.cpp.o
ninja: build stopped: subcommand failed.
```

To get these header files, install `cudatoolkit-dev`:
```
conda install -c conda-forge cudatoolkit-dev
```

Re-run the installation after this.

**libdouble-conversion missing**
```
~/torchrec/torchrec/inference/build$ ./example
./example: error while loading shared libraries: libdouble-conversion.so.3: cannot open shared object file: No such file or directory
```

If this issue persists even after adding double-conversion's path to $LD_LIBRARY_PATH (step 2) then solve by creating a symlink to `libdouble-conversion.so.3` with folly's installation of double-conversion:

```
sudo ln -s ~/folly-build/installed/double-conversion-skGL6pOaPHjtDwdXY-agzdwT1gvTXP0bD-7P4gKJD9I/lib/libdouble-conversion.so.3.1.4 \
libdouble-conversion.so.3
```

**Two installations of glog**
```
~/torchrec/torchrec/inference/build$ ./example
ERROR: flag 'logtostderr' was defined more than once (in files '/home/shabab/glog/src/logging.cc' and
'/home/shabab/folly-build/extracted/glog-v0.4.0.tar.gz/glog-0.4.0/src/logging.cc').
```
The above issue, along with a host of others during building, can potentially occur if libinference is pointing to two different versions of glog (if one was
previously installed in your system). You can find this out by running `ldd` on your libinference shared object within the build path. The issue can be solved by using the glog version provided by folly.

To use the glog version provided by folly, add the glog install path (in your folly-build directory) to your LD_LIBRARY_PATH much like step 2.

**Undefined symbols with std::string or cxx11**

If you get undefined symbol errors and the errors mention `std::string` or `cxx11`, it's likely
that your dependencies were compiled with different ABI values. Re-compile your dependencies
and ensure they all have the same value for `_GLIBCXX_USE_CXX11_ABI` in their build.

The ABI value of pytorch can be checked via:

```
import torch
torch._C._GLIBCXX_USE_CXX11_ABI
```
