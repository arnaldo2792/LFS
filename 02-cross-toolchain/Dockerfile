# Phase 0: adds the user builder and updates the
# dnf metadata cache, since other phases will
# install some dependencies
FROM fedora:33 as base

# Add user builder
RUN useradd builder -m
WORKDIR /home/builder

# Phase 2: initial LFS build
FROM base as base-builder

# The arch for the build
ARG ARCH

# Install dependencies
RUN dnf makecache && dnf install -y \
    gawk gcc g++ m4                 \
    make patch python3 texinfo      \
    xz bison diffutils findutils    \
    byacc bzip2 file hostname

# Setup environment
ENV LFS=/mnt/lfs

# Setting LC_ALL to “POSIX” or “C” (the two are equivalent)
# ensures that everything will work as expected in the chroot
# environment.
ENV LC_ALL=POSIX \
# The LFS_TGT variable sets a non-default, but compatible
# machine description for use when building our cross compiler
# and linker and when cross compiling our temporary toolchain.
LFS_TGT=$ARCH-lfs-linux-gnu \
# By putting $LFS/tools/bin ahead of the standard PATH, the cross-compiler
# is picked up by the shell immediately after its installation. 
PATH=$LFS/tools/bin:$PATH \
# If this variable is not set, configure scripts may attempt to load
# configuration items specific to some distributions
CONFIG_SITE=$LFS/usr/share/config.site

# Check all required packages are installed in its own layer
# to prevent rebuilds in the cache
FROM base-builder as test-dependencies

ADD ./version-check version-check
RUN chown builder:builder ./version-check \
  && chmod u+x ./version-check \
  && ./version-check

# Begin initial cross toolchain compilation
FROM base-builder as compiled-cross-toolchain

# CPUs to use for the build
ARG CPUS=8
ENV CPUS=$CPUS

# Create /mnt/lfs, used to create the root filesystem
# and change group's ownership
RUN mkdir -p $LFS/{bin,etc,lib,sbin,usr,var,tools,sources} \
  && case $(uname -m) in \
        x86_64) mkdir -p $LFS/lib64 ;; \
     esac \
  && chown -R builder:builder $LFS && chmod -v a+wt $LFS/sources

# From this point, it is safe to use the builder user
USER builder

# Compile binutils
ENV PACKAGE_VER=binutils-2.36.1
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz
ENV PACKAGE_SOURCE=/$LFS/sources/$PACKAGE_VER

WORKDIR $LFS/sources

# Copy package source
COPY --from=arnaldo2792/lfs-sources /home/builder/sources/$PACKAGE_TAR /$LFS/sources/

# Extract source
RUN tar -xf $PACKAGE_TAR && mkdir $PACKAGE_SOURCE/build

# Change workdir 
WORKDIR $PACKAGE_SOURCE/build

# Configure package
RUN ../configure --prefix=$LFS/tools  \
                  --with-sysroot=$LFS \
                  --target=$LFS_TGT   \
                  --disable-nls       \
                  --disable-werror

# Build package 
RUN make -j$CPUS

# Install package 
RUN make install

# Compile gcc
WORKDIR $LFS/sources

ENV PACKAGE_VER=gcc-10.2.0
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz
ENV PACKAGE_SOURCE=/$LFS/sources/$PACKAGE_VER

ENV MPFR_VER=mpfr-4.1.0
ENV MPFR_TAR=$MPFR_VER.tar.xz
ENV MPFR_SOURCE=/$LFS/sources/$MPFR_VER

ENV GMP_VER=gmp-6.2.1
ENV GMP_TAR=$GMP_VER.tar.xz
ENV GMP_SOURCE=/$LFS/sources/$GMP_VER

ENV MPC_VER=mpc-1.2.1
ENV MPC_TAR=$MPC_VER.tar.gz
ENV MPC_SOURCE=/$LFS/sources/$MPC_VER

# Copy package and dependencies sources
COPY --from=arnaldo2792/lfs-sources /home/builder/sources/$PACKAGE_TAR \
  /home/builder/sources/$MPFR_TAR                      \
  /home/builder/sources/$GMP_TAR                       \
  /home/builder/sources/$MPC_TAR                       \
  /$LFS/sources/

# Extract source
RUN tar -xf $PACKAGE_TAR && mkdir $PACKAGE_SOURCE/build
RUN tar -xf $MPFR_TAR && mv $MPFR_VER $PACKAGE_SOURCE/mpfr
RUN tar -xf $GMP_TAR && mv $GMP_VER $PACKAGE_SOURCE/gmp
RUN tar -xf $MPC_TAR && mv $MPC_VER $PACKAGE_SOURCE/mpc

# On x86_64 hosts, set the default directory name for 64-bit
# libraries to “lib”
RUN cd $PACKAGE_SOURCE \
  && case $(uname -m) in \
    x86_64) \
        sed -e '/m64=/s/lib64/lib/' \
          -i.orig gcc/config/i386/t-linux64 \
    ;; \
  esac

