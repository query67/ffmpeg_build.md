Minimalist static FFmpeg build on Ubuntu 16.04 with Nvidia NVENC enabled.
------------------------------------------------------------------------

Original guide with a standard build is [here](https://gist.github.com/Brainiarc7/95c9338a737aa36d9bb2931bed379219).

With this guide, I'm adding more instructions to enable support for [NVIDIA CUVID](http://docs.nvidia.com/cuda/video-decoder/) and [NVIDIA NPP](https://developer.nvidia.com/npp) for enhanced encode and decode performance.

First, prepare for the build and create the work space directory: 

    cd ~/
    mkdir ~/ffmpeg_sources
    sudo apt-get -y update && apt-get dist-upgrade -y
    sudo apt-get -y install autoconf automake build-essential libass-dev \
      libtool \
      pkg-config texinfo zlib1g-dev cmake mercurial
      
      
**Install CUDA 9.1 SDK from Nvidia's repository:**
 
 Note that this phase will prompt you to install the device driver. Skip it, and skip the samples too.We will install the driver later. Fetch the repository installers first:

    sudo dpkg -i cuda-repo-ubuntu1604_9.1.85-1_amd64.deb
    
    sudo apt-key adv --fetch-keys http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/7fa2af80.pub
    
    sudo apt-get update
    
    sudo apt-get install cuda

Now, set up the environment variables for CUDA:

Edit the `/etc/environment` file and append the following:

    CUDA_HOME=/usr/local/cuda
    
Now, append the PATH variable with the following:

    /usr/local/cuda/bin

When done, remember to source the file:

    source /etc/environment


   **Build FFmpeg's dependency chain:** 

**Build and deploy nasm:**
[Nasm](http://www.nasm.us/) is an assembler for x86 optimizations used by x264 and FFmpeg. Highly recommended or your resulting build may be very slow. Note that we're using the latest release candidate, and not the stable version as of the time of writing.

    cd ~/ffmpeg_sources
    wget http://www.nasm.us/pub/nasm/releasebuilds/2.14rc0/nasm-2.14rc0.tar.gz
    tar xzvf nasm-2.14rc0.tar.gz
    cd nasm-2.14rc0
    ./configure --prefix="$HOME/ffmpeg_build" --bindir="$HOME/bin"
    make -j$(nproc) VERBOSE=1
    make -j$(nproc) install
    make -j$(nproc) distclean


**Build and deploy libx264 statically:**
This library provides a H.264 video encoder. See the [H.264 Encoding Guide](https://trac.ffmpeg.org/wiki/Encode/H.264) for more information and usage examples.
This requires ffmpeg to be configured with *--enable-gpl* *--enable-libx264*.

    cd ~/ffmpeg_sources
    git clone http://git.videolan.org/git/x264.git -b stable
    cd x264/
    PATH="$HOME/bin:$PATH" ./configure --prefix="$HOME/ffmpeg_build" --bindir="$HOME/bin" --enable-static --disable-opencl
    PATH="$HOME/bin:$PATH" make -j$(nproc) VERBOSE=1
    make -j$(nproc) install
    make -j$(nproc) distclean


**Build and configure libx265:**
This library provides a H.265/HEVC video encoder. See the [H.265 Encoding Guide](https://trac.ffmpeg.org/wiki/Encode/H.265) for more information and usage examples.

    cd ~/ffmpeg_sources
    hg clone https://bitbucket.org/multicoreware/x265
    cd ~/ffmpeg_sources/x265/build/linux
    PATH="$HOME/bin:$PATH" cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX="$HOME/ffmpeg_build" -DENABLE_SHARED:bool=off ../../source
    make -j$(nproc) VERBOSE=1
    make -j$(nproc) install
    make -j$(nproc) clean

**Build and deploy the libfdk-aac library:**
This provides an AAC audio encoder. See the [AAC Audio Encoding Guide](https://trac.ffmpeg.org/wiki/Encode/AAC) for more information and usage examples.
This requires ffmpeg to be configured with *--enable-libfdk-aac* (and *--enable-nonfree* if you also included *--enable-gpl*).

    cd ~/ffmpeg_sources
    wget -O fdk-aac.tar.gz https://github.com/mstorsjo/fdk-aac/tarball/master
    tar xzvf fdk-aac.tar.gz
    cd mstorsjo-fdk-aac*
    autoreconf -fiv
    ./configure --prefix="$HOME/ffmpeg_build" --disable-shared
    make -j$(nproc) VERBOSE=1
    make -j$(nproc) install
    make -j$(nproc) distclean


**Deploy NVENC SDK:**
At this phase, we will assume that the end user has rebooted the node prior to proceeding with this phase.
If not, proceed via:


    sudo systemctl reboot

Then proceed to download the [latest Nvidia NVENC SDK](https://developer.nvidia.com/nvidia-video-codec-sdk) from the Nvidia Developer portal when the host is booted up:

We are using the latest NVENC SDK for this.

Ensure that the SDK is downloaded to your `~/ffmpeg_sources` directory (`cd ~/ffmpeg_sources` to be sure) so as to  maintain the needed directory structure.

Extract and copy the NVENC SDK headers as needed:

Then navigate to the extracted directory:

    unzip Video_Codec_SDK_*.zip
    cd Video_Codec_SDK_*/Samples

From within the SDK directory, do:

    sudo cp -vr Samples/common/inc/GL/* /usr/include/GL/
    sudo cp -vr Samples/common/inc/*.h /usr/include/

When done,do:

    cd ~/ffmpeg_sources
    mv Video_Codec_SDK_* nv_sdk

That will allow us to statically link to the SDK with ease, below.

Note that there may be a newer version of the SDK available at the time, please adjust as appropriate.

Take note that [changes to the inclusion of third party headers](https://git.videolan.org/?p=ffmpeg/nv-codec-headers.git) affects new builds, and this is fixed by:

```
git clone https://git.videolan.org/git/ffmpeg/nv-codec-headers.git
cd nv-codec-headers
make
sudo make install

```

Proceed as usual:


**Building a static ffmpeg binary with the required options:**

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
      --extra-cflags=-I../nv_sdk \
      --extra-ldflags=-L../nv_sdk \
      --extra-cflags="-I/usr/local/cuda/include/" \
      --extra-ldflags=-L/usr/local/cuda/lib64/ \
      --enable-gpl \
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
    
    
You may also want to tune your build further by calling upon NVCC to generate a build optimized for your GPU's CUDA architecture only.

The example below shows the build options to pass for Pascal's GM10x-series GPUs, with an SM version of 6.1:

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
      --extra-cflags=-I../nv_sdk \
      --extra-ldflags=-L../nv_sdk \
      --extra-cflags="-I/usr/local/cuda/include/" \
      --extra-ldflags=-L/usr/local/cuda/lib64/ \
      --nvccflags="-gencode arch=compute_61,code=sm_61 -O2" \
      --enable-gpl \
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
--extra-cflags=-I../nv_sdk \
--extra-ldflags=-L../nv_sdk \
--extra-cflags="-I/usr/local/cuda/include/" \
--extra-ldflags=-L/usr/local/cuda/lib64/ \
--nvccflags="-gencode arch=compute_52,code=sm_52 -O2" \
--enable-gpl \
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

**Handling package upgrades:**

For individual packages availed via git, simply navigate to their source directory and run git pull followed by re-building them:

**(a). For nasm:**

```
git clone git://repo.or.cz/nasm.git
cd nasm
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

**(c).  For x265:**
```
cd ~/ffmpeg_sources/x265
git pull
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
**(e). For FFmpeg:**

**i. For Pascal - based GPU systems (GP10x):**

```
cd ~/ffmpeg_sources/FFmpeg
PATH="$HOME/bin:$PATH" PKG_CONFIG_PATH="$HOME/ffmpeg_build/lib/pkgconfig:/usr/lib/x86_64-linux-gnu/pkgconfig" ./configure \
  --prefix="$HOME/ffmpeg_build" \
  --pkg-config-flags="--shared" \
  --extra-cflags="-I$HOME/ffmpeg_build/include" \
  --extra-ldflags="-L$HOME/ffmpeg_build/lib" \
  --bindir="$HOME/bin" \
  --enable-cuda-sdk \
  --enable-cuvid \
  --enable-libnpp \
  --extra-cflags=-I../nv_sdk \
  --extra-ldflags=-L../nv_sdk \
  --extra-cflags="-I/usr/local/cuda/include/" \
  --extra-ldflags=-L/usr/local/cuda/lib64/ \
  --nvccflags="-gencode arch=compute_61,code=sm_61 -O2" \
  --enable-gpl \
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
PATH="$HOME/bin:$PATH" PKG_CONFIG_PATH="$HOME/ffmpeg_build/lib/pkgconfig:/usr/lib/x86_64-linux-gnu/pkgconfig" ./configure \
--prefix="$HOME/ffmpeg_build" \
--extra-cflags="-I$HOME/ffmpeg_build/include" \
--extra-ldflags="-L$HOME/ffmpeg_build/lib" \
--bindir="$HOME/bin" \
--enable-cuda \
--enable-cuvid \
--enable-libnpp \
--extra-cflags=-I../nv_sdk \
--extra-ldflags=-L../nv_sdk \
--extra-cflags="-I/usr/local/cuda/include/" \
--extra-ldflags=-L/usr/local/cuda/lib64/ \
--nvccflags="-gencode arch=compute_52,code=sm_52 -O2" \
--enable-gpl \
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

**On nasm:**

We build nasm from source, using the git master tip as it contains the latest assembler optimizations for modern processor architectures. When considering subsequent updates to FFmpeg, consider switching to the git clone rather than the tarball fetched from nasm.us. However, we retain both versions for assembler testing and compatibility, should the master tip version fail to build due to compiler errors and warnings.

**Confirm that all GPUs are working:**

```
nvidia-smi -q | grep Encoder | wc -l

```

This should return the number of GPUs present , and in the case of the dual Tesla M60s, based on the [GM204GL](https://www.nvidia.com/object/tesla-m60.html) SKUs, expect the number to be 4 on a dual-GPU system as each card has a single NVENC chip per graphics processor.

Note that on newer platforms (such as the Nvidia Pascal P1000), the number of NVENC chips per GPU may vary, and may be up to 3 per GPU, totalling to six per Tesla board. See the [GPU support matrix](https://developer.nvidia.com/video-encode-decode-gpu-support-matrix) for more information.


If `~/bin` is already in your path, you can call up ffmpeg directly.
Note that the build instructions assume that the NVIDIA CUDA toolkit is on the system path, as is recommended during setup.

**Hint:** Use [this](https://gist.github.com/Brainiarc7/2afac8aea75f4e01d7670bc2ff1afad1) guide to learn how to  launch ffmpeg in multiple instances for faster NVENC based encoding on capable hardware.










