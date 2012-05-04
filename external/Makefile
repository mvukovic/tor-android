# Please install the following prerequisites (instructions for each follows):
# 	Android OS SDK: http://source.android.com/download
#
# Install and prepare the Android OS SDK ( http://source.android.com/download )
# on Debian or Ubuntu

### these modify the calling shell
# point pkg-config to the .pc files generated from these builds
export PKG_CONFIG_PATH=$(LOCAL)/lib/pkgconfig
# workaround for cross-compiling bug in autoconf
export ac_cv_func_malloc_0_nonnull=yes
export ac_cv_func_setpgrp_void=yes

CWD = $(shell pwd)
PROJECT_ROOT = $(CWD)/..
EXTERNAL_ROOT = $(CWD)

# Android NDK setup:
NDK_BASE ?= /usr/local/android-ndk
NDK_PLATFORM_LEVEL ?= 8
NDK_SYSROOT=$(NDK_BASE)/platforms/android-$(NDK_PLATFORM_LEVEL)/arch-arm
NDK_UNAME=`uname -s | tr '[A-Z]' '[a-z]'`
NDK_TOOLCHAIN=$(NDK_BASE)/toolchains/arm-linux-androideabi-4.4.3/prebuilt/$(NDK_UNAME)-x86

# to use the real HOST tag, you need the latest libtool files:
# http://stackoverflow.com/questions/4594736/configure-does-not-recognize-androideabi
#HOST := arm-none-linux-gnueabi
HOST := arm-linux-androideabi

# install root for built files
DESTDIR = $(EXTERNAL_ROOT)
# TODO try adding the Android-style /data/app.name here
prefix = 
LOCAL := $(DESTDIR)$(prefix)

export PATH := ${PATH}:$(NDK_TOOLCHAIN)/bin:$(LOCAL)/bin

CC := $(NDK_TOOLCHAIN)/bin/arm-linux-androideabi-gcc --sysroot=$(NDK_SYSROOT)
CXX := $(NDK_TOOLCHAIN)/bin/arm-linux-androideabi-g++
CPP := $(NDK_TOOLCHAIN)/bin/arm-linux-androideabi-cpp
LD := $(NDK_TOOLCHAIN)/bin/arm-linux-androideabi-ld
AR := $(NDK_TOOLCHAIN)/bin/arm-linux-androideabi-ar
RANLIB := $(NDK_TOOLCHAIN)/bin/arm-linux-androideabi-ranlib
STRIP := $(NDK_TOOLCHAIN)/bin/arm-linux-androideabi-strip \
	--strip-unneeded -R .note -R .comment

CFLAGS = -DANDROID -I$(LOCAL)/include -I$(NDK_SYSROOT)/usr/include
LDFLAGS = -L$(LOCAL)/lib -L$(NDK_SYSROOT)/usr/lib

# build as small as possible, mostly useful for static binaries
CFLAGS += -fdata-sections -ffunction-sections -Os
LDFLAGS += -Wl,--gc-sections

all: assets

#------------------------------------------------------------------------------#
# openssl-static

openssl-static/obj/local/armeabi/libcrypto.a:
	cd openssl-static && \
		ndk-build

openssl-static/obj/local/armeabi/libssl.a:
	cp config.sub openssl-static
	cp config.guess openssl-static
	cd openssl-static && \
		ndk-build

openssl-static-build-stamp: openssl-static/obj/local/armeabi/libcrypto.a openssl-static/obj/local/armeabi/libssl.a
	touch openssl-static-build-stamp

openssl-static: openssl-static-build-stamp
	test -d lib || mkdir lib
	test -d include || mkdir include
	cp openssl-static/obj/local/armeabi/libcrypto.a lib
	cp openssl-static/obj/local/armeabi/libssl.a lib
	cp -R openssl-static/include/openssl include 

openssl-static-clean:
	-cd openssl-static && \
		ndk-build clean
	-rm openssl-static-build-stamp
	-rm lib/libcrypto.a
	-rm lib/libssl.a
	-cd openssl-static && \
		git clean -fdx

#------------------------------------------------------------------------------#
# libevent

libevent/Makefile:
	cd libevent && ./autogen.sh
	cp config.sub libevent
	cp config.guess libevent
	cd libevent && \
 		CC="$(CC)" AR="$(AR)" RANLIB=$(RANLIB) CFLAGS="$(CFLAGS)" LDFLAGS="$(LDFLAGS)" \
			./configure \
				--host=$(HOST) \
				--disable-shared

libevent-build-stamp: libevent/Makefile
	$(MAKE) -C libevent
	touch libevent-build-stamp

