# 简介

本文档是在合肥超算平台上配置各种PIC程序的教程。由于合肥超算默认环境较为杂乱，本文档尽可能自行编译所有所需要的程序，以保证环境干净可控。

## 加载模块

合肥超算使用 `module` 工具管理环境。常用命令如下：

```bash
module list # 列出已加载的模块
module avail # 查看所有可用的模块
module load # 加载指定模块
module unload # 卸载指定模块
module purge # 清除所有模块
```

在默认情况下，运行 `module list` 会得到：

```bash
$ module list
Currently Loaded Modulefiles:
 1) compiler/devtoolset/7.3.1   2) compiler/rocm/3.3           3) compiler/intel/2017.5.239   4) mpi/intelmpi/2018.4.274
```

按照笔者的喜好，使用可用模块中的最新版本，并且清除不需要、重复或是版本较旧的模块。在 `.bashrc` 中添加以下内容[^bashrc]：

```bash
module purge
module load compiler/gcc/12.3.0 # gcc编译器
module load apps/cmake/3.21.0 # make和cmake
module load mpi/intelmpi/2021.3.0 # intel的mpi实现
```

[^bashrc]:
    `.bashrc` 文件会在每次打开新的 bash 终端窗口时运行，我们将一些常驻命令写入 `.bashrc` 作为 shell 的默认设置。
    还有另一个文件 `.bash_profile` 的作用与 `.bashrc` 相似，它会在用户每次登录时运行。
    事实上，更推荐将固定的环境设置，比如 module 和环境变量写入 `.bash_profile`，而将函数和别称写入 `.bashrc`，并且在 `.bash_profile` 中调用 `.bashrc` 以保证配置被加载。
    此外，我们希望每次更新 `.bashrc` 或是 `.bash_profile` 配置后改动立即生效，我们可以运行 `source .bashrc` 或是 `source .bash_profile` 命令使得改动生效，或是干脆重新连接终端。
    最后，将笔者的 `.bash_profile` 和 `.bashrc` 文件附上，以供参考。

    ```bash
    # .bashrc

    # Source global definitions
    if [ -f /etc/bashrc ]; then
            . /etc/bashrc
    fi

    # User specific aliases and functions
    ```

    ```bash
    # .bash_profile

    # set modules
    module purge
    module load compiler/gcc/12.3.0
    module load apps/cmake/3.21.0
    module load mpi/intelmpi/2021.3.0

    # set env variables
    export PATH=$HOME/.local/bin:$PATH
    export LD_LIBRARY_PATH=$HOME/.local/programs/python/lib:$HOME/.local/programs/hdf5/lib:$LD_LIBRARY_PATH

    # Get the aliases and functions
    if [ -f ~/.bashrc ]; then
            . ~/.bashrc
    fi
    ```

## 文件目录组织

同样的，按照笔者的喜好，我们将所有的软件安装在 `.local` 目录下。具体的目录结构组织如下：

```bash
$HOME
└── .local
    ├── bin    # 可执行文件目录，将可执行文件软链接到此文件夹下
    │   ├── @python3
    │   ├── @pip3
    │   └── ...
    ├── programs
    │   ├── source_code # 源码目录
    │   │   ├── python3.9.20
    │   │   └── ...
    │   ├── python # 软件安装目录
    │   └── ...
    ├── share
    └── state
```

推荐提前创建好需要的目录

```bash
$ mkdir -p ~/.local/bin ~/.local/programs/source_code
```

同时，我们可以默认将 `$HOME/.local/bin` 添加到 `$PATH` 中，即可方便的使用软连接到 `$HOME/.local/bin` 目录下的文件。
在 `.bashrc` 中添加：

```bash
export PATH=$HOME/.local/bin:$PATH
```

如果希望选择别的目录组织方式，则文档内涉及目录的命令均要做相应的修正，需要注意。

## 安装 Python

Python 是大多数 PIC 程序所需的依赖项。合肥超算提供了许多个版本的 Python Module，但是默认提供的版本会遇到多种依赖问题。
让我们自行编译安装 Python 以得到干净可控的环境。

