# Test the compiler
FROM arnaldo2792/lfs-cross-toolchain as test-compiler

RUN echo 'int main(){}' > dummy.c \
  && $LFS_TGT-gcc dummy.c \
  && readelf -l a.out | grep '/ld-linux' \
  && rm -v dummy.c a.out

# Temp layer to get CPUS argument
FROM arnaldo2792/lfs-cross-toolchain as builder
# CPUs to use for the build
ARG CPUS=8
ENV CPUS=$CPUS

# Compile m4
FROM builder as m4

ENV PACKAGE_VER=m4-1.4.18
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz \
    PACKAGE_SOURCE=$PACKAGE_VER

# Copy source
COPY --from=arnaldo2792/lfs-sources /home/builder/sources/$PACKAGE_TAR $LFS/sources/

# Extract source
RUN tar -xf $PACKAGE_TAR

WORKDIR $LFS/sources/$PACKAGE_SOURCE

# Make some fixes introduced by glibc-2.28
RUN sed -i 's/IO_ftrylockfile/IO_EOF_SEEN/' lib/*.c \
  && echo "#define _IO_IN_BACKUP 0x100" >> lib/stdio-impl.h

RUN ./configure --prefix=/usr   \
                --host=$LFS_TGT \
                --build=$(build-aux/config.guess)

# Build package 
RUN make -j$CPUS

# Install package
RUN make DESTDIR=$LFS install

# Clean up
RUN rm -rf $LFS/sources/*

# Compile ncurses
FROM builder as ncurses

ENV PACKAGE_VER=ncurses-6.2
ENV PACKAGE_TAR=$PACKAGE_VER.tar.gz \
    PACKAGE_SOURCE=$PACKAGE_VER

# Copy source
COPY --from=arnaldo2792/lfs-sources /home/builder/sources/$PACKAGE_TAR $LFS/sources/

# Extract source
RUN tar -xf $PACKAGE_TAR && mkdir $PACKAGE_SOURCE/build

WORKDIR $LFS/sources/$PACKAGE_SOURCE

# Ensure that gawk is found first during configuration
RUN sed -i s/mawk// configure

RUN pushd build \
  && ../configure \
  && make -C include \
  && make -C progs tic \
  && popd

RUN ./configure --prefix=/usr                \
                --host=$LFS_TGT              \
                --build=$(./config.guess)    \
                --mandir=/usr/share/man      \
                --with-manpage-format=normal \
                --with-shared                \
                --without-debug              \
                --without-ada                \
                --without-normal             \
                --enable-widec

# Build package 
RUN make -j$CPUS

# Install package
RUN make DESTDIR=$LFS TIC_PATH=$(pwd)/build/progs/tic install \
  && echo "INPUT(-lncursesw)" > $LFS/usr/lib/libncurses.so \
  && mv -v $LFS/usr/lib/libncursesw.so.6* $LFS/lib/ \
  && ln -sfv ../../lib/$(readlink $LFS/usr/lib/libncursesw.so) $LFS/usr/lib/libncursesw.so

# Clean up
RUN rm -rf $LFS/sources/*

# Compile bash, bash depends on ncurses
FROM ncurses as bash

ENV PACKAGE_VER=bash-5.1
ENV PACKAGE_TAR=$PACKAGE_VER.tar.gz \
    PACKAGE_SOURCE=$PACKAGE_VER

# Copy source
WORKDIR $LFS/sources
COPY --from=arnaldo2792/lfs-sources /home/builder/sources/$PACKAGE_TAR $LFS/sources/

# Extract package
RUN tar -xf $PACKAGE_TAR

WORKDIR $LFS/sources/$PACKAGE_SOURCE

RUN ./configure --prefix=/usr                   \
                --build=$(support/config.guess) \
                --host=$LFS_TGT                 \
                --without-bash-malloc

# Build package 
RUN make -j$CPUS

# Install package
RUN make DESTDIR=$LFS install
RUN mv $LFS/usr/bin/bash $LFS/bin/bash \
  && ln -sv bash $LFS/bin/sh

# Clean up
RUN rm -rf $LFS/sources/*

# Compile coreutils
FROM builder as coreutils 

ENV PACKAGE_VER=coreutils-8.32
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz \
    PACKAGE_SOURCE=$PACKAGE_VER

# Copy source
COPY --from=arnaldo2792/lfs-sources /home/builder/sources/$PACKAGE_TAR $LFS/sources/

# Extract package
RUN tar -xf $PACKAGE_TAR

WORKDIR $LFS/sources/$PACKAGE_SOURCE

RUN ./configure --prefix=/usr                 \
            --host=$LFS_TGT                   \
            --build=$(build-aux/config.guess) \
            --enable-install-program=hostname \
            --enable-no-install-program=kill,uptime

# Build package 
RUN make -j$CPUS

# Install package
RUN make DESTDIR=$LFS install \
  && mv -v $LFS/usr/bin/{cat,chgrp,chmod,chown,cp,date,dd,df,echo} $LFS/bin \
  && mv -v $LFS/usr/bin/{false,ln,ls,mkdir,mknod,mv,pwd,rm}        $LFS/bin \
  && mv -v $LFS/usr/bin/{rmdir,stty,sync,true,uname}    $LFS/bin \
  && mv -v $LFS/usr/bin/{head,nice,sleep,touch}         $LFS/bin \
  && mv -v $LFS/usr/bin/chroot                          $LFS/usr/sbin/ \
  && mkdir -pv $LFS/usr/share/man/man8 \
  && mv -v $LFS/usr/share/man/man1/chroot.1             $LFS/usr/share/man/man8/chroot.8 \
  && sed -i 's/"1"/"8"/'                                    $LFS/usr/share/man/man8/chroot.8

# Clean up
RUN rm -rf $LFS/sources/*

# Compile diffutils
FROM builder as diffutils

ENV PACKAGE_VER=diffutils-3.7
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz \
    PACKAGE_SOURCE=$PACKAGE_VER

# Copy source
COPY --from=arnaldo2792/lfs-sources /home/builder/sources/$PACKAGE_TAR $LFS/sources

# Extract package
RUN tar -xf $PACKAGE_TAR

WORKDIR $PACKAGE_SOURCE

# Build package 
RUN ./configure --prefix=/usr --host=$LFS_TGT
RUN make -j$CPUS

# Install package
RUN make DESTDIR=$LFS install

# Clean up
RUN rm -rf $LFS/sources/*

# Compile file
FROM builder as file

ENV PACKAGE_VER=file-5.39
ENV PACKAGE_TAR=$PACKAGE_VER.tar.gz \
    PACKAGE_SOURCE=$PACKAGE_VER

# Copy source
COPY --from=arnaldo2792/lfs-sources /home/builder/sources/$PACKAGE_TAR $LFS/sources/

# Extract source
RUN tar -xf $PACKAGE_TAR && mkdir $PACKAGE_SOURCE/build

WORKDIR $LFS/sources/$PACKAGE_SOURCE

# Build package 
RUN pushd build                          \
  && ../configure --disable-bzlib        \
                  --disable-libseccomp   \
                  --disable-xzlib        \
                  --disable-zlib         \
  && make -j$CPUS && popd                \
  && ./configure --prefix=/usr --host=$LFS_TGT --build=$(./config.guess)

RUN make FILE_COMPILE=$(pwd)/build/src/file -j$CPUS

# Install package
RUN make DESTDIR=$LFS install

# Clean up
RUN rm -rf $LFS/sources/*

# Compile findutils
FROM builder as findutils

ENV PACKAGE_VER=findutils-4.8.0
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz \
    PACKAGE_SOURCE=$PACKAGE_VER

# Extract package
COPY --from=arnaldo2792/lfs-sources /home/builder/sources/$PACKAGE_TAR $LFS/sources/

# Extract source
RUN tar -xf $PACKAGE_TAR

WORKDIR $PACKAGE_SOURCE

RUN ./configure --prefix=/usr   \
            --host=$LFS_TGT \
            --build=$(build-aux/config.guess)

# Build package 
RUN make -j$CPUS

RUN make DESTDIR=$LFS install \
  && mv -v $LFS/usr/bin/find $LFS/bin \
  && sed -i 's|find:=${BINDIR}|find:=/bin|' $LFS/usr/bin/updatedb

# Clean up
RUN rm -rf $LFS/sources/*

# Compile gawk
FROM builder as gawk

ENV PACKAGE_VER=gawk-5.1.0
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz \
    PACKAGE_SOURCE=$PACKAGE_VER

# Extract package
COPY --from=arnaldo2792/lfs-sources /home/builder/sources/$PACKAGE_TAR $LFS/sources/

# Extract source
RUN tar -xf $PACKAGE_TAR

WORKDIR $LFS/sources/$PACKAGE_SOURCE

RUN sed -i 's/extras//' Makefile.in \
    && ./configure --prefix=/usr   \
                --host=$LFS_TGT \
                --build=$(./config.guess)

# Build package 
RUN make -j$CPUS

RUN make DESTDIR=$LFS install

# Clean up
RUN rm -rf $LFS/sources/*

# Compile grep
FROM builder as grep

ENV PACKAGE_VER=grep-3.6
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz \
    PACKAGE_SOURCE=$PACKAGE_VER

# Extract package
COPY --from=arnaldo2792/lfs-sources /home/builder/sources/$PACKAGE_TAR $LFS/sources/

# Extract source
RUN tar -xf $PACKAGE_TAR

WORKDIR $LFS/sources/$PACKAGE_SOURCE

RUN ./configure --prefix=/usr   \
                --host=$LFS_TGT \
                --bindir=/bin

# Build package 
RUN make -j$CPUS

# Install package
RUN make DESTDIR=$LFS install

# Clean up
RUN rm -rf $LFS/sources/*

# Compile gzip
FROM builder as gzip

ENV PACKAGE_VER=gzip-1.10
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz \
    PACKAGE_SOURCE=$PACKAGE_VER

# Extract package
COPY --from=arnaldo2792/lfs-sources /home/builder/sources/$PACKAGE_TAR $LFS/sources/

# Extract source
RUN tar -xf $PACKAGE_TAR

WORKDIR $LFS/sources/$PACKAGE_SOURCE

RUN ./configure --prefix=/usr --host=$LFS_TGT

# Build package 
RUN make -j$CPUS

# Install package
RUN make DESTDIR=$LFS install && mv -v $LFS/usr/bin/gzip $LFS/bin

# Clean up
RUN rm -rf $LFS/sources/*

# Compile make
FROM builder as make

ENV PACKAGE_VER=make-4.3
ENV PACKAGE_TAR=$PACKAGE_VER.tar.gz \
    PACKAGE_SOURCE=$PACKAGE_VER

# Extract package
COPY --from=arnaldo2792/lfs-sources /home/builder/sources/$PACKAGE_TAR $LFS/sources/

# Extract source
RUN tar -xf $PACKAGE_TAR

WORKDIR $LFS/sources/$PACKAGE_SOURCE

RUN ./configure --prefix=/usr   \
                --without-guile \
                --host=$LFS_TGT \
                --build=$(build-aux/config.guess)

# Build package 
RUN make -j$CPUS

# Install package
RUN make DESTDIR=$LFS install

# Clean up
RUN rm -rf $LFS/sources/*

# Compile patch
FROM builder as patch

ENV PACKAGE_VER=patch-2.7.6
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz \
    PACKAGE_SOURCE=$PACKAGE_VER

# Extract package
COPY --from=arnaldo2792/lfs-sources /home/builder/sources/$PACKAGE_TAR $LFS/sources/

# Extract source
RUN tar -xf $PACKAGE_TAR

WORKDIR $LFS/sources/$PACKAGE_SOURCE

RUN ./configure --prefix=/usr   \
                --host=$LFS_TGT \
                --build=$(build-aux/config.guess)

# Build package 
RUN make -j$CPUS

# Install package
RUN make DESTDIR=$LFS install

# Clean up
RUN rm -rf $LFS/sources/*

# Compile sed
FROM builder as sed

ENV PACKAGE_VER=sed-4.8
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz \
    PACKAGE_SOURCE=$PACKAGE_VER

# Extract package
COPY --from=arnaldo2792/lfs-sources /home/builder/sources/$PACKAGE_TAR $LFS/sources/

# Extract source
RUN tar -xf $PACKAGE_TAR

WORKDIR $LFS/sources/$PACKAGE_SOURCE

RUN ./configure --prefix=/usr   \
                --host=$LFS_TGT \
                --bindir=/bin
# Build package 
RUN make -j$CPUS

# Install package
RUN make DESTDIR=$LFS install

# Clean up
RUN rm -rf $LFS/sources/*

# Compile tar
FROM builder as tar

ENV PACKAGE_VER=tar-1.34
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz \
    PACKAGE_SOURCE=$PACKAGE_VER

# Extract package
COPY --from=arnaldo2792/lfs-sources /home/builder/sources/$PACKAGE_TAR $LFS/sources/

# Extract source
RUN tar -xf $PACKAGE_TAR

WORKDIR $LFS/sources/$PACKAGE_SOURCE

RUN ./configure --prefix=/usr                     \
                --host=$LFS_TGT                   \
                --build=$(build-aux/config.guess) \
                --bindir=/bin

# Build package 
RUN make -j$CPUS

# Install package
RUN make DESTDIR=$LFS install

# Clean up
RUN rm -rf $LFS/sources/*

# Compile xz
FROM builder as xz

ENV PACKAGE_VER=xz-5.2.5
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz \
    PACKAGE_SOURCE=$PACKAGE_VER

# Extract package
COPY --from=arnaldo2792/lfs-sources /home/builder/sources/$PACKAGE_TAR $LFS/sources/

# Extract source
RUN tar -xf $PACKAGE_TAR

WORKDIR $LFS/sources/$PACKAGE_SOURCE

RUN ./configure --prefix=/usr                     \
                --host=$LFS_TGT                   \
                --build=$(build-aux/config.guess) \
                --disable-static                  \
                --docdir=/usr/share/doc/xz-5.2.5

# Build package 
RUN make -j$CPUS

# Install package
RUN make DESTDIR=$LFS install \
    && mv -v $LFS/usr/bin/{lzma,unlzma,lzcat,xz,unxz,xzcat}  $LFS/bin \
    && mv -v $LFS/usr/lib/liblzma.so.*                       $LFS/lib \
    && ln -svf ../../lib/$(readlink $LFS/usr/lib/liblzma.so) $LFS/usr/lib/liblzma.so

# Clean up
RUN rm -rf $LFS/sources/*

# Compile binutils
FROM builder as binutils

ENV PACKAGE_VER=binutils-2.36.1
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz \
    PACKAGE_SOURCE=$PACKAGE_VER

# Extract package
COPY --from=arnaldo2792/lfs-sources /home/builder/sources/$PACKAGE_TAR $LFS/sources/

# Extract source
RUN tar -xf $PACKAGE_TAR \
    && mkdir $PACKAGE_SOURCE/build

WORKDIR $LFS/sources/$PACKAGE_SOURCE/build

RUN ../configure                   \
        --prefix=/usr              \
        --build=$(../config.guess) \
        --host=$LFS_TGT            \
        --disable-nls              \
        --enable-shared            \
        --disable-werror           \
        --enable-64-bit-bfd

# Build package 
RUN make -j$CPUS

# Install package
RUN make DESTDIR=$LFS install \
    && install -vm755 libctf/.libs/libctf.so.0.0.0 $LFS/usr/lib

# Clean up
RUN rm -rf $LFS/sources/*

# Compile gcc
FROM binutils as gcc

WORKDIR $LFS/sources/

ENV PACKAGE_VER=gcc-10.2.0 \
    MPFR_VER=mpfr-4.1.0 \
    GMP_VER=gmp-6.2.1 \
    MPC_VER=mpc-1.2.1

ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz \
    PACKAGE_SOURCE=$PACKAGE_VER \
    MPFR_TAR=$MPFR_VER.tar.xz \
    MPFR_SOURCE=$MPFR_VER \
    GMP_TAR=$GMP_VER.tar.xz \
    GMP_SOURCE=$GMP_VER \
    MPC_TAR=$MPC_VER.tar.gz \
    MPC_SOURCE=$MPC_VER

# Copy package and dependency sources
COPY --from=arnaldo2792/lfs-sources /home/builder/sources/$PACKAGE_TAR \
  /home/builder/sources/$MPFR_TAR                      \
  /home/builder/sources/$GMP_TAR                      \
  /home/builder/sources/$MPC_TAR                      \
  $LFS/sources/

# Extract source
RUN tar -xf $PACKAGE_TAR && mkdir $PACKAGE_SOURCE/build \
    && tar -xf $MPFR_TAR && mv $MPFR_VER $PACKAGE_SOURCE/mpfr \
    && tar -xf $GMP_TAR && mv $GMP_VER $PACKAGE_SOURCE/gmp \
    && tar -xf $MPC_TAR && mv $MPC_VER $PACKAGE_SOURCE/mpc

# If building on x86_64, change the default directory name for 64-bit libraries to "lib"
RUN cd $PACKAGE_SOURCE \
    && case $(uname -m) in \
        x86_64)            \
          sed -e '/m64=/s/lib64/lib/' -i.orig gcc/config/i386/t-linux64 \
        ;; \
       esac

WORKDIR $LFS/sources/$PACKAGE_SOURCE/build

# Create a symlink that allows libgcc to be built with posix threads support
RUN mkdir -pv $LFS_TGT/libgcc \
    && ln -s ../../../libgcc/gthr-posix.h $LFS_TGT/libgcc/gthr-default.h

RUN ../configure                                   \
       --build=$(../config.guess)                  \
       --host=$LFS_TGT                             \
       --prefix=/usr                               \
       CC_FOR_TARGET=$LFS_TGT-gcc                  \
       --with-build-sysroot=$LFS                   \
       --enable-initfini-array                     \
       --disable-nls                               \
       --disable-multilib                          \
       --disable-decimal-float                     \
       --disable-libatomic                         \
       --disable-libgomp                           \
       --disable-libquadmath                       \
       --disable-libssp                            \
       --disable-libvtv                            \
       --disable-libstdcxx                         \
       --enable-languages=c,c++

RUN make -j8

RUN make DESTDIR=$LFS install \
    && ln -sv gcc $LFS/usr/bin/cc

# Clean up
RUN rm -rf $LFS/sources/*

# RUN mkdir $DESTDIR/lib && cd $DESTDIR/lib \
#     && ls ../ \
#     && ln -s ../usr/bin/cpp $DESTDIR/lib/cpp 

# Build temp layer
FROM scratch as temp

COPY --from=m4 /mnt/lfs/ /
COPY --from=bash /mnt/lfs/ /
COPY --from=coreutils /mnt/lfs/ /
COPY --from=diffutils /mnt/lfs/ /
COPY --from=file /mnt/lfs/ /
COPY --from=findutils /mnt/lfs/ /
COPY --from=gawk /mnt/lfs/ /
COPY --from=grep /mnt/lfs/ /
COPY --from=gzip /mnt/lfs/ /
COPY --from=make /mnt/lfs/ /
COPY --from=patch /mnt/lfs/ /
COPY --from=sed /mnt/lfs/ /
COPY --from=tar /mnt/lfs/ /
COPY --from=xz /mnt/lfs/ /
COPY --from=gcc /mnt/lfs/ /

FROM scratch as temporary-tools

ARG ARCH
ENV LFS_TGT=$ARCH-lfs-linux-gnu

COPY --from=temp / /

# Change the ownership of the /* directories to user root
RUN chown -R 0:0 /{usr,lib,etc,bin}
