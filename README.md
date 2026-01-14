# multimon-ng (RTL-SDR Fork)

This is a fork of multimon-ng with integrated RTL-SDR support, allowing direct reception from RTL-SDR dongles without needing to pipe from rtl_fm.

multimon-ng is the successor of multimon. It decodes the following digital transmission modes:

- POCSAG512 POCSAG1200 POCSAG2400
- FLEX FLEX_NEXT
- EAS
- UFSK1200 CLIPFSK FMSFSK AFSK1200 AFSK2400 AFSK2400_2 AFSK2400_3
- HAPN4800
- FSK9600
- DTMF
- ZVEI1 ZVEI2 ZVEI3 DZVEI PZVEI
- EEA EIA CCIR
- MORSE_CW
- DUMPCSV X10 SCOPE

## Building

multimon-ng can be built using either qmake or CMake:

#### qmake
```
mkdir build
cd build
qmake ../multimon-ng.pro
make
sudo make install
```

#### CMake
```
mkdir build
cd build
cmake ..
make
sudo make install
```

The installation prefix can be set by passing a 'PREFIX' parameter to qmake. e.g:
```qmake multimon-ng.pro PREFIX=/usr/local```

### Building with RTL-SDR Support

RTL-SDR support is automatically enabled if librtlsdr is detected during the CMake build:

```
mkdir build
cd build
cmake ..
make
sudo make install
```

If librtlsdr is not found, multimon-ng will still build but without RTL-SDR support. To install librtlsdr:

**Debian/Ubuntu:**
```
sudo apt-get install librtlsdr-dev
```

**Arch Linux:**
```
sudo pacman -S rtl-sdr
```

**From source:**
```
git clone https://github.com/osmocom/rtl-sdr.git
cd rtl-sdr
mkdir build && cd build
cmake ..
make
sudo make install
```

### Windows MinGW Builds

#### On Windows (MSYS2/MinGW)
Install MSYS2, then from the MinGW64 or MinGW32 shell:
```
mkdir build
cd build
cmake ..
make
```

#### Cross-compiling from Linux
Install MinGW cross-compiler, then use the provided toolchain files:
```
# For 64-bit Windows
mkdir build-mingw64
cd build-mingw64
cmake .. -DCMAKE_TOOLCHAIN_FILE=../cmake/toolchain-mingw64.cmake
make

# For 32-bit Windows
mkdir build-mingw32
cd build-mingw32
cmake .. -DCMAKE_TOOLCHAIN_FILE=../cmake/toolchain-mingw32.cmake
make
```

### Environments

So far multimon-ng has been successfully built on:

- Arch Linux
- Debian
- Gentoo
- Kali Linux
- Ubuntu
- OS X
- Windows (Qt-MinGW build environment, Cygwin, and VisualStudio/MSVC)
- FreeBSD

## Examples

### Wav to raw

Files can be easily converted into multimon-ng's native raw format using *sox*. e.g:

    sox -R -t wav pocsag_short.wav -esigned-integer -b16 -r 22050 -t raw pocsag_short.raw

GNURadio can also generate the format using the file sink in input mode *short*. 

### Pipe sox to multimon-ng

You can also "pipe" raw samples into multimon-ng using something like:

    sox -R -t wav pocsag_short.wav -esigned-integer -b16 -r 22050 -t raw - | ./multimon-ng -

> [!NOTE]
> Note the trailing dash, means write/read to/from stdin

### Pipe rtl_fm to multimon-ng

As a last example, here is how you can use it in combination with RTL-SDR:

    rtl_fm -f 403600000 -s 22050 | multimon-ng -t raw -a FMSFSK -a AFSK1200 /dev/stdin

### Direct RTL-SDR Input (This Fork)

This fork includes integrated RTL-SDR support, eliminating the need to pipe from rtl_fm:

**APRS on 144.390 MHz:**
```
multimon-ng -a AFSK1200 --rtl-freq 144390000 --rtl-gain 42
```

**POCSAG pager on 403.6 MHz:**
```
multimon-ng -a POCSAG512 -a POCSAG1200 -a POCSAG2400 --rtl-freq 403600000 --rtl-gain 40
```

**With custom sample rate:**
```
multimon-ng -a AFSK1200 --rtl-freq 144390000 --rtl-samp 24000 --rtl-gain 42
```

#### RTL-SDR Command-Line Options

- `--rtl-dev <n>` : RTL-SDR device index or serial (default: 0)
- `--rtl-freq <freq>` : Frequency to tune to in Hz (required for RTL-SDR mode)
- `--rtl-gain <gain>` : Tuner gain in dB (default: auto gain)
- `--rtl-samp <rate>` : Sample rate in Hz (default: 22050)
- `--rtl-ppm <ppm>` : Frequency correction in ppm (default: 0)

The RTL-SDR integration automatically handles:
- FM demodulation with proper oversampling (~1 MHz capture rate)
- Frequency offset tuning to avoid DC spike
- Downsampling to the target audio rate (default 22050 Hz)
- DC blocking filter

### Flac record and parse live data

A more advanced sample that combines `rtl_fm`, `flac`, and `tee` to split the output from `rtl_rm` into separate streams. One stream to be passed to `flac` to record the audio and another stream to for example an application that does text parsing of `mulimon-ng` output



```sh
rtl_fm -s 22050 -f 123.456M -g -9.9 | tee >(flac -8 --endian=little --channels=1 --bps=16 --sample-rate=22050 --sign=signed - -o ~/recordings/rtlfm.$EPOCHSECONDS.flac -f) | multimon-ng -v 0 -a FLEX -a FLEX_NEXT -t raw /dev/stdin
```

1. You can pass `-l` to `rtl_fm` for the squelch level, this will cut the noise floor so less data gets encoded by flac and will significantly reduce the file size but could result in loss of signal data. **This value must be tuned!**
2. Flac uses `-8` here, if you run an a resource constraint device you may want to lower this value
3. The Flac `-o` argument value contains `$EPOCHSECONDS` to make unique files when this gets restarted

To replay the recorded flac file to multimon-ng (requires sox):

```sh
flac -d --stdout ~/recordings/rtlf/rtlfm.1725033204.flac | multimon-ng -r -v 0 -a FLEX_NEXT -t flac -
```

## Packaging

```
qmake multimon-ng.pro PREFIX=/usr/local
make
make install INSTALL_ROOT=/
```

## Testing

After building, run the test suite:

```
./test/run_tests.sh
```

> [!NOTE]
> Testing non-raw sample files (flac, wav) requires [SoX](https://sourceforge.net/projects/sox/) to be installed.
