# Purpose of this software
This software is meant to be flashed on an esp8266, connected with an RGB LED ws2812b adressable led strip. It uses i2s to control leds. You have to connect your led strip data pin to RX on the esp8266. Otherwise it won't work.
## Homekit integration
No bridge is required. Instead, this project uses the esp-homekit implementation found
here: https://github.com/maximkulkin/esp-homekit to directly connect with homekit
controller devices.

# Installation
Here is build setup that I use. Note that you need Docker, python2.7 and python pip tool installed.

Create an empty directory and change into it.
Create a file esp-sdk-dockerfile with following content:
```dockerfile
FROM ubuntu:20.04 as builder

RUN groupadd -g 1000 docker && useradd docker -u 1000 -g 1000 -s /bin/bash -d /build
RUN mkdir /build && chown docker:docker /build

ENV DEBIAN_FRONTEND=noninteractive

RUN apt update && apt-get install -y \
  make unrar-free autoconf automake libtool gcc g++ gperf \
  flex bison texinfo gawk ncurses-dev libexpat-dev python3-dev python3 python3-serial \
  sed git unzip bash help2man wget bzip2 libtool-bin

# Manually download some dependent packages as builtin URLs are not working
RUN su docker -c " \
    wget -P /build https://libisl.sourceforge.io/isl-0.14.tar.bz2 ; \
    wget -P /build https://github.com/libexpat/libexpat/releases/download/R_2_1_0/expat-2.1.0.tar.gz ; \
"
# Checkout main source code
RUN su docker -c " \
    git clone --recursive https://github.com/pfalcon/esp-open-sdk.git /build/esp-open-sdk ; \
"
COPY crosstool-NG-configure.patch /build
# Patch source code to make it work on newer Linux version
RUN su docker -c " \
    cd /build/esp-open-sdk ; \
    patch -p1 < /build/crosstool-NG-configure.patch ; \
    echo CT_LOCAL_TARBALLS_DIR=/build >> crosstool-config-overrides ; \
    echo CT_DEBUG_gdb=n >> crosstool-config-overrides ; \
    cd esptool ; \
    git checkout v1.3 ; \
"

# Build code
RUN su docker -c "cd /build/esp-open-sdk && make STANDALONE=n"


FROM ubuntu:20.04

RUN DEBIAN_FRONTEND=noninteractive apt update && \
    DEBIAN_FRONTEND=noninteractive apt install -y make python3 python3-serial

COPY --from=builder /build/esp-open-sdk/xtensa-lx106-elf /opt/xtensa-lx106-elf
ENV PATH /opt/xtensa-lx106-elf/bin:$PATH
````

Create a file crosstool-NG-configure.patch with following content:

```git
diff --git a/crosstool-NG/configure.ac b/crosstool-NG/configure.ac
index 5d512fe8..bf9c30f3 100644
--- a/crosstool-NG/configure.ac
+++ b/crosstool-NG/configure.ac
@@ -190,7 +190,7 @@ AC_CACHE_VAL([ac_cv_path__BASH],
 AC_CACHE_CHECK([for bash >= 3.1], [ac_cv_path__BASH],
     [AC_PATH_PROGS_FEATURE_CHECK([_BASH], [bash],
         [[_BASH_ver=$($ac_path__BASH --version 2>&1 \
-                     |$EGREP '^GNU bash, version (3\.[1-9]|4)')
+                     |$EGREP '^GNU bash, version (3\.[1-9]|[4-9])')
           test -n "$_BASH_ver" && ac_cv_path__BASH=$ac_path__BASH ac_path__BASH_found=:]],
         [AC_MSG_RESULT([no])
          AC_MSG_ERROR([could not find bash >= 3.1])])])
````

Create a file esp-rtos-dockerfile with following content:

```dockerfile
FROM ubuntu:20.04 as builder

RUN apt-get update && apt-get install -y git

RUN git clone --recursive https://github.com/Superhouse/esp-open-rtos.git /opt/esp-open-rtos


FROM esp-sdk:latest

RUN apt-get update && apt-get install -y python3 python-is-python3

COPY --from=builder /opt/esp-open-rtos /opt/esp-open-rtos

ENV SDK_PATH /opt/esp-open-rtos
````

Build esp-sdk Docker container:
```bash
docker build . -f esp-sdk-dockerfile -t esp-sdk
```
Build esp-rtos Docker container:
```bash
docker build . -f esp-rtos-dockerfile -t esp-rtos
```
Clone esp-open-rtos repository:
```bash
git clone --recursive https://github.com/SuperHouse/esp-open-rtos.git
```
Install esptool.py:
```bash
pip install esptool
```
Clone esp-homekit-demo repository:
```bash
git clone --recursive https://github.com/maximkulkin/esp-homekit-demo.git
```
Setup enviroment variables:
```bash
export SDK_PATH="$(pwd)/esp-open-rtos"
export ESPPORT=/dev/tty.SLAB_USBtoUART
```
To find out what is the name of your USB device to put to ESPPORT environment variable, first do ls /dev/tty.* before you connect your ESP8266 to USB, then do same command after you have connected it to USB and notice which new device has appeared.

Copy wifi.h.sample -> wifi.h and edit it with correct WiFi SSID and password.
Configure settings
To build an example", first change into esp-homekit-demo directory (into it's root directory:
```bash
cd esp-homekit-demo
```
Then clone this repository inside the example folder :
```bash
cd examples/
git clone https://github.com/EG-Julien/esp8266-led-strip-ws2812b.git
cd ..
```
Then build the firmware you want by running
```bash
docker run -it --rm -v "$(pwd)":/project -w /project esp-rtos make -C examples/esp8266-led-strip-ws2812b all
```
Then flash it (and optionally immediately run monitor)
```bash
make -C examples/esp8266-led-strip-ws2812b flash monitor
```

NOTE: personally I do a lot of stuff in Docker containers, so I have following helper function in my ~/.bashrc:

```bash
docker-run() {
  docker run -it --rm -v "$(pwd)":/project -w /project "$@"
}
```
Then, to run a container I just do
```bash
docker-run esp-rtos make -C examples/esp8266-led-strip-ws2812b all
```

12. Open Home app on your device, and click '+'. If Home does not recognize the ESP8266,
proceed to adding device anyways by entering code.

# Troubleshooting
Try to monitor the ESP8266 while it's executing the software:
```shell
    cd esp-homekit-demo/examples/esp8266-led-strip-ws2812b
    make -C . monitor
```
It is often also a good idea to reset the ESP (cutoff power).