<!--
!!以下内容已过时，请勿参考
### 安装 OpenSSL3.4.0

Python 依赖1.1.1及以上版本的 OpenSSL 以使用 pip，让我们首先检查系统的 Openssl 版本。
```bash
$ which openssl # 查看是否安装了OpenSSL
/usr/bin/openssl
$ openssl version -a # 查看OpenSSL的版本和库文件位置
OpenSSL 1.0.2k-fips  26 Jan 2017
built on: reproducible build, date unspecified
platform: linux-x86_64
options:  bn(64,64) md2(int) rc4(8x,int) des(idx,cisc,16,int) idea(int) blowfish(idx)
compiler: gcc -I. -I.. -I../include  -fPIC -DOPENSSL_PIC -DZLIB -DOPENSSL_THREADS -D_REENTRANT -DDSO_DLFCN -DHAVE_DLFCN_H -DKRB5_MIT -m64 -DL_ENDIAN -Wall -O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches   -m64 -mtune=generic -Wa,--noexecstack -DPURIFY -DOPENSSL_IA32_SSE2 -DOPENSSL_BN_ASM_MONT -DOPENSSL_BN_ASM_MONT5 -DOPENSSL_BN_ASM_GF2m -DRC4_ASM -DSHA1_ASM -DSHA256_ASM -DSHA512_ASM -DMD5_ASM -DAES_ASM -DVPAES_ASM -DBSAES_ASM -DWHIRLPOOL_ASM -DGHASH_ASM -DECP_NISTZ256_ASM
OPENSSLDIR: "/etc/pki/tls"
engines:  rdrand dynamic
```
系统提供的 OpenSSL 版本为 1.0.2k-fips，不满足我们的要求，需要手动编译安装满足要求的 OpenSSL 版本。
我们首先下载和并解压合适版本的 OpenSSL 源码包[^download]
```bash
$ cd ~/.local/programs/source_code/ # 进入源代码目录
$ wget https://github.com/openssl/openssl/releases/download/openssl-3.4.0/openssl-3.4.0.tar.gz # 下载 OpenSSL 源码包
$ tar -zxvf openssl-3.4.0.tar.gz # 解压缩
$ mv openssl-3.4.0.tar.gz # 删除压缩包，可选

```
[^download]:虽然文档中出现的下载操作统一使用 `wget` 命令行工具，但是因为众所周知的原因，这样下载很容易失败或是速度极慢。
笔者推荐手动在本地下载目标文件，再上传到远程目标目录下。
上传的方式有很多，比如 `ftp` 和 `scp` 命令行工具，一些 ssh 软件比如 xshell 自带的文件传输工具，以及合肥超算网页端提供的文件管理工具。
可以自行选择熟悉的方式。

接下来进入解压得到的文件夹，进行编译和安装
```bash
$ cd ./openssl3.4.0/
$ ./configure --prefix=$HOME/.local/programs/openssl # 指定安装目录，生成Makefile文件
$ make -j2 # 编译
$ make install # 安装
```
将安装后的可执行文件软链接到 `$HOME/.local/bin/` 目录下
```bash
$ ln -s $HOME/.local/programs/openssl/bin/openssl $HOME/.local/bin
$ which openssl
~/.local/bin/openssl
$ openssl version -a
OpenSSL 3.4.0 22 Oct 2024 (Library: OpenSSL 3.4.0 22 Oct 2024)
built on: Tue Jan  7 06:14:56 2025 UTC
platform: linux-x86_64
options:  bn(64,64)
compiler: gcc -fPIC -pthread -m64 -Wa,--noexecstack -Wall -O3 -DOPENSSL_USE_NODELETE -DL_ENDIAN -DOPENSSL_PIC -DOPENSSL_BUILDING_OPENSSL -DNDEBUG
OPENSSLDIR: "/work/home/ustc_wutong/.local/programs/openssl/ssl"
ENGINESDIR: "/work/home/ustc_wutong/.local/programs/openssl/lib64/engines-3"
MODULESDIR: "/work/home/ustc_wutong/.local/programs/openssl/lib64/ossl-modules"
Seeding source: os-specific
CPUINFO: OPENSSL_ia32cap=0x7ed8320b078bffff:0x209c01a9
```

