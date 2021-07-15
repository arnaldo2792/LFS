FROM lfs-additional-temporary-tools as glibc

ARG CPUS=8
ENV CPUS=$CPUS

WORKDIR /sources

COPY --from=arnaldo2792/lfs-sources /home/builder/sources/glibc-2.33.tar.xz .
COPY --from=arnaldo2792/lfs-sources /home/builder/sources/glibc-2.33-fhs-1.patch .

RUN tar -xf glibc-2.33.tar.xz
RUN cd glibc-2.33 \
  && patch -Np1 -i ../glibc-2.33-fhs-1.patch \
  && sed -e '402a\      *result = local->data.services[database_index];' -i nss/nss_database.c \
  && mkdir build \
  && cd build \
  && ../configure --prefix=/usr                       \
             --disable-werror                         \
             --enable-kernel=3.2                      \
             --enable-stack-protector=strong          \
             --with-headers=/usr/include              \
             libc_cv_slibdir=/lib                     \
  && make -j$CPUS                                     

FROM glibc as test-glibc
RUN cd /sources/glibc-2.33/build && make check

FROM glibc as builder
RUN touch /etc/ld.so.conf                            \
  && cd /sources/glibc-2.33/build                    \
  && sed '/test-installation/s@$(PERL)@echo not running@' -i ../Makefile \
  && make install -j$CPUS

FROM builder as man-pages

ENV PACKAGE_VER=man-pages-5.10
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz \
    PACKAGE_SOURCE=$PACKAGE_VER \
    PACKAGE_BUILD=/build

# Copy source
COPY --from=lfs-sources /home/builder/sources/$PACKAGE_TAR .

# Extract source
RUN tar -xf $PACKAGE_TAR && mkdir $PACKAGE_BUILD

RUN cd /sources/$PACKAGE_SOURCE && make DESTDIR=$PACKAGE_BUILD install 

FROM builder as iana-etc

ENV PACKAGE_VER=iana-etc-20210202
ENV PACKAGE_TAR=$PACKAGE_VER.tar.gz \
    PACKAGE_SOURCE=$PACKAGE_VER \
    PACKAGE_BUILD=/build/etc

# Copy source
COPY --from=lfs-sources /home/builder/sources/$PACKAGE_TAR .

# Extract source
RUN tar -xf $PACKAGE_TAR  && mkdir -p $PACKAGE_BUILD

RUN cd /sources/$PACKAGE_SOURCE && cp services protocols $PACKAGE_BUILD
