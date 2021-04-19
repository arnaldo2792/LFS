FROM arnaldo2792/lfs-temporary-tools as test-compiler

RUN echo 'int main(){}' > dummy.c \
  && gcc dummy.c \
  && rm -v dummy.c a.out

FROM arnaldo2792/lfs-temporary-tools as filesystem

ARG CPUS=8
ENV CPUS=$CPUS

RUN mkdir -pv /{boot,home,mnt,opt,srv,run} \
 && mkdir -pv /etc/{opt,sysconfig} \
 && mkdir -pv /lib/firmware \
 && mkdir -pv /media/{floppy,cdrom} \
 && mkdir -pv /usr/{,local/}{bin,include,lib,sbin,src} \
 && mkdir -pv /usr/{,local/}share/{color,dict,doc,info,locale,man} \
 && mkdir -pv /usr/{,local/}share/{misc,terminfo,zoneinfo} \
 && mkdir -pv /usr/{,local/}share/man/man{1..8} \
 && mkdir -pv /var/{cache,local,log,mail,opt,spool} \
 && mkdir -pv /var/lib/{color,misc,locate}

RUN ln -sfv /run /var/run \
 && ln -sfv /run/lock /var/lock \
 && install -dv -m 0750 /root \
 && install -dv -m 1777 /tmp /var/tmp \
 && echo "127.0.0.1 localhost $(hostname)" > /etc/hosts

ADD ./passwd /etc/passwd
ADD ./group /etc/group

# Some tests require a regular user. We add this user here and
# delete this account at the end
RUN echo "tester:x:0:101::/home/tester:/bin/bash" >> /etc/passwd \
    && echo "tester:x:101:" >> /etc/group \
    && install -o tester -d /home/tester

# The login, agetty, and init programs (and others) use a number of log files to record
# information such as who was logged into the system and when. However, these programs
# will not write to the log files if they do not already exist. Initialize the log files
# and give them proper permissions
RUN touch /var/log/{btmp,lastlog,faillog,wtmp} \
    && chgrp -v utmp /var/log/lastlog \
    && chmod -v 664  /var/log/lastlog \
    && chmod -v 600  /var/log/btmp

FROM filesystem as builder

WORKDIR /sources

# Libstdc++ from GCC-10.2.0, Pass 2
ENV PACKAGE_VER=gcc-10.2.0
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz \
    PACKAGE_SOURCE=$PACKAGE_VER

# Copy source
COPY --from=lfs-sources /home/builder/sources/$PACKAGE_TAR .

# Extract source
RUN tar -xf $PACKAGE_TAR && mkdir $PACKAGE_SOURCE/build

RUN cd /sources/$PACKAGE_SOURCE \
    && ln -s gthr-posix.h libgcc/gthr-default.h \
    && cd /sources/$PACKAGE_SOURCE/build \
    && ../libstdc++-v3/configure         \
        CXXFLAGS="-g -O2 -D_GNU_SOURCE"  \
        --prefix=/usr                    \
        --disable-multilib               \
        --disable-nls                    \
        --host=$(uname -m)-lfs-linux-gnu \
        --disable-libstdcxx-pch