目前 Python 依然无法找到我们安装的 OpenSSL，这是因为 Python 使用 `pkg-config` 工具处理依赖，而在合肥超算上 `pkg-config` 无法正确找到 OpenSSL 的库文件。
```bash
$ pkg-config --modversion openssl
1.0.2k
```
我们需要手动指定正确的库文件搜索路径，使得 `pkg-config` 工具可以寻找到我们需要的 OpenSSL 版本。
这可以通过设定 `PKG_CONFIG_PATH` 环境变量实现。
```bash
$ export PKG_CONFIG_PATH=$HOME/.local/programs/openssl/lib64/pkgconfig:$PKG_CONFIG_PATH
$ pkg-config --modversion openssl
3.4.0
```
最后我们还需要将 OpenSSL 库路径添加到 `LD_LIBRARY_PATH` 环境变量[^LD_LIBRARY_PATH]，将以下内容写入 `.bashrc`：
```bash
export LD_LIBRARY_PATH=$HOME/.local/programs/openssl/lib64:$LD_LIBRARY_PATH
```
[^LD_LIBRARY_PATH]:`LD_LIBRARY_PATH` 环境变量指定了运行时库的位置。
对于我们当前的例子（pip 依赖 OpenSSL），编译时需要的动态链接库和运行时需要的动态链接库目录相同，前者由 `pkg-config` 工具指定。
所以事实上，在编译阶段，我们不需要 `LD_LIBRARY_PATH` 这一环境变量。我们只需要保证每次运行 `pip3` 命令前 `LD_LIBRARY_PATH` 被正确设定就好。

### 安装 Python

至此我们满足了 Python 所需的依赖项，可以正式安装 Python 了！我们首先下载并解压 Python 的源码包
-->

Python3.10 以上版本依赖1.1.1以上版本的 OpenSSL，这一依赖在合肥超算上没有默认提供。为了避免麻烦，我们选择 3.10 以下版本的 Python3，这里选择3.9.20[^download]

```bash
$ cd ~/.local/programs/source_code/ # 进入源代码目录
$ wget https://www.python.org/ftp/python/3.9.20/Python-3.9.20.tgz # 下载 Python 源码包
$ tar -zxvf Python-3.9.20.tgz # 解压
$ rm Python-3.9.20.tgz # 删除压缩包，可选
```
[^download]:虽然文档中出现的下载操作统一使用 `wget` 命令行工具，但是因为众所周知的原因，这样下载很容易失败或是速度极慢。
笔者推荐手动在本地下载目标文件，再上传到远程目标目录下。
上传的方式有很多，比如 `ftp` 和 `scp` 命令行工具，一些 ssh 软件比如 xshell 自带的文件传输工具，以及合肥超算网页端提供的文件管理工具。
可以自行选择熟悉的方式。

接下来进行编译和安装：

```bash
$ cd ./Python-3.13.0/ # 进入 Python 源代码目录
$ ./configure --prefix=$HOME/.local/programs/python --enable-shared # 指定安装目录，生成 Makefile 文件
$ make -j2 # 编译
$ make install # 安装
```

现在我们应该可以在 `$HOME/.local/programs/python/` 目录找到安装完成的 Python 了。

```bash
$ cd ~/.local/programs/python/
$ ls
bin/ include/ lib/ share/
```

最后，我们将需要的 Python 可执行文件软链接到 `$HOME/.local/bin/` 目录下：

```bash
$ ln -s $HOME/.local/programs/python/bin/python3.13 $HOME/.local/bin/python3
$ ln -s $HOME/.local/programs/python/bin/pip3.13 $HOME/.local/bin/pip3
```

这样就可以方便地运行我们安装的 Python 了：

```bash
$ which python3
/public/home/$YOUR_ID/.local/bin/python3
```

不要忘记下载 numpy，这是大多数依赖 Python 的 PIC 程序同时依赖的库：

```bash
$ pip3 install numpy -i https://pypi.tuna.tsinghua.edu.cn/simple # 通过清华源下载 numpy 库
```