# Change workdir 
WORKDIR $PACKAGE_SOURCE/build

RUN ../configure                                       \
        --target=$LFS_TGT                              \
        --prefix=$LFS/tools                            \
        --with-glibc-version=2.11                      \
        --with-sysroot=$LFS                            \
        --with-newlib                                  \
        --without-headers                              \
        --enable-initfini-array                        \
        --disable-nls                                  \
        --disable-shared                               \
        --disable-multilib                             \
        --disable-decimal-float                        \
        --disable-threads                              \
        --disable-libatomic                            \
        --disable-libgomp                              \
        --disable-libquadmath                          \
        --disable-libssp                               \
        --disable-libvtv                               \
        --disable-libstdcxx                            \
        --enable-languages=c,c++

# Build package 
RUN make -j$CPUS

# Install package 
RUN make install

RUN cd .. && cat gcc/limitx.h gcc/glimits.h gcc/limity.h > \
  `dirname $($LFS_TGT-gcc -print-libgcc-file-name)`/install-tools/include/limits.h

# Compile Linux API Headers
WORKDIR $LFS/sources

ENV PACKAGE_VER=linux-5.10.17
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz
ENV PACKAGE_SOURCE=/$LFS/sources/$PACKAGE_VER

# Copy package 
COPY --from=arnaldo2792/lfs-sources /home/builder/sources/$PACKAGE_TAR /$LFS/sources

# Extract source
RUN tar -xf $PACKAGE_TAR

# Change workdir 
WORKDIR $PACKAGE_SOURCE

# Build package
RUN make mrproper -j$CPUS
RUN make headers -j$CPUS
RUN find usr/include -name '.*' -delete \
  && rm usr/include/Makefile            \
  && cp -rv usr/include $LFS/usr

# Compile glibc
WORKDIR /$LFS/sources

ENV PACKAGE_VER=glibc-2.33
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz
ENV PACKAGE_SOURCE=$LFS/sources/$PACKAGE_VER

# Copy package 
COPY --from=arnaldo2792/lfs-sources /home/builder/sources/$PACKAGE_TAR \
  /home/builder/sources/glibc-2.33-fhs-1.patch \
  /$LFS/sources/

# Create a symbolic link for LSB compliance 
RUN case $(uname -m) in \
    i?86)   ln -sfvl ../lib/ld-linux.so.2 $LFS/lib/ld-lsb.so.3 \
    ;; \
    # TODO: FIX symlinks
    x86_64) ln -sfv ../lib/ld-linux-x86-64.so.2 $LFS/lib64 \
            && ln -sfv ../lib/ld-linux-x86-64.so.2 $LFS/lib64/ld-lsb-x86-64.so.3 \
    ;; \
esac

# Extract source
RUN tar -xf /$LFS/sources/$PACKAGE_TAR && mkdir $PACKAGE_SOURCE/build

# Change workdir 
WORKDIR $PACKAGE_SOURCE/build

RUN cd ../ && patch -Np1 -i ../glibc-2.33-fhs-1.patch

RUN ../configure                         \
      --prefix=/usr                      \
      --host=$LFS_TGT                    \
      --build=$(../scripts/config.guess) \
      --enable-kernel=3.2                \
      --with-headers=$LFS/usr/include    \
      libc_cv_slibdir=/lib

# Build package 
RUN make -j$CPUS

# Install package
RUN make DESTDIR=$LFS install

# Finalize the installation of the limits.h header
RUN $LFS/tools/libexec/gcc/$LFS_TGT/10.2.0/install-tools/mkheaders

# Compile Libstdc++. 

ENV PACKAGE_VER=gcc-10.2.0
ENV PACKAGE_SOURCE=$LFS/sources/$PACKAGE_VER
ENV BUILD_LIBSTDC=$PACKAGE_SOURCE/build-libstdc
RUN mkdir $BUILD_LIBSTDC

WORKDIR $BUILD_LIBSTDC

RUN ../libstdc++-v3/configure       \
    --host=$LFS_TGT                 \
    --build=$(../config.guess)      \
    --prefix=/usr                   \
    --disable-multilib              \
    --disable-nls                   \
    --disable-libstdcxx-pch         \
    --with-gxx-include-dir=/tools/$LFS_TGT/include/c++/10.2.0

# Build package 
RUN make -j$CPUS

# Install package
RUN make DESTDIR=$LFS install

# At this point the initial cross toolchain is ready,
# it can be used to compile temporary tools
RUN rm -rf $LFS/sources/*

# Build layer with only the compiled cross toolchain
FROM base-builder as cross-toolchain

COPY --from=compiled-cross-toolchain $LFS /mnt/lfs/

WORKDIR $LFS/sources