RUN cd /sources/$PACKAGE_SOURCE/build && make -j$CPUS
RUN cd /sources/$PACKAGE_SOURCE/build && make install -j$CPUS
RUN rm -rf /sources/*

# Compile gettext
FROM builder as gettext
ENV PACKAGE_VER=gettext-0.21
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz \
    PACKAGE_SOURCE=$PACKAGE_VER \
    PACKAGE_BUILD=/build

# Copy source
COPY --from=lfs-sources /home/builder/sources/$PACKAGE_TAR .

# Extract source
RUN tar -xf $PACKAGE_TAR && mkdir $PACKAGE_BUILD

RUN cd /sources/$PACKAGE_VER/ \
    && ./configure --disable-shared

RUN cd /sources/$PACKAGE_VER/ \
    && make -j$CPUS

RUN cp -v /sources/$PACKAGE_VER/gettext-tools/src/{msgfmt,msgmerge,xgettext} $PACKAGE_BUILD

# Compile bison
FROM builder as bison
ENV PACKAGE_VER=bison-3.7.5
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz \
    PACKAGE_SOURCE=$PACKAGE_VER \
    PACKAGE_BUILD=/build

# Copy source
COPY --from=lfs-sources /home/builder/sources/$PACKAGE_TAR .

# Extract source
RUN tar -xf $PACKAGE_TAR && mkdir $PACKAGE_BUILD

RUN cd /sources/$PACKAGE_VER/ \
    && ./configure --prefix=/usr \
                   --docdir=/usr/share/doc/bison-3.7.5

RUN cd /sources/$PACKAGE_VER/ \
    && make -j$CPUS

RUN cd /sources/$PACKAGE_VER/ \
    && make DESTDIR=$PACKAGE_BUILD install

# Compile Perl
FROM builder as perl
ENV PACKAGE_VER=perl-5.32.1
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz \
    PACKAGE_SOURCE=$PACKAGE_VER \
    PACKAGE_BUILD=/build

# Copy source
COPY --from=lfs-sources /home/builder/sources/$PACKAGE_TAR .

# Extract source
RUN tar -xf $PACKAGE_TAR && mkdir $PACKAGE_BUILD

RUN cd /sources/$PACKAGE_VER/ \
    && sh Configure -des                                 \
             -Dprefix=/usr                               \
             -Dvendorprefix=/usr                         \
             -Dprivlib=/usr/lib/perl5/5.32/core_perl     \
             -Darchlib=/usr/lib/perl5/5.32/core_perl     \
             -Dsitelib=/usr/lib/perl5/5.32/site_perl     \
             -Dsitearch=/usr/lib/perl5/5.32/site_perl    \
             -Dvendorlib=/usr/lib/perl5/5.32/vendor_perl \
             -Dvendorarch=/usr/lib/perl5/5.32/vendor_perl

RUN cd /sources/$PACKAGE_VER && make -j$CPUS

RUN cd /sources/$PACKAGE_VER && make DESTDIR=$PACKAGE_BUILD install

# Perl installs man pages if `DESTDIR` is set
RUN rm -rf $PACKAGE_BUILD/*.0

# Compile python
FROM builder as python
ENV PACKAGE_VER=Python-3.9.2
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz \
    PACKAGE_SOURCE=$PACKAGE_VER \
    PACKAGE_BUILD=/build

# Copy source
COPY --from=lfs-sources /home/builder/sources/$PACKAGE_TAR .

# Extract source
RUN tar -xf $PACKAGE_TAR && mkdir $PACKAGE_BUILD

RUN cd /sources/$PACKAGE_VER/ \
    && ./configure --prefix=/usr   \
                   --enable-shared \
                   --without-ensurepip

RUN cd /sources/$PACKAGE_VER && make -j$CPUS

RUN cd /sources/$PACKAGE_VER && make DESTDIR=$PACKAGE_BUILD install

# Compile texinfo
FROM builder as texinfo
ENV PACKAGE_VER=texinfo-6.7
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz \
    PACKAGE_SOURCE=$PACKAGE_VER \
    PACKAGE_BUILD=/build

# Copy source
COPY --from=lfs-sources /home/builder/sources/$PACKAGE_TAR .

# Texinfo depends on perl
COPY --from=perl /build/ /

# Extract source
RUN tar -xf $PACKAGE_TAR && mkdir $PACKAGE_BUILD

RUN cd /sources/$PACKAGE_VER/ \
    && ./configure --prefix=/usr

RUN cd /sources/$PACKAGE_VER && make -j$CPUS 

RUN cd /sources/$PACKAGE_VER && make DESTDIR=$PACKAGE_BUILD install

# Compile util-linux
FROM builder as util-linux
ENV PACKAGE_VER=util-linux-2.36.2
ENV PACKAGE_TAR=$PACKAGE_VER.tar.xz \
    PACKAGE_SOURCE=$PACKAGE_VER \
    PACKAGE_BUILD=/build

# Copy source
COPY --from=lfs-sources /home/builder/sources/$PACKAGE_TAR .

# Extract source
RUN tar -xf $PACKAGE_TAR \
    && mkdir -pv /var/lib/hwclock \
    && mkdir -p $PACKAGE_BUILD/var/lib/hwclock

RUN cd /sources/$PACKAGE_VER/ \
    && ./configure ADJTIME_PATH=/var/lib/hwclock/adjtime    \
                  --docdir=/usr/share/doc/util-linux-2.36.2 \
                  --disable-chfn-chsh  \
                  --disable-login      \
                  --disable-nologin    \
                  --disable-su         \
                  --disable-setpriv    \
                  --disable-runuser    \
                  --disable-pylibmount \
                  --disable-static     \
                  --without-python     \
                  runstatedir=/run

RUN cd /sources/$PACKAGE_VER && make -j$CPUS 

RUN cd /sources/$PACKAGE_VER && make DESTDIR=$PACKAGE_BUILD install

FROM scratch as temp 

COPY --from=builder / /
COPY --from=gettext /build /usr/bin
COPY --from=bison /build /
COPY --from=perl /build /
COPY --from=python /build /
COPY --from=texinfo /build /
COPY --from=util-linux /build /

RUN find /usr/{lib,libexec} -name \*.la -delete \
    && rm -rf /usr/share/{info,man,doc}/*

RUN strip --strip-debug /usr/lib/* 2>/dev/null \
    || strip --strip-unneeded $LFS/usr/{,s}bin/* 2>/dev/null \
    || strip --strip-unneeded $LFS/tools/bin/* 2>/dev/null \
    || exit 0

FROM scratch as additional-temporary-tools

COPY --from=temp / /
