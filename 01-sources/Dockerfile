FROM fedora:33

# Add user builder
RUN useradd builder -m
WORKDIR /home/builder

# Install wget and findutils (provides xargs)
RUN dnf makecache && dnf install curl findutils -y

# Use `builder` to prevent permissions issues
USER builder

# Download the sources
RUN mkdir sources \
  && curl -LO http://linuxfromscratch.org/lfs/view/stable/wget-list \
  && sed -i "/expat/c\https://prdownloads.sourceforge.net/expat/expat-2.4.1.tar.xz" wget-list \
  && pushd sources \
  && cat ../wget-list | xargs -n 1 -P16 curl -LOSs \
  && popd 

# Check digests
RUN pushd sources \
  && curl -SLO http://linuxfromscratch.org/lfs/view/stable/md5sums \
  && sed -i "/expat/c\a4fb91a9441bcaec576d4c4a56fa3aa6  expat-2.4.1.tar.xz" md5sums \
  && md5sum -c md5sums \
  && popd
