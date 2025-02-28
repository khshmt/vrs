# What is VRS?

VRS is a **file format** optimized to record & playback streams of sensor data,
such as images, audio samples, and any other discrete sensors (IMU, temperature,
etc), stored in per-device streams of time-stamped records.

VRS was first created to record images and sensor data from early prototypes of
the Quest device, to develop the device’s positional tracking system now known
as [Insight](https://ai.facebook.com/blog/powered-by-ai-oculus-insight/), and
Quest's hand tracking software. It is also the file format used by the
[Aria glasses](https://about.facebook.com/realitylabs/projectaria/).

## Main features

- VRS files contain multiple streams of time-sorted records generated by a set
  of sensors(camera, IMU, thermometer, GPS, etc), typically one set of sensors
  per stream.
- The file and each stream contain an independent set of tags, which are string
  name/value pairs that describe them.
- Streams may contain `Configuration`, `State` and `Data` records, each with a
  timestamp in a common time domain for the whole file.\
  Typically, streams contain with one `Configuration` and one `State` record, followed
  one to millions of `Data` records.
- Records are structured as a succession of typed content blocks.\
  Typical content blocks are metadata, image, audio and custom content blocks.
- Metadata content blocks contain raw sensor data described once per stream,
  making the file format very efficient. The marginal cost of adding 1 byte of
  data to each metadata content block of a stream is 1 byte per record (or less,
  when lossless compression happens).
- Records can be losslessly compressed using lz4 or zstd, which can be fast
  enough to compress while recording on device.
- Multiple threads can create records concurrently for the same file, without
  CPU contention.
- VRS supports huge file size (tested with multi terabytes use cases).
- VRS supports chunked files: auto-chunking on creation, automated chunk
  detection for playback.
- Playback is optimized for timestamp order, which is key for network streaming.
- Random-access playback is supported.
- Custom `FileHandler` implementations can add support for cloud storage
  streaming.

# Documentation

[The VRS documentation](https://facebookresearch.github.io/vrs/) explains how
VRS works. It is complemented by the
[API documentation](https://facebookresearch.github.io/vrs/doxygen/index.html),
while the sample code and the sample apps below demonstrate in code how to use
the API.

We plan on having a VRS Users group dedicated on discussing VRS usage. Stay
tuned for details.

# Getting Started

To work with VRS files, the vrs open source project provides a C++ library with
external open source dependencies such as
[boost](https://github.com/boostorg/boost),
[fmt](https://github.com/fmtlib/fmt), [lz4](https://github.com/lz4/lz4),
[zstd](https://github.com/facebook/zstd),
[xxhash](https://github.com/Cyan4973/xxHash), and
[googletest](https://github.com/google/googletest) for unit tests. To build &
run VRS, you’ll need a C++17 compiler, such as a recent enough version of clang
or Visual Studio.

The simplest way to build VRS is to install the libraries on your system using
some package system, such as [Brew](https://brew.sh/) on macOS, or
[apt](<https://en.wikipedia.org/wiki/APT_(software)>) on Ubuntu, and then use
cmake to build & test. VRS supports many other platforms such as Windows,
Android, iOS and other flavors of Linux, but we currently only provide
instructions for macOS and Ubuntu. You can also build VRS in a container and
avoid installing any library on your system.

## Instructions (macOS and Ubuntu and container)

### Install build tools & libraries (macOS)

- install Brew, following the instruction on
  [Brew’s web site](https://brew.sh/).
- install tools & libraries:
  ```
  brew install cmake git ninja ccache boost fmt libpng jpeg-turbo lz4 zstd xxhash glog googletest
  brew install qt5 portaudio pybind11
  brew install node doxygen
  ```

### Install build tools & libraries (Ubuntu)

_These instructions are validated using Ubuntu 20.04, whereas Ubuntu 18.04
doesn't install recent enough versions of cmake, fmt, lz4, and zstd, and is
therefore not supported._

- install tools & libraries:
  ```
  sudo apt-get install cmake git ninja-build ccache libgtest-dev libfmt-dev libturbojpeg-dev libpng-dev
  sudo apt-get install liblz4-dev libzstd-dev libxxhash-dev
  sudo apt-get install libboost-system-dev libboost-filesystem-dev libboost-thread-dev libboost-chrono-dev libboost-date-time-dev
  sudo apt-get install qtbase5-dev portaudio19-dev
  sudo apt-get install npm doxygen
  ```

## Build & run (macOS & Linux)

- Run cmake:

```
cmake -S <path_to_vrs_folder> -B <path_to_build_folder> -G Ninja
```

If you want to build vrsplayer, you need to specify where your installation of
Qt is. Where Qt is depends on how you've installed it, using a package manager
such as Brew or APT, or downloading it directly from
[Qt's official website](https://www.qt.io/download).

To tell cmake where to find Qt, you can either add
`-DCMAKE_PREFIX_PATH=<path_to_qt>` to the cmake command above, or set the
environment variable `QT_DIR=<path_to_qt>` to point to your Qt installation
(same path). As a sanity check, you should be able to find the qmake tool at
`<path_to_qt>/bin/qmake`.

Note: We ran into strange build issues when Qt5 and Qt6 were both installed at
the same time via Brew on macOS, but uninstalling either fixed the problem.

### Qt 5 or Qt 6

At this time, vrsplayer is mostly tested using Qt 5.15.3 LTS, but the code has
been updated to build and run with Qt 6.3.0. However, testing with Qt 6 was
pretty superficial.

- Build everything & run tests:

```
cd <path_to_build_folder>
ninja all
ctest -j8
```

- To include VRS in your cmake project:

```
cd <path_to_build_folder>
ninja install # install VRS on your system as a library cmake can find
```

In your cmake project (probably one your project's `CMakeLists.txt` files):

```
find_package(vrslib REQUIRED) # find the vrs package, break if not found

add_executable(your_app your_app.cpp) # that's your app
target_link_libraries(your_app vrs::vrslib) # so your app can use the vrs includes and libraries
```

You can then use VRS in your `your_app.cpp` code:

```
#include <vrs/RecordFileReader.h>

int main() {
  vrs::RecordFileReader reader;
  if (reader.openFile("myfile.vrs") == 0) {
    do something...
  }
  return 0;
}

```

## Windows Support

We don’t have equivalent instructions for Windows.
[vcpkg](https://vcpkg.io/en/index.html) looks like a promising package manager
for Windows, but the cmake build system needs more massaging to work.\
Contributions welcome! :-)

## Container build & Usage

- Build VRS in a container and use it on your local data:

```
cd <path_to_vrs_folder>
podman/docker build . -t vrs
podman/docker run -it --volume <your_local_data>:/data vrs:latest
```

# Sample Code

- [The sample code](./sample_code) demonstrates the basics to record and read a
  VRS file, then how to work with `RecordFormat` and `DataLayout` definitions.
  The code is extensively documented, functional, compiles but isn’t meant to be
  run.
  - [SampleRecordAndPlay.cpp](./sample_code/SampleRecordAndPlay.cpp)\
     Demonstrates different ways to create VRS files, and how to read them, but
    the format of the records is deliberately trivial, as it is not the focus of
    this code.
  - [SampleImageReader.cpp](./sample_code/SampleImageReader.cpp)\
     Demonstrates how to read records containing images.
  - [SampleRecordFormatDataLayout.cpp](./sample_code/SampleRecordFormatDataLayout.cpp)\
     Demonstrates how to use `RecordFormat` and `DataLayout` in more details.
- [The sample apps](./sample_apps) are fully functional apps demonstrate how to
  create, then read, a VRS file with 3 types of streams.
  - a metadata stream
  - a stream with metadata and uncompressed images
  - a stream with audio images

# Tools

The `vrs` command line tool allows you to inspect and extract data out of VRS
files. It can create new VRS files by generating a modified copy of an original
recording. It can also extract images, dump metadata for human or computer
consumption (json). The vrs command line tool can be found in the cmake build
folder at `oss/tools/vrs/vrs`. It has many options,
[documented here](https://facebookresearch.github.io/vrs/docs/VrsCliTool).

`vrsplayer` is a GUI tool which can "play" VRS files as multi-stream video files
and audio files. It also provides ways to visualize record's metadata as they
are played. It can be found in the cmake build folder at
`oss/tools/vrsplayer/vrsplayer`. For more information, see the
[`vrsplayer` documentation](https://facebookresearch.github.io/vrs/docs/vrsplayer).

# Python interface

[`pyvrs`](https://github.com/facebookresearch/pyvrs) is a python interface for
VRS library. You can install it via pip

```
pip install vrs
```

# Contributing

We welcome contributions! See [CONTRIBUTING](CONTRIBUTING.md) for details on how
to get started, and our [code of conduct](CODE_OF_CONDUCT.md).

# License

VRS is released under the [Apache 2.0 license](LICENSE).

vrsplayer requires an installation of Qt 5.15+ or Qt 6.3+ on your system, maybe
using Brew (macOS) or APT (Linux), as demonstrated above, or using an official
distribution from [Qt's official website](https://www.qt.io/download). If found,
vrsplayer will be built and will link dynamically against the LGPL v3 Qt
libraries at runtime.

We provide no pre-built binaries, so you must build vrsplayer from source to use
it.
