FROM fedora:33

# Add user builder
RUN useradd builder -m
WORKDIR /home/builder

# Install wget and findutils (provides xargs)
RUN dnf makecache && dnf install wget findutils -y

# Use `builder` to prevent permissions issues
USER builder

# Download the sources
RUN mkdir sources \
  && wget -qO- http://linuxfromscratch.org/lfs/view/stable/wget-list \
  | xargs -n 1 -P16 wget -nv -P sources/

# Check digests
RUN pushd sources \
  && wget -qO- http://linuxfromscratch.org/lfs/view/stable/md5sums | md5sum -c --quiet \
  && popd
