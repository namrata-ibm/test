# Building Envoy

The instructions provided below specify the steps to build [Envoy](https://www.envoyproxy.io/) version 1.29.2 on Linux on IBM Z for the following distributions:
*   Ubuntu (22.04)

_**General Notes:**_
*   _When following the steps below please use a standard permission user unless otherwise specified._
*   _A directory `/<source_root>/` will be referred to in these instructions, this is a temporary writable directory anywhere you'd like to place it._

### Build and Install Envoy

#### Step 1: Build using script

If you want to build Envoy using manual steps, go to Step 2.

Use the following commands to build Envoy using the build script.

```bash
# Build Envoy
bash build_envoy.sh   [Provide -t option for executing build with tests]
```

In case of error, check `logs` for more details or go to STEP 2 to follow manual build steps.

#### Step 2: Install dependencies

```bash
export SOURCE_ROOT=/<source_root>/
```
*   Install basic dependencies
    ```bash
    sudo apt-get update 
    sudo apt install -y lsb-release wget software-properties-common gnupg 

    sudo apt-get update
    sudo apt-get install -y autoconf curl libtool patch python3-pip unzip virtualenv pkg-config 
    ```

*   Install Bazel 6.4.0 
    ```bash
    cd $SOURCE_ROOT
  
    wget -q https://raw.githubusercontent.com/linux-on-ibm-z/scripts/master/Bazel/6.4.0/build_bazel.sh
    sed -i 's/apt-get install/DEBIAN_FRONTEND=noninteractive apt-get install/'  build_bazel.sh
    bash build_bazel.sh -y

    export PATH=$PATH:${SOURCE_ROOT}/bazel/output/
    ```
    
*   Install GCC 12 from repo
    ```bash
    sudo apt-get install -y gcc-12 g++-12

    sudo update-alternatives --install /usr/bin/cc cc /usr/bin/gcc-12 12
    sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-12 12
    sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-12 12
    sudo update-alternatives --install /usr/bin/c++ c++ /usr/bin/g++-12 12
    ```

*   Install Clang 14
    ```bash
    cd $SOURCE_ROOT
    wget https://apt.llvm.org/llvm.sh
    sed -i 's,add-apt-repository "${REPO_NAME}",add-apt-repository "${REPO_NAME}" -y,g' llvm.sh
    chmod +x llvm.sh
    sudo ./llvm.sh 14
    rm ./llvm.sh

    export CC=clang-14
    export CXX=clang++-14

    sudo ln -sf /usr/bin/clang-14 /usr/bin/clang
    sudo ln -sf /usr/bin/clang++-14 /usr/bin/clang++
    ```
    
*   Install Rust
    ```bash
    cd $SOURCE_ROOT
    curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh /dev/stdin -y
    export PATH="$HOME/.cargo/bin:$PATH"
    ```

*   Install Go `1.21.5`
    ```bash
    export GOPATH=$SOURCE_ROOT
    cd $GOPATH
    GO_VERSION=1.21.5
    wget -q https://golang.org/dl/go"${GO_VERSION}".linux-s390x.tar.gz
    chmod ugo+r go"${GO_VERSION}".linux-s390x.tar.gz
    sudo rm -rf /usr/local/go /usr/bin/go
    sudo tar -C /usr/local -xzf go"${GO_VERSION}".linux-s390x.tar.gz
    sudo ln -sf /usr/local/go/bin/go /usr/bin/ 
    sudo ln -sf /usr/local/go/bin/gofmt /usr/bin/
    go version  
    export PATH=$PATH:$GOPATH/bin
    ```

*   Install Buildifier and Buildozer
    ```bash
    cd $SOURCE_ROOT
    git clone -b v6.4.0 https://github.com/bazelbuild/buildtools.git
    
    #Build buildifer
    cd $SOURCE_ROOT/buildtools/buildifier
    bazel build //buildifier
    export BUILDIFIER_BIN=$GOPATH/bin/buildifier

    #Build buildozer
    cd $SOURCE_ROOT/buildtools/buildozer
    bazel build //buildozer 
    export BUILDOZER_BIN=$GOPATH/bin/buildozer
    ```
    
    OR

    ```bash
    cd $SOURCE_ROOT
    go install github.com/bazelbuild/buildtools/buildifier@latest
    go install github.com/bazelbuild/buildtools/buildozer@latest

    export BUILDIFIER_BIN=$GOPATH/bin/buildifier
    export BUILDOZER_BIN=$GOPATH/bin/buildozer
    ```

#### Step 3: Build and Install Envoy

*   Clone rules_foreign_cc
    ```bash
    cd $SOURCE_ROOT
    git clone --depth 1 -b 0.10.1 https://github.com/bazelbuild/rules_foreign_cc.git
	cd rules_foreign_cc/
	git apply $SOURCE_ROOT/patch/rules_foreign_cc.patch
    cp $SOURCE_ROOT/patch/pkgconfig-valgrind.patch toolchains/pkgconfig-valgrind.patch
    ```

*   Build and install Envoy
    ```bash
    cd $SOURCE_ROOT
    git clone --depth 1 -b v1.29.2 https://github.com/envoyproxy/envoy.git
    cd envoy
    ```

    ```bash
    # Apply patches-
    git apply $SOURCE_ROOT/patch/envoy_patch.diff
    git apply $SOURCE_ROOT/patch/luajit_patch.diff

    # Patch for failing tests
    git apply $SOURCE_ROOT/patch/envoy-test.patch

    # Copy patch files to envoy/bazel which will be applied to external packages while building envoy
    cp $SOURCE_ROOT/patch/boringssl-s390x.patch bazel/boringssl-s390x.patch
    cp $SOURCE_ROOT/patch/cel-cpp-memory.patch bazel/cel-cpp-memory.patch
    cp $SOURCE_ROOT/patch/grpc-s390x.patch bazel/grpc-s390x.patch
    cp $SOURCE_ROOT/patch/quiche-s390x.patch bazel/quiche-s390x.patch
    ```

    ```bash
    bazel build envoy -c opt --override_repository=rules_foreign_cc=${SOURCE_ROOT}/rules_foreign_cc --config=clang
    ```

    The binary will be generated in `$SOURCE_ROOT/envoy/source/exe/envoy-static`.

*   Verify the version of Envoy
    ```bash
    $SOURCE_ROOT/envoy/bazel-bin/source/exe/envoy-static --version
    ```
    Output should be similar to:
    ```bash
    bazel-bin/source/exe/envoy-static  version: 2092d65bd4d476be8235ea541e5d25c096b513e6/1.29.2/Modified/RELEASE/BoringSSL
    ```

#### Step 4: Test (optional)
```bash
cd $SOURCE_ROOT/envoy
bazel test //test/... -c opt --override_repository=rules_foreign_cc=${SOURCE_ROOT}/rules_foreign_cc --config=clang --keep_going --test_output=all
```

_**Notes:**_

Below mentioned test failures are observed on Ubuntu 22.04 and Investigation is in progress-
```bash
//test/extensions/filters/listener/original_dst:original_dst_integration_test
//test/extensions/tracers/dynamic_ot:dynamic_opentracing_driver_impl_test
//test/integration:hotrestart_handoff_test
```

Some more tests fail due to timeout, but pass when heapcheck is disabled.
```bash
# Disables the heap checker
bazel test //test/... -c opt --override_repository=rules_foreign_cc=${SOURCE_ROOT}/rules_foreign_cc --config=clang --keep_going --test_output=all --test_env=HEAPCHECK=
```

#### References:
- https://github.com/envoyproxy/envoy/blob/v1.29.2/bazel/README.md