libevent: libevent-build-stamp
	test -d lib || mkdir lib
	test -d include || mkdir include
	cp libevent/.libs/libevent.a lib
	cp -R libevent/include/* include

libevent-clean:
	-rm -f include/*.h
	-rm -f lib/libevent.a
	-rm -f libevent-build-stamp
	-cd libevent && \
		git clean -fdx

#------------------------------------------------------------------------------#
# tor

tor/configure: 
	cd tor && \
	 ./autogen.sh

tor/Makefile: tor/configure
	cp config.sub tor
	cp config.guess tor
	cd tor && \
                CC="$(CC)" AR="$(AR)" RANLIB=$(RANLIB) CFLAGS="$(CFLAGS)" LDFLAGS="$(LDFLAGS)" \
                        ./configure \
                                --host=$(HOST) \
                                --prefix=$(NDK_TOOLCHAIN) \
				--disable-asciidoc \
				--with-libevent-dir=$(EXTERNAL_ROOT) --enable-static-libevent \
				--with-openssl-dir=$(EXTERNAL_ROOT) --enable-static-openssl

tor-build-stamp: tor/Makefile
	$(MAKE) -C tor 
	touch tor-build-stamp

tor: libevent openssl-static tor-build-stamp
	test -d bin || mkdir bin
	cp tor/src/or/tor bin

tor-clean:
	-rm -f bin/tor
	-rm -f tor-build-stamp
	-cd tor && \
		git clean -fdx


#------------------------------------------------------------------------------#
# obfsproxy


obfsproxy/Makefile: 
	cd obfsproxy && \
	 ./autogen.sh
	cp config.sub obfsproxy
	cp config.guess obfsproxy
	cd obfsproxy && \
                CC="$(CC)" AR="$(AR)" RANLIB=$(RANLIB) CFLAGS="$(CFLAGS) -D_FORTIFY_SOURCE=2 -fPIE -fwrapv -fno-strict-aliasing -fno-strict-overflow" LDFLAGS="$(LDFLAGS)" \
			libevent_LIBS="-L$(EXTERNAL_ROOT)/lib -levent" libevent_CFLAGS="-I$(EXTERNAL_ROOT)/include" \
			libcrypto_LIBS="-L$(EXTERNAL_ROOT)/lib -lcrypto" libcrypto_CFLAGS="-I$(EXTERNAL_ROOT)/include" \
                        ./configure \
                                --host=$(HOST) \
                                --prefix=$(NDK_TOOLCHAIN) \

obfsproxy-build-stamp: obfsproxy/Makefile
	$(MAKE) -C obfsproxy
	touch obfsproxy-build-stamp

obfsproxy: openssl-static libevent obfsproxy-build-stamp
	test -d bin || mkdir bin
	cp obfsproxy/obfsproxy bin

obfsproxy-clean:
	-rm -f bin/obfsproxy
	-rm -f obfsproxy-build-stamp
	-cd obfsproxy && \
		git clean -fdx

#------------------------------------------------------------------------------#
# privoxy

privoxy/configure: 
	cd privoxy && \
	 ./autoheader
	 ./autoconf

privoxy/Makefile: privoxy/configure
	cp config.sub privoxy
	cp config.guess privoxy	
	cd privoxy && \
                CC="$(CC)" AR="$(AR)" RANLIB=$(RANLIB) CFLAGS="$(CFLAGS)" LDFLAGS="$(LDFLAGS)" \
                        ./configure \
                                --host=arm-linux-eabi --disable-pthread 

privoxy-build-stamp: privoxy/Makefile
	$(MAKE) -C privoxy
	touch privoxy-build-stamp

privoxy: privoxy-build-stamp
	test -d bin || mkdir bin
	cp privoxy/privoxy bin

privoxy-clean:
	$(MAKE) -C privoxy clean
	-rm -f bin/privoxy
	-rm -f privoxy-build-stamp
	-rm -f privoxy/config.status
	-rm -f privoxy/config.log
	-rm -f privoxy/Makefile


#------------------------------------------------------------------------------#
# JTorControl library

jtorctrl:
	test -d jtorctl/bin || mkdir jtorctl/bin
	cd jtorctl && \
		javac net/freehaven/tor/control/TorControlConnection.java -d bin
	cd jtorctl/bin && \
		jar cvf jtorctrl.jar *
	cp jtorctl/bin/jtorctrl.jar bin

jtorctrl-clean:
	-rm bin/jtorctrl.jar

#------------------------------------------------------------------------------#
# cleanup, cleanup, put the toys away
assets: tor privoxy jtorctrl
	install -d ../res/raw
	install -d ../libs
	install bin/tor ../res/raw	
	install bin/privoxy ../res/raw	
	install bin/jtorctrl.jar ../libs

assets-clean:
	-rm ../res/raw/tor
	-rm ../res/raw/privoxy
	-rm ../libs/jtorctrl.jar	

#------------------------------------------------------------------------------#
# cleanup, cleanup, put the toys away

clean: openssl-static-clean libevent-clean tor-clean privoxy-clean jtorctrl-clean assets-clean