此外，我们还需要把 Python 的动态链接库路径添加到 `LD_LIBRARY_PATH` [^LD_LIBRARY_PATH]环境变量：

```bash
export LD_LIBRARY_PATH=$HOME/.local/programs/python/lib:$LD_LIBRARY_PATH
```
[^LD_LIBRARY_PATH]:`LD_LIBRARY_PATH` 环境变量指定了运行时库的位置。
对于我们当前的例子（pip 依赖 OpenSSL），编译时需要的动态链接库和运行时需要的动态链接库目录相同，前者由 `pkg-config` 工具指定。
所以事实上，在编译阶段，我们不需要 `LD_LIBRARY_PATH` 这一环境变量。我们只需要保证每次运行 `pip3` 命令前 `LD_LIBRARY_PATH` 被正确设定就好。

## 安装 SymPIC

如果正确配置了模块并安装了 Python，那我们已经满足了 SymPIC 需要的依赖，但还有最后一个前置步骤。
SymPIC的安装依赖 Python 提供的 `python3-config` 脚本，而这一脚本的输出又依赖于他所在的目录，故我们无法把他软链接到 `$HOME/.local/bin` 文件夹。
让我们临时把 Python 的安装目录添加到 `PATH` 环境变量，使得 `python3-config` 脚本能够按预期工作。

```bash
$ export PATH=$HOME/.local/programs/python/bin:$PATH
```

接下来，将 SymPIC 的 tar 包上传到 `$HOME/.local/programs` 目录并解压：

```bash
$ tar -zxvf sympic_avx2.tar.gz
```

只需要在解压后的文件夹内运行 `compile.sh` 脚本即可完成编译安装：

```bash
$ cd $HOME/.local/programs/sympic_avx2
$ ./compile.sh
```

编译完成的可执行文件为 `comp_python`，将其软链接到 `$HOME/.local/bin`：

```bash
$ ln -s $HOME/.local/programs/sympic_avx2/comp_python $HOME/.local/bin/comp_python
```

SymPIC的安装就完成了。

## 安装 Smilei

在合肥超算上，Smilei 的依赖项有 Python 和 parallel HDF5，我们已经安装了 Python，只需要安装 HDF5 就行了。
与之前安装 OpenSSL 和 Python 一样，我们需要进行的流程为下载-解压缩-编译-安装-调整环境变量。

### 安装 HDF5

下载解压

```bash
$ cd $HOME/.local/programs/source_code/
$ wget https://github.com/HDFGroup/hdf5/releases/download/hdf5_1.14.5/hdf5-1.14.5.tar.gz
$ tar -zxvf hdf5-1.14.5.tar.gz
$ rm hdf5-1.14.5.tar.gz
```

编译安装

```bash
$ cd ./hdf5-1.14.5/
$ ./configure --prefix=$HOME/.local/programs/hdf5 --enable-parallel --with-pic --enable-shared --enable-build-mode=production --disable-sharedlib-rpath --enable-static CC=mpicc FC=mpif90
$ make -j2
$ make install
```

添加环境变量

```bash
export LD_LIBRARY_PATH=$HOME/.local/programs/hdf5/lib:$HOME/.local/programs/python/lib:$HOME/.local/programs/openssl/lib64:$LD_LIBRARY_PATH
```

### 安装 Smilei

Smilei 提供了脚本处理依赖，我们只需要提供合适的环境变量即可

```bash
$ export HDF5_ROOT_DIR=$HOME/.local/programs/hdf5
$ export PYTHONEXE=python3
```

接下来我们进行相同的流程，下载解压

```bash
$ cd $HOME/.local/programs
$ wget https://github.com/SmileiPIC/Smilei/archive/refs/tags/v5.1.tar.gz
$ tar -zxvf Smilei-5.1.tar.gz
$ rm Smilei-5.1.tar.gz
```

编译

```bash
$ cd ./Smilei-5.1/
$ make -j2
```

最后将 `smilei` 软链接到 `$HOME/.local/bin`

```bash
$ ln -s $HOME/.local/programs/Smilei-5.1/smilei $HOME/.local/bin/smilei
```
