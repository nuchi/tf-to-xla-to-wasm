# Tensorflow to WebAssembly

This is a set of **bare-bones** instructions to compile your Tensorflow model (as a `.pb` file) to WebAssembly. You can see a live demo I created of a Tensorflow model running in the browser at <https://humantoanimal.com/>.

To use these bare-bones instructions, you will need to:

1. Already know how to use Tensorflow's `tfcompile` tool to [compile your graph to XLA and from XLA to machine code](https://www.tensorflow.org/performance/xla/tfcompile),
2. Have a toolchain to compile C++ to WebAssembly, such as [Emscripten](https://kripken.github.io/emscripten-site/).

To get the most use out of this, start by making sure that you can use `tfcompile` to compile your Tensorflow graph into a standalone binary that runs natively, before attempting to compile it to WebAssembly. Likewise, make sure you can use Emscripten to compile and run a "Hello World!" program in WebAssembly.

## 1. Compile a modified `tfcompile`
```bash
$ git clone --recursive https://github.com/nuchi/tf-to-xla-to-wasm.git
$ cd tf-to-xla-to-wasm/tensorflow
$ git apply ../wasm-patch.diff
$ ./configure # configure with XLA support
$ bazel build --config=opt //tensorflow/compiler/aot:tfcompile
```
This will build the executable `bazel-bin/tensorflow/compiler/aot/tfcompile`.

## 2. Compile your graph
```bash
$ bazel-bin/tensorflow/compiler/aot/tfcompile \
    --target_triple="wasm32-unknown-unknown-wasm" \
    --target_cpu="generic" \
    --xla_cpu_multi_thread_eigen=false \
    --graph=/path/to/my_graph.pb \
    --config=/path/to/my_config.pbtxt \
    --out_function_object=out_model.o \
    --out_header=out_header.h \
    --out_metadata_object=out_helper.o \
    --cpp_class=MyClass
```
This creates a WebAssembly object file `out_model.o` and a C++ header `out_header.h`

Note: the line `--xla_cpu_multi_thread_eigen=false` is important because WebAssembly is single-threaded.

## 3. Compile and link supporting files
There are a whole bunch of supporting C++ files that will also need to be compiled and linked with `out_model.o` for it to be any use; for example, `tensorflow/compiler/tf2xla/xla_compiled_cpu_function.cc`, and any other runtime calls (such as matrix multiplication or convolution) used by your graph.

If you have been using the `tf_library` macro to compile a native binary, you can either try to create a WebAssembly toolchain in bazel so as to keep using it, or you can compile and link the dependencies by hand (or shell script or Makefile). I did the latter. You can follow the source files without too much work, by navigating the dependencies in the bazel build files. It's a bit harder to find the list of include directories; with some trial and error, I found that it should look something like the following: (from within the Tensorflow directory)
```bash
-I ./ \
-I ./bazel-tensorflow/external/eigen_archive \
-I ./bazel-tensorflow/external/nsync/public \
-I ./bazel-tensorflow/external/protobuf_archive/src \
-I ./bazel-genfiles
```

## 4. Potential problems
**Consistent LLVM versions:** I used Emscripten with LLVM's wasm backend. Support for WebAssembly in LLVM moves fast, so it's important for the version of LLVM that you use with Emscripten to be the same version that Tensorflow uses internally with `tfcompile`. If they differ too much, you'll get errors when you try to link your `out_model.o` with the Emscripten-compiled files.

You can see what version Tensorflow uses by looking in `tensorflow/tensorflow/workspace.bzl`, and searching for LLVM's git commit hash. Use that commit to build LLVM for use with Emscripten.

**Emscripten and math functions:** WebAssembly doesn't have intrinsics for some math operations, like e.g. cosine. Emscripten doesn't link all of libm (math functions) by default; if you use something like cosine in your graph, you may get a link error. Check out Emscripten's `tools/system_libs.py`, the method `create_wasm_libc_rt`, to add the extra source files you need.
