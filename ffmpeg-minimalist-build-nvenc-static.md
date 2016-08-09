Minimalist static FFmpeg build on Ubuntu 16.04 with Nvidia NVENC enabled.
------------------------------------------------------------------------

Original guide with a standard build is [here](https://gist.github.com/Brainiarc7/95c9338a737aa36d9bb2931bed379219).

First, prepare for the build and create the workspace directory: 

    cd ~/
    mkdir ~/ffmpeg_sources
    sudo apt-get -y update && apt-get dist-upgrade -y
    sudo apt-get -y install autoconf automake build-essential libass-dev \
      libtool \
      pkg-config texinfo zlib1g-dev

**Install dependencies for NVENC:**

    sudo apt-get -y install glew-utils libglew-dbg libglew-dev libglew1.13 \
    libglewmx-dev libglewmx-dbg freeglut3 freeglut3-dev freeglut3-dbg libghc-glut-dev \
    libghc-glut-doc libghc-glut-prof libalut-dev libxmu-dev libxmu-headers libxmu6 \
    libxmu6-dbg libxmuu-dev libxmuu1 libxmuu1-dbg --install-recommends --install-suggests -y

    

**Build and deploy Yasm:**
[Yasm](http://yasm.tortall.net/) is an assembler for x86 optimizations used by x264 and FFmpeg. Highly recommended or your resulting build may be very slow.

    cd ~/ffmpeg_sources
    wget http://www.tortall.net/projects/yasm/releases/yasm-1.3.0.tar.gz
    tar xzvf yasm-1.3.0.tar.gz
    cd yasm-1.3.0
    ./configure --prefix="$HOME/ffmpeg_build" --bindir="$HOME/bin"
    make -j$(nproc)
    make -j$(nproc) install
    make -j$(nproc) distclean


**Build and deploy libx264 statically:**
This library provides a H.264 video encoder. See the [H.264 Encoding Guide](https://trac.ffmpeg.org/wiki/Encode/H.264) for more information and usage examples.
This requires ffmpeg to be configured with *--enable-gpl* *--enable-libx264*.

    cd ~/ffmpeg_sources
    wget http://download.videolan.org/pub/x264/snapshots/last_x264.tar.bz2
    tar xjvf last_x264.tar.bz2
    cd x264-snapshot*
    PATH="$HOME/bin:$PATH" ./configure --prefix="$HOME/ffmpeg_build" --bindir="$HOME/bin" --enable-static --disable-opencl
    PATH="$HOME/bin:$PATH" make -j88
    make -j$(nproc) install
    make -j$(nproc) distclean


**Build and configure libx265:**
This library provides a H.265/HEVC video encoder. See the [H.265 Encoding Guide](https://trac.ffmpeg.org/wiki/Encode/H.265) for more information and usage examples.

    sudo apt-get install cmake mercurial
    cd ~/ffmpeg_sources
    hg clone https://bitbucket.org/multicoreware/x265
    cd ~/ffmpeg_sources/x265/build/linux
    PATH="$HOME/bin:$PATH" cmake -G "Unix Makefiles" -DCMAKE_INSTALL_PREFIX="$HOME/ffmpeg_build" -DENABLE_SHARED:bool=off ../../source
    make -j$(nproc)
    make -j$(nproc) install
    make -j$(nproc) distclean

**Build and deploy the libfdk-aac library:**
This provides an AAC audio encoder. See the [AAC Audio Encoding Guide](https://trac.ffmpeg.org/wiki/Encode/AAC) for more information and usage examples.
This requires ffmpeg to be configured with *--enable-libfdk-aac* (and *--enable-nonfree* if you also included *--enable-gpl*).

    cd ~/ffmpeg_sources
    wget -O fdk-aac.tar.gz https://github.com/mstorsjo/fdk-aac/tarball/master
    tar xzvf fdk-aac.tar.gz
    cd mstorsjo-fdk-aac*
    autoreconf -fiv
    ./configure --prefix="$HOME/ffmpeg_build" --disable-shared
    make -j$(nproc)
    make -j$(nproc) install
    make -j$(nproc) distclean


**Deploy NVENC SDK:**

First, install Nvidia's drivers:

    sudo apt-get install nvidia-361-updates nvidia-361-updates-dev opencl-headers \
    nvidia-cuda-toolkit --install-recommends -y 

At this stage, please reboot the node.

    sudo systemctl reboot

Then proceed to download the Nvidia NVENC 6.0 SDK from the Nvidia Developer portal when the host is booted up:

    wget -c -v -nc https://developer.nvidia.com/video-sdk-601 -O video-sdk-601.zip

Then extract with zip and navigate to the extracted directory:

    unzip video-sdk-601.zip

Extract and copy the *NVENC SDK headers* as needed:

    cd nvidia_video_sdk_6.0.1/Samples

From within the SDK directory, do:

    sudo cp -vr common/inc/GL/* /usr/include/GL/
    sudo cp -vr common/inc/*.h /usr/include/



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
      --enable-gpl \
      --enable-libass \
      --enable-libfdk-aac \
      --enable-libx264 \
      --enable-libx265 \
      --enable-nvenc \
      --enable-nonfree
    PATH="$HOME/bin:$PATH" make -j$(nproc)
    make -j$(nproc) install
    make -j$(nproc) distclean
    hash -r

If ~/bin is already in your path, you can call up ffmpeg directly.



