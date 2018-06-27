## Minimalist static FFmpeg build on Ubuntu 16.04 with Nvidia NVENC enabled.

Original guide with a standard build is [here](https://gist.github.com/Brainiarc7/95c9338a737aa36d9bb2931bed379219).

With this guide, I'm adding more instructions to enable support for [NVIDIA CUVID](http://docs.nvidia.com/cuda/video-decoder/) and [NVIDIA NPP](https://developer.nvidia.com/npp) for enhanced encode and decode performance.

**Warning:**

If **all** you require is NVENC's enablement, you do NOT need the CUDA SDK.
The `nv-codec-headers` (below) is ALL you require.
However, the SDK is needed IF, and only IF, the usage of the `scale_npp` and any other CUDA-based filters is required.

With that in mind, do note that the NVIDIA proprietary driver is mandatory. See the driver setup instructions below, and the warning notes for Ubuntu 18.04LTS.

**Steps:**

First, prepare for the build and create the work space directory:

```
cd ~/
mkdir ~/ffmpeg_sources
sudo apt-get -y update && apt-get dist-upgrade -y
sudo apt-get -y install autoconf automake build-essential libass-dev \
  libtool \
  pkg-config texinfo zlib1g-dev cmake mercurial

```

**Install CUDA 9.2 SDK from Nvidia's repository:**

Note that this phase will prompt you to install the device driver. Skip it, and skip the samples too.We will install the driver later. Fetch the repository installers first:

```
cd ~/ffmpeg_sources

wget -c -v -nc https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/cuda-repo-ubuntu1604_9.2.88-1_amd64.deb

sudo dpkg -i cuda-repo-ubuntu1604_9.2.88-1_amd64.deb

sudo apt-key adv --fetch-keys http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/7fa2af80.pub

sudo apt-get update

sudo apt-get install cuda

```

Ensure that you have the latest driver:

```
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt-get update && sudo apt-get -y upgrade

```

On Ubuntu 18.04LTS, this should be enough for the device driver:

```
nvidia-kernel-source-396 nvidia-kernel-source-396 nvidia-driver-396

```

We keep the device driver up to the latest version so as to pass FFmpeg's NVENC driver version check.

Now, set up the environment variables for CUDA:

Edit the `/etc/environment` file and append the following:

```
CUDA_HOME=/usr/local/cuda

```

Now, append the PATH variable with the following:

```
/usr/local/cuda/bin:$HOME/bin

```

When done, remember to source the file:

```
source /etc/environment

```

**Build FFmpeg's dependency chain:**

**Build and deploy nasm:** [Nasm](http://www.nasm.us/) is an assembler for x86 optimizations used by x264 and FFmpeg. Highly recommended or your resulting build may be very slow. Note that we're using the latest release candidate, and not the stable version as of the time of writing.

```
cd ~/ffmpeg_sources
wget http://www.nasm.us/pub/nasm/releasebuilds/2.14rc0/nasm-2.14rc0.tar.gz
tar xzvf nasm-2.14rc0.tar.gz
cd nasm-2.14rc0
./configure --prefix="$HOME/ffmpeg_build" --bindir="$HOME/bin"
make -j$(nproc) VERBOSE=1
make -j$(nproc) install
make -j$(nproc) distclean

```

**Build and deploy libx264 statically:** This library provides a H.264 video encoder. See the [H.264 Encoding Guide](https://trac.ffmpeg.org/wiki/Encode/H.264) for more information and usage examples. This requires ffmpeg to be configured with _--enable-gpl_ _--enable-libx264_.

```
cd ~/ffmpeg_sources
git clone http://git.videolan.org/git/x264.git -b stable
cd x264/
PATH="$HOME/bin:$PATH" ./configure --prefix="$HOME/ffmpeg_build" --bindir="$HOME/bin" --enable-static --enable-shared --disable-opencl
PATH="$HOME/bin:$PATH" make -j$(nproc) VERBOSE=1
make -j$(nproc) install
make -j$(nproc) distclean

```

**Note:** If you need to enable OpenCL support for either libx264 or FFmpeg, ensure that:

(a). The flag `--disable-opencl` is removed from libx264's configuration.

(b). The flag `enable-opencl` is present in FFmpeg's configure options.

(c ). The prerequisite packages for OpenCL development are present:

With OpenCL, the [installable client drivers](https://www.khronos.org/news/permalink/opencl-installable-client-driver-icd-loader) (ICDs) are normally issued with the accelerator's device drivers, namely:

1.  The [NVIDIA CUDA toolkit](https://developer.nvidia.com/cuda-toolkit) (and the device driver) for NVIDIA GPUs.
2.  AMD's [RoCM](https://github.com/RadeonOpenCompute/ROCm) for GCN-class AMD hardware.
3.  Intel's [beignet](https://github.com/intel/beignet) and the newer [Neo compute runtime](https://github.com/intel/compute-runtime).

The purpose of the installable client driver model is to allow multiple OpenCL platforms to coexist on the same platform. That way, multiple OpenCL accelerators, be they discrete GPUs paired with a combination of FPGAs and integrated GPUs can all coexist.

However, for linkage purposes, you'll require the [ocl-icd](https://github.com/OCL-dev/ocl-icd) package, which can be installed by:

```
sudo apt install ocl-icd-* 

```

Why ocl-icd? Simple: Whereas other ICDs may permit you to link against them directly, it is discouraged so as to limit the risk of unexpected runtime behavior. Assume ocl-icd to be the gold link target if your goal is to be platform-neutral as possible.

**OpenCL in FFmpeg:**

OpenCL's enablement in FFmpeg comes in two ways:  
  
**(a):.** Some encoders, such as libx264, if built with OpenCL enablement, can utilize these capabilities for accelerated lookahead functions. The performance impact for this enablement will vary with the GPU on the platform, and with older GPUs, may slow down the encoder. Lower power platforms such as specific AMD APUs and their SoCs may see modest performance improvements at best, but on modern, high performance GPUs, your mileage may vary. Expect no miracles. The reason OpenCL lookahead is available for this library in particular is that the lookahead algorithms for OpenCL are easily parallelized.  
  

For instance, you can combine the `-hwaccel auto` option which allows you to select the hardware-based accelerated decoding to use for the encode session with libx264. You can add this parameter with "auto" before input (if your x264 is compiled with OpenCL support you can try to add -x264opts param), for example:

```
ffmpeg -hwaccel auto -i input -vcodec libx264 -x264opts opencl output

```

**(b):.** FFmpeg, in particular, can utilize OpenCL with *some* filters, namely [program_opencl](https://ffmpeg.org/ffmpeg-filters.html#program_005fopencl-1) and [opencl_src](https://ffmpeg.org/ffmpeg-filters.html#openclsrc) as documented in the filters documentation, among others.  
  
See the sample command below:

```
ffmpeg -hide_banner -v verbose -init_hw_device 
opencl=ocl:1.0 -filter_hw_device ocl -i 
"cheeks.mkv" -an -map_metadata -1 -sws_flags 
lanczos+accurate_rnd+full_chroma_int+full_chroma_inp -filter_complex 
"[0:v]yadif=0:0:0,hwupload,unsharp_opencl=lx=3:ly=3:la=0.5:cx=3:cy=3:ca=0.5,hwdownload,setdar=dar=16/9" 
 -r 25 -c:v h264_nvenc -preset:v llhq -bf 2 -g 50 -refs 3 -rc:v 
vbr_hq -rc-lookahead:v 32 -coder:v cabac -movflags 
+faststart -profile:v high -level 4.1 -pixel_format yuv420p -y 
".crunchy_cheeks.mp4"

```

List OpenCL platform devices:

```
ffmpeg -hide_banner -v verbose -init_hw_device list
ffmpeg -hide_banner -v verbose -init_hw_device opencl
ffmpeg -hide_banner -v verbose -init_hw_device opencl:1.0 

```

For the filter, see:

```
ffmpeg -hide_banner -v verbose -h filter=unsharp_opencl 

```

**Bonus score:**
If you're adventurous, you could also try out [this](https://github.com/ittiamvpx/libvpx) OpenCL build of libvpx from Ittiam systems, especially if you're using Integrated graphics or an FPGA (Xilinx).

**Carrying on:**

**Build and configure libx265:** This library provides a H.265/HEVC video encoder. See the [H.265 Encoding Guide](https://trac.ffmpeg.org/wiki/Encode/H.265) for more information and usage examples.

```
cd ~/ffmpeg_sources
hg clone https://bitbucket.org/multicoreware/x265
cd ~/ffmpeg_sources/x265/build/linux
PATH="$HOME/bin:$PATH" cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX="$HOME/ffmpeg_build" -DENABLE_SHARED:bool=off ../../source
make -j$(nproc) VERBOSE=1
make -j$(nproc) install
make -j$(nproc) clean

```

**Build and deploy the libfdk-aac library:** This provides an AAC audio encoder. See the [AAC Audio Encoding Guide](https://trac.ffmpeg.org/wiki/Encode/AAC) for more information and usage examples. This requires ffmpeg to be configured with _--enable-libfdk-aac_ (and _--enable-nonfree_ if you also included _--enable-gpl_).

```
cd ~/ffmpeg_sources
wget -O fdk-aac.tar.gz https://github.com/mstorsjo/fdk-aac/tarball/master
tar xzvf fdk-aac.tar.gz
cd mstorsjo-fdk-aac*
autoreconf -fiv
./configure --prefix="$HOME/ffmpeg_build" --disable-shared
make -j$(nproc) VERBOSE=1
make -j$(nproc) install
make -j$(nproc) distclean

```

**Build and deploy libaom:**

This library implements the Alliance for Open Media Video Codec reference implementation, and enabling the configuration switch `--enable-libaom` will enable FFmpeg's AV1 encoders.

```
mkdir -p ~/ffmpeg_sources/libaom
cd ~/ffmpeg_sources/libaom 
git clone https://aomedia.googlesource.com/aom
cmake ./aom -DENABLE_CCACHE=1 -DCMAKE_BUILD_TYPE=Release -DCONFIG_MULTITHREAD=1 -DCMAKE_INSTALL_PREFIX="$HOME/ffmpeg_build" \
-DCONFIG_LOWBITDEPTH=1 -DCONFIG_HIGHBITDEPTH=1 \
-DCONFIG_AV1=1 -DHAVE_PTHREAD=1 -DBUILD_SHARED_LIBS=0 -DENABLE_DOCS=0 -DCONFIG_INSTALL_DOCS=0 \
-DCONFIG_INSTALL_BINS=1 -DCONFIG_INSTALL_LIBS=1 \
-DCONFIG_INSTALL_SRCS=1 -DCONFIG_UNIT_TESTS=0 \
-DCONFIG_AV1_DECODER=1 -DCONFIG_AV1_ENCODER=1 \
-DCONFIG_MULTITHREAD=1 -DCONFIG_PIC=1 -DCONFIG_COEFFICIENT_RANGE_CHECKING=1 \
-DCONFIG_RUNTIME_CPU_DETECT=1 -DAOM_TARGET_CPU=generic -DCONFIG_WEBM_IO=1 \
-DCMAKE_SYSTEM_PROCESSOR=x86_64 -DCONFIG_SPATIAL_RESAMPLING=1 -DENABLE_NASM=on
time make -j$(nproc) VERBOSE=1
make install -j$(nproc) VERBOSE=1
```



Take note that [changes to the inclusion of third party headers](https://git.videolan.org/?p=ffmpeg/nv-codec-headers.git) affects new builds, and this is fixed by:

```
cd ~/ffmpeg_sources
git clone https://git.videolan.org/git/ffmpeg/nv-codec-headers.git
cd nv-codec-headers
make
sudo make install


```

Proceed as usual:

**Building a static ffmpeg binary with the required options:**

```
cd ~/ffmpeg_sources
git clone https://github.com/FFmpeg/FFmpeg -b master
cd FFmpeg
PATH="$HOME/bin:$PATH" PKG_CONFIG_PATH="$HOME/ffmpeg_build/lib/pkgconfig" ./configure \
  --prefix="$HOME/ffmpeg_build" \
  --pkg-config-flags="--static" \
  --extra-cflags="-I$HOME/ffmpeg_build/include" \
  --extra-ldflags="-L$HOME/ffmpeg_build/lib" \
  --bindir="$HOME/bin" \
  --enable-cuda-sdk \
  --enable-cuvid \
  --enable-libnpp \
  --extra-cflags="-I/usr/local/cuda/include/" \
  --extra-ldflags=-L/usr/local/cuda/lib64/ \
  --enable-gpl \
  --enable-libaom \
  --enable-libass \
  --enable-libfdk-aac \
  --enable-libx264 \
  --enable-libx265 \
  --enable-nvenc \
  --enable-nonfree
PATH="$HOME/bin:$PATH" make -j$(nproc) VERBOSE=1
make -j$(nproc) install
make -j$(nproc) distclean
hash -r

```

You may also want to tune your build further by calling upon NVCC to generate a build optimized for your GPU's CUDA architecture only.

The example below shows the build options to pass for Pascal's GM10x-series GPUs, with an SM version of 6.1:

```
cd ~/ffmpeg_sources
git clone https://github.com/FFmpeg/FFmpeg -b master
cd FFmpeg
PATH="$HOME/bin:$PATH" PKG_CONFIG_PATH="$HOME/ffmpeg_build/lib/pkgconfig" ./configure \
  --prefix="$HOME/ffmpeg_build" \
  --pkg-config-flags="--static" \
  --extra-cflags="-I$HOME/ffmpeg_build/include" \
  --extra-ldflags="-L$HOME/ffmpeg_build/lib" \
  --bindir="$HOME/bin" \
  --enable-cuda-sdk \
  --enable-cuvid \
  --enable-libnpp \
  --extra-cflags="-I/usr/local/cuda/include/" \
  --extra-ldflags=-L/usr/local/cuda/lib64/ \
  --nvccflags="-gencode arch=compute_61,code=sm_61 -O2" \
  --enable-gpl \
  --enable-libaom \
  --enable-libass \
  --enable-libfdk-aac \
  --enable-libx264 \
  --extra-libs=-lpthread \
  --enable-libx265 \
  --enable-nvenc \
  --enable-nonfree
PATH="$HOME/bin:$PATH" make -j$(nproc) VERBOSE=1
make -j$(nproc) install
make -j$(nproc) distclean
hash -r

```

For the older Maxwell (GM204*-series) cards, the build below will generate optimized binaries for that CUDA architecture:

```
cd ~/ffmpeg_sources
git clone https://github.com/FFmpeg/FFmpeg -b master
cd FFmpeg
PATH="$HOME/bin:$PATH" PKG_CONFIG_PATH="$HOME/ffmpeg_build/lib/pkgconfig" ./configure \
--prefix="$HOME/ffmpeg_build" \
--pkg-config-flags="--static" \
--extra-cflags="-I$HOME/ffmpeg_build/include" \
--extra-ldflags="-L$HOME/ffmpeg_build/lib" \
--bindir="$HOME/bin" \
--enable-cuda \
--enable-cuvid \
--enable-libnpp \
--extra-cflags="-I/usr/local/cuda/include/" \
--extra-ldflags=-L/usr/local/cuda/lib64/ \
--nvccflags="-gencode arch=compute_52,code=sm_52 -O2" \
--enable-gpl \
--enable-libaom \
--enable-libass \
--enable-libfdk-aac \
--enable-libx264 \
--enable-libx265 \
--extra-libs=-lpthread \
--enable-nvenc \
--enable-nonfree
PATH="$HOME/bin:$PATH" make -j$(nproc)
make -j$(nproc) install
make -j$(nproc) distclean
hash -r

```

Remember the notes above on builds that do not require any CUDA-based filters? This build below will give you exactly that:

```
cd ~/ffmpeg_sources
git clone https://github.com/FFmpeg/FFmpeg -b master
cd FFmpeg
PATH="$HOME/bin:$PATH" PKG_CONFIG_PATH="$HOME/ffmpeg_build/lib/pkgconfig" ./configure \
  --prefix="$HOME/ffmpeg_build" \
  --pkg-config-flags="--static" \
  --extra-cflags="-I$HOME/ffmpeg_build/include" \
  --extra-ldflags="-L$HOME/ffmpeg_build/lib" \
  --bindir="$HOME/bin" \
  --enable-gpl \
  --enable-libaom \
  --enable-libass \
  --enable-libfdk-aac \
  --enable-libx264 \
  --enable-libx265 \
  --enable-nvenc \
  --enable-nonfree
PATH="$HOME/bin:$PATH" make -j$(nproc) VERBOSE=1
make -j$(nproc) install
make -j$(nproc) distclean
hash -r

```

**Handling package upgrades:**

For individual packages availed via git, simply navigate to their source directory and run git pull followed by re-building them:

**(a). For nasm:**

```
cd ~/ffmpeg_sources
git clone git://repo.or.cz/nasm.git
cd nasm
./autogen.sh
./configure --prefix="$HOME/ffmpeg_build" --bindir="$HOME/bin"
make -j$(nproc) VERBOSE=1
make -j$(nproc) install
make -j$(nproc) distclean

```

**(b). For x264:**

```
cd ~/ffmpeg_sources/x264
git pull
PATH="$HOME/bin:$PATH" ./configure --prefix="$HOME/ffmpeg_build" --bindir="$HOME/bin" --enable-shared --disable-opencl
PATH="$HOME/bin:$PATH" make -j$(nproc) VERBOSE=1
make -j$(nproc) install
make -j$(nproc) distclean

```

**(c). For x265:**

```
cd ~/ffmpeg_sources/x265
hg pull
hg update
cd ~/ffmpeg_sources/x265/build/linux
PATH="$HOME/bin:$PATH" cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX="$HOME/ffmpeg_build" -DENABLE_SHARED:bool=on ../../source
make -j$(nproc) VERBOSE=1
make -j$(nproc) install
make -j$(nproc) clean

```

**(d). For the FFmpeg NVENC headers:**

```
cd nv-codec-headers
git pull
make
sudo make install

```

(e). For libaom:

```
cd ~/ffmpeg_sources/libaom/aom
git pull
cd ..
cmake ./aom -DENABLE_CCACHE=1 -DCMAKE_BUILD_TYPE=Release -DCONFIG_MULTITHREAD=1 -DCMAKE_INSTALL_PREFIX="$HOME/ffmpeg_build" \
-DCONFIG_LOWBITDEPTH=1 -DCONFIG_HIGHBITDEPTH=1 \
-DCONFIG_AV1=1 -DHAVE_PTHREAD=1 -DBUILD_SHARED_LIBS=0 -DENABLE_DOCS=0 -DCONFIG_INSTALL_DOCS=0 \
-DCONFIG_INSTALL_BINS=1 -DCONFIG_INSTALL_LIBS=1 \
-DCONFIG_INSTALL_SRCS=1 -DCONFIG_UNIT_TESTS=0 \
-DCONFIG_AV1_DECODER=1 -DCONFIG_AV1_ENCODER=1 \
-DCONFIG_MULTITHREAD=1 -DCONFIG_PIC=1 -DCONFIG_COEFFICIENT_RANGE_CHECKING=1 \
-DCONFIG_RUNTIME_CPU_DETECT=1 -DAOM_TARGET_CPU=generic -DCONFIG_WEBM_IO=1 \
-DCMAKE_SYSTEM_PROCESSOR=x86_64 -DCONFIG_SPATIAL_RESAMPLING=1 -DENABLE_NASM=on
time make -j$(nproc) VERBOSE=1
make install -j$(nproc) VERBOSE=1

```


**(f). For FFmpeg:**

**i. For Pascal - based GPU systems (GP10x):**

```
cd ~/ffmpeg_sources/FFmpeg
git pull
PATH="$HOME/bin:$PATH" PKG_CONFIG_PATH="$HOME/ffmpeg_build/lib/pkgconfig:/usr/lib/x86_64-linux-gnu/pkgconfig" ./configure \
  --prefix="$HOME/ffmpeg_build" \
  --pkg-config-flags="--shared" \
  --extra-cflags="-I$HOME/ffmpeg_build/include" \
  --extra-ldflags="-L$HOME/ffmpeg_build/lib" \
  --bindir="$HOME/bin" \
  --enable-cuda-sdk \
  --enable-cuvid \
  --enable-libnpp \
  --extra-cflags="-I/usr/local/cuda/include/" \
  --extra-ldflags=-L/usr/local/cuda/lib64/ \
  --nvccflags="-gencode arch=compute_61,code=sm_61 -O2" \
  --enable-gpl \
  --enable-libaom \
  --enable-libass \
  --enable-libfdk-aac \
  --enable-libx264 \
  --extra-libs=-lpthread \
  --enable-libx265 \
  --enable-nvenc \
  --enable-nonfree
PATH="$HOME/bin:$PATH" make -j$(nproc) VERBOSE=1
make -j$(nproc) install
make -j$(nproc) distclean
hash -r

```

**ii. For Maxwell (GM20x-based Tesla systems):**

```
cd ~/ffmpeg_sources/FFmpeg
git pull
PATH="$HOME/bin:$PATH" PKG_CONFIG_PATH="$HOME/ffmpeg_build/lib/pkgconfig:/usr/lib/x86_64-linux-gnu/pkgconfig" ./configure \
--prefix="$HOME/ffmpeg_build" \
--extra-cflags="-I$HOME/ffmpeg_build/include" \
--extra-ldflags="-L$HOME/ffmpeg_build/lib" \
--bindir="$HOME/bin" \
--enable-cuda \
--enable-cuvid \
--enable-libnpp \
--extra-cflags="-I/usr/local/cuda/include/" \
--extra-ldflags=-L/usr/local/cuda/lib64/ \
--nvccflags="-gencode arch=compute_52,code=sm_52 -O2" \
--enable-gpl \
--enable-libaom \
--enable-libass \
--enable-libfdk-aac \
--enable-libx264 \
--enable-libx265 \
--extra-libs=-lpthread \
--enable-nvenc \
--enable-nonfree
PATH="$HOME/bin:$PATH" make -j$(nproc) VERBOSE=1
make -j$(nproc) install
make -j$(nproc) distclean
hash -r

```

3. An FFmpeg build with NVENC only (without CUDA dependencies):

```
cd ~/ffmpeg_sources/FFmpeg
git pull
PATH="$HOME/bin:$PATH" PKG_CONFIG_PATH="$HOME/ffmpeg_build/lib/pkgconfig" ./configure \
  --prefix="$HOME/ffmpeg_build" \
  --pkg-config-flags="--static" \
  --extra-cflags="-I$HOME/ffmpeg_build/include" \
  --extra-ldflags="-L$HOME/ffmpeg_build/lib" \
  --bindir="$HOME/bin" \
  --enable-gpl \
  --enable-libaom \
  --enable-libass \
  --enable-libfdk-aac \
  --enable-libx264 \
  --enable-libx265 \
  --enable-nvenc \
  --enable-nonfree
PATH="$HOME/bin:$PATH" make -j$(nproc) VERBOSE=1
make -j$(nproc) install
make -j$(nproc) distclean
hash -r

```

**On nasm:**

We build nasm from source, using the git master tip as it contains the latest assembler optimizations for modern processor architectures. When considering subsequent updates to FFmpeg, consider switching to the git clone rather than the tarball fetched from nasm.us. However, we retain both versions for assembler testing and compatibility, should the master tip version fail to build due to compiler errors and warnings.

**Confirm that all GPUs are working:**

```
nvidia-smi -q | grep Encoder | wc -l


```

This should return the number of GPUs present , and in the case of the dual Tesla M60s, based on the [GM204GL](https://www.nvidia.com/object/tesla-m60.html) SKUs, expect the number to be 4 on a dual-GPU system as each card has a single NVENC chip per graphics processor.

Note that on newer platforms (such as the Nvidia Pascal P1000), the number of NVENC chips per GPU may vary, and may be up to 3 per GPU, totalling to six per Tesla board. See the [GPU support matrix](https://developer.nvidia.com/video-encode-decode-gpu-support-matrix) for more information.

If `~/bin` is already in your path, you can call up ffmpeg directly. Note that the build instructions assume that the NVIDIA CUDA toolkit is on the system path, as is recommended during setup.

**Hint:** Use [this](https://gist.github.com/Brainiarc7/2afac8aea75f4e01d7670bc2ff1afad1) guide to learn how to launch ffmpeg in multiple instances for faster NVENC based encoding on capable hardware.
