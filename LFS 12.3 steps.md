# Before starting, ensure you are logged in as root on your host system.

# 2.4. Creating a New Partition
# Remember your LFS partition (e.g., /dev/sdaX) and swap partition (e.g., /dev/sdaY)
# Use a partitioning tool like cfdisk or fdisk to create partitions.
# Example for creating an ext4 filesystem on your LFS partition:
mkfs -v -t ext4 /dev/<xxx>

# Example for initializing a new swap partition:
mkswap /dev/<yyy>

# 2.6. Setting the $LFS Variable and the Umask
export LFS=/mnt/lfs
umask 022

# 2.7. Mounting the New Partition
mkdir -pv $LFS
mount -v -t ext4 /dev/<xxx> $LFS

# If using multiple partitions for LFS (e.g., /home):
# mkdir -v $LFS/home
# mount -v -t ext4 /dev/<yyy> $LFS/home

chown root:root $LFS
chmod 755 $LFS

# If using a swap partition, enable it:
# /sbin/swapon -v /dev/<zzz>

# 3.1. Introduction (Creating sources directory and setting permissions)
mkdir -v $LFS/sources
chmod -v a+wt $LFS/sources

# Download all packages and patches into $LFS/sources.
# You can use the wget-list-systemd file provided by LFS.
# Example:
# wget --input-file=wget-list-systemd --continue --directory-prefix=$LFS/sources

# Verify downloaded files (optional but recommended):
# pushd $LFS/sources
# md5sum -c md5sums
# popd

# If downloaded as non-root user, change ownership:
# chown root:root $LFS/sources/*

# 4.2. Creating a Limited Directory Layout in the LFS Filesystem
mkdir -pv $LFS/{etc,var} $LFS/usr/{bin,lib,sbin}
for i in bin lib sbin; do
  ln -sv usr/$i $LFS/$i
done
case $(uname -m) in
  x86_64) mkdir -pv $LFS/lib64 ;;
esac
mkdir -pv $LFS/tools

# 4.3. Adding the LFS User
groupadd lfs
useradd -s /bin/bash -g lfs -m -k /dev/null lfs

# Set a password for the lfs user (optional, needed if you want to log in as lfs directly)
# passwd lfs

chown -v lfs $LFS/{usr,lib,var,etc,tools}
case $(uname -m) in
  x86_64) chown -v lfs $LFS/lib64;;
esac

su lfs

# 4.4. Setting Up the Environment (as lfs user)
cat > ~/.bash_profile << "EOF"
exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
EOF

cat > ~/.bashrc << "EOF"
set +h
umask 022
LFS=/mnt/lfs
LC_ALL=POSIX
LFS_TGT=$(uname -m)-lfs-linux-gnu
PATH=/usr/bin
if [ -L /bin ]; then PATH=/bin:$PATH; fi
PATH=$LFS/tools/bin:$PATH
CONFIG_SITE=$LFS/usr/share/config.site
export LFS LC_ALL LFS_TGT PATH CONFIG_SITE
EOF

# Optional: Add MAKEFLAGS for parallel compilation
cat >> ~/.bashrc << "EOF"
export MAKEFLAGS='-j$(nproc)'
EOF

source ~/.bash_profile

# From this point onwards, all package building commands are executed as the 'lfs' user
# unless explicitly stated otherwise.

# 5.2. Binutils-2.44 - Pass 1
cd $LFS/sources
tar -xf binutils-2.44.tar.xz
cd binutils-2.44
mkdir -v build
cd build
time { ../configure --prefix=$LFS/tools \
            --with-sysroot=$LFS \
            --target=$LFS_TGT \
            --disable-nls \
            --enable-gprofng=no \
            --disable-werror \
            --enable-new-dtags \
            --enable-default-hash-style=gnu && \
       make && make install; }
cd $LFS/sources
rm -rf binutils-2.44

# 5.3. GCC-14.2.0 - Pass 1
cd $LFS/sources
tar -xf gcc-14.2.0.tar.xz
cd gcc-14.2.0
tar -xf ../mpfr-4.2.1.tar.xz
mv -v mpfr-4.2.1 mpfr
tar -xf ../gmp-6.3.0.tar.xz
mv -v gmp-6.3.0 gmp
tar -xf ../mpc-1.3.1.tar.gz
mv -v mpc-1.3.1 mpc
case $(uname -m) in
  x86_64)
    sed -e /m64=/s/lib64/lib/ \
        -i.orig gcc/config/i386/t-linux64
  ;;
esac
mkdir -v build
cd build
../configure \
    --target=$LFS_TGT \
    --prefix=$LFS/tools \
    --with-glibc-version=2.41 \
    --with-sysroot=$LFS \
    --with-newlib \
    --without-headers \
    --enable-default-pie \
    --enable-default-ssp \
    --disable-nls \
    --disable-shared \
    --disable-multilib \
    --disable-threads \
    --disable-libatomic \
    --disable-libgomp \
    --disable-libquadmath \
    --disable-libssp \
    --disable-libvtv \
    --disable-libstdcxx \
    --enable-languages=c,c++
make
make install
cd $LFS/sources/gcc-14.2.0
cat gcc/limitx.h gcc/glimits.h gcc/limity.h > \
  $(dirname $($LFS_TGT-gcc -print-libgcc-file-name))/include/limits.h
cd $LFS/sources
rm -rf gcc-14.2.0

# 5.4. Linux-6.13.4 API Headers
cd $LFS/sources
tar -xf linux-6.13.4.tar.xz
cd linux-6.13.4
make mrproper
make headers
find usr/include -type f ! -name '*.h' -delete
cp -rv usr/include $LFS/usr
cd $LFS/sources
rm -rf linux-6.13.4

# 5.5. Glibc-2.41
cd $LFS/sources
tar -xf glibc-2.41.tar.xz
cd glibc-2.41
case $(uname -m) in
  i?86) ln -sfv ld-linux.so.2 $LFS/lib/ld-lsb.so.3
  ;;
  x86_64) ln -sfv ../lib/ld-linux-x86-64.so.2 $LFS/lib64
          ln -sfv ../lib/ld-linux-x86-64.so.2 $LFS/lib64/ld-lsb-x86-64.so.3
  ;;
esac
patch -Npl -i ../glibc-2.41-fhs-1.patch
mkdir -v build
cd build
echo "rootsbindir=/usr/sbin" > configparms
../configure \
    --prefix=/usr \
    --host=$LFS_TGT \
    --build=$(../scripts/config.guess) \
    --enable-kernel=5.4 \
    --with-headers=$LFS/usr/include \
    --disable-nscd \
    libc_cv_slibdir=/usr/lib
make
make DESTDIR=$LFS install
sed '/RTLDLIST=/s@/usr@@g' -i $LFS/usr/bin/ldd
cd $LFS/sources
rm -rf glibc-2.41

# Sanity check for Glibc
echo 'int main(){}' > dummy.c
$LFS_TGT-gcc -xc dummy.c
readelf -l a.out | grep ld-linux
rm -v a.out
rm -v dummy.c

# 5.6. Libstdc++ from GCC-14.2.0
cd $LFS/sources
tar -xf gcc-14.2.0.tar.xz
cd gcc-14.2.0
mkdir -v build
cd build
../libstdc++-v3/configure \
    --host=$LFS_TGT \
    --build=$(../config.guess) \
    --prefix=/usr \
    --disable-multilib \
    --disable-nls \
    --disable-libstdcxx-pch \
    --with-gxx-include-dir=/tools/$LFS_TGT/include/c++/14.2.0
make
make DESTDIR=$LFS install
rm -v $LFS/usr/lib/libstdc++{,exp,fs,supc++}.la
cd $LFS/sources
rm -rf gcc-14.2.0

# 6.2. M4-1.4.19
cd $LFS/sources
tar -xf m4-1.4.19.tar.xz
cd m4-1.4.19
./configure --prefix=/usr \
            --host=$LFS_TGT \
            --build=$(build-aux/config.guess)
make
make DESTDIR=$LFS install
cd $LFS/sources
rm -rf m4-1.4.19

# 6.3. Ncurses-6.5
cd $LFS/sources
tar -xf ncurses-6.5.tar.gz
cd ncurses-6.5
mkdir build
pushd build
  ../configure AWK=gawk
  make -C include
  make -C progs tic
popd
./configure --prefix=/usr \
            --host=$LFS_TGT \
            --build=$(./config.guess) \
            --mandir=/usr/share/man \
            --with-manpage-format=normal \
            --with-shared \
            --without-normal \
            --with-cxx-shared \
            --without-debug \
            --without-ada \
            --disable-stripping \
            AWK=gawk
make
make DESTDIR=$LFS TIC_PATH=$(pwd)/build/progs/tic install
ln -sv libncursesw.so $LFS/usr/lib/libncurses.so
sed -e 's/^#if.*XOPEN.*\$/#if 1/' \
    -i $LFS/usr/include/curses.h
cd $LFS/sources
rm -rf ncurses-6.5

# 6.4. Bash-5.2.37
cd $LFS/sources
tar -xf bash-5.2.37.tar.gz
cd bash-5.2.37
./configure --prefix=/usr \
            --build=$(sh support/config.guess) \
            --host=$LFS_TGT \
            --without-bash-malloc
make
make DESTDIR=$LFS install
ln -sv bash $LFS/bin/sh
cd $LFS/sources
rm -rf bash-5.2.37

# 6.5. Coreutils-9.6
cd $LFS/sources
tar -xf coreutils-9.6.tar.xz
cd coreutils-9.6
./configure --prefix=/usr \
            --host=$LFS_TGT \
            --build=$(build-aux/config.guess) \
            --enable-install-program=hostname \
            --enable-no-install-program=kill,uptime
make
make DESTDIR=$LFS install
mv -v $LFS/usr/bin/chroot $LFS/usr/sbin
mkdir -pv $LFS/usr/share/man/man8
mv -v $LFS/usr/share/man/man1/chroot.1 $LFS/usr/share/man/man8/chroot.8
sed -i 's/"1"/"8"/' $LFS/usr/share/man/man8/chroot.8
cd $LFS/sources
rm -rf coreutils-9.6

# 6.6. Diffutils-3.11
cd $LFS/sources
tar -xf diffutils-3.11.tar.xz
cd diffutils-3.11
./configure --prefix=/usr \
            --host=$LFS_TGT \
            --build=$(./build-aux/config.guess)
make
make DESTDIR=$LFS install
cd $LFS/sources
rm -rf diffutils-3.11

# 6.7. File-5.46
cd $LFS/sources
tar -xf file-5.46.tar.gz
cd file-5.46
mkdir build
pushd build
  ../configure --disable-bzlib \
               --disable-libseccomp \
               --disable-xzlib \
               --disable-zlib
  make
popd
./configure --prefix=/usr --host=$LFS_TGT --build=$(./config.guess)
make FILE_COMPILE=$(pwd)/build/src/file
make DESTDIR=$LFS install
rm -v $LFS/usr/lib/libmagic.la
cd $LFS/sources
rm -rf file-5.46

# 6.8. Findutils-4.10.0
cd $LFS/sources
tar -xf findutils-4.10.0.tar.xz
cd findutils-4.10.0
./configure --prefix=/usr \
            --localstatedir=/var/lib/locate \
            --host=$LFS_TGT \
            --build=$(build-aux/config.guess)
make
make DESTDIR=$LFS install
cd $LFS/sources
rm -rf findutils-4.10.0

# 6.9. Gawk-5.3.1
cd $LFS/sources
tar -xf gawk-5.3.1.tar.xz
cd gawk-5.3.1
sed -i 's/extras//' Makefile.in
./configure --prefix=/usr \
            --host=$LFS_TGT \
            --build=$(build-aux/config.guess)
make
make DESTDIR=$LFS install
cd $LFS/sources
rm -rf gawk-5.3.1

# 6.10. Grep-3.11
cd $LFS/sources
tar -xf grep-3.11.tar.xz
cd grep-3.11
./configure --prefix=/usr \
            --host=$LFS_TGT \
            --build=$(./build-aux/config.guess)
make
make DESTDIR=$LFS install
cd $LFS/sources
rm -rf grep-3.11

# 6.11. Gzip-1.13
cd $LFS/sources
tar -xf gzip-1.13.tar.xz
cd gzip-1.13
./configure --prefix=/usr --host=$LFS_TGT
make
make DESTDIR=$LFS install
cd $LFS/sources
rm -rf gzip-1.13

# 6.12. Make-4.4.1
cd $LFS/sources
tar -xf make-4.4.1.tar.gz
cd make-4.4.1
./configure --prefix=/usr \
            --without-guile \
            --host=$LFS_TGT \
            --build=$(build-aux/config.guess)
make
make DESTDIR=$LFS install
cd $LFS/sources
rm -rf make-4.4.1

# 6.13. Patch-2.7.6
cd $LFS/sources
tar -xf patch-2.7.6.tar.xz
cd patch-2.7.6
./configure --prefix=/usr \
            --host=$LFS_TGT \
            --build=$(build-aux/config.guess)
make
make DESTDIR=$LFS install
cd $LFS/sources
rm -rf patch-2.7.6

# 6.14. Sed-4.9
cd $LFS/sources
tar -xf sed-4.9.tar.xz
cd sed-4.9
./configure --prefix=/usr \
            --host=$LFS_TGT \
            --build=$(./build-aux/config.guess)
make
make DESTDIR=$LFS install
cd $LFS/sources
rm -rf sed-4.9

# 6.15. Tar-1.35
cd $LFS/sources
tar -xf tar-1.35.tar.xz
cd tar-1.35
./configure --prefix=/usr \
            --host=$LFS_TGT \
            --build=$(build-aux/config.guess)
make
make DESTDIR=$LFS install
cd $LFS/sources
rm -rf tar-1.35

# 6.16. Xz-5.6.4
cd $LFS/sources
tar -xf xz-5.6.4.tar.xz
cd xz-5.6.4
./configure --prefix=/usr \
            --host=$LFS_TGT \
            --build=$(build-aux/config.guess) \
            --disable-static \
            --docdir=/usr/share/doc/xz-5.6.4
make
make DESTDIR=$LFS install
rm -v $LFS/usr/lib/liblzma.la
cd $LFS/sources
rm -rf xz-5.6.4

# 6.17. Binutils-2.44 - Pass 2
cd $LFS/sources
tar -xf binutils-2.44.tar.xz
cd binutils-2.44
sed -i '6031s/$add_dir//' ltmain.sh
mkdir -v build
cd build
../configure \
    --prefix=/usr \
    --build=$(../config.guess) \
    --host=$LFS_TGT \
    --disable-nls \
    --enable-shared \
    --enable-gprofng=no \
    --disable-werror \
    --enable-64-bit-bfd \
    --enable-new-dtags \
    --enable-default-hash-style=gnu
make
make DESTDIR=$LFS install
rm -v $LFS/usr/lib/lib{bfd,ctf,ctf-nobfd,opcodes,sframe}.{a,la}
cd $LFS/sources
rm -rf binutils-2.44

# 6.18. GCC-14.2.0 - Pass 2
cd $LFS/sources
tar -xf gcc-14.2.0.tar.xz
cd gcc-14.2.0
tar -xf ../mpfr-4.2.1.tar.xz
mv -v mpfr-4.2.1 mpfr
tar -xf ../gmp-6.3.0.tar.xz
mv -v gmp-6.3.0 gmp
tar -xf ../mpc-1.3.1.tar.gz
mv -v mpc-1.3.1 mpc
case $(uname -m) in
  x86_64)
    sed -e /m64=/s/lib64/lib/ \
        -i.orig gcc/config/i386/t-linux64
  ;;
esac
sed '/thread_header =/s/@.*@/gthr-posix.h/' \
    -i libgcc/Makefile.in libstdc++-v3/include/Makefile.in
mkdir -v build
cd build
unset CXXFLAGS
../configure \
    --build=$(../config.guess) \
    --host=$LFS_TGT \
    --target=$LFS_TGT \
    LDFLAGS_FOR_TARGET=-L$PWD/$LFS_TGT/libgcc \
    --prefix=/usr \
    --with-build-sysroot=$LFS \
    --enable-default-pie \
    --enable-default-ssp \
    --disable-nls \
    --disable-multilib \
    --disable-libatomic \
    --disable-libgomp \
    --disable-libquadmath \
    --disable-libsanitizer \
    --disable-libssp \
    --disable-libvtv \
    --enable-languages=c,c++
make
make DESTDIR=$LFS install
ln -sv gcc $LFS/usr/bin/cc
cd $LFS/sources
rm -rf gcc-14.2.0

# **Switch from 'lfs' user back to 'root' user**
exit

# 7.2. Changing Ownership (as root)
chown --from lfs -R root:root $LFS/{usr,lib,var,etc,bin,sbin,tools}
case $(uname -m) in
  x86_64) chown --from lfs -R root:root $LFS/lib64;;
esac

# 7.3. Preparing Virtual Kernel File Systems (as root)
mkdir -pv $LFS/{dev,proc,sys,run}
mount -v --bind /dev $LFS/dev
mount -vt devpts devpts -o gid=5,mode=0620 $LFS/dev/pts
mount -vt proc proc $LFS/proc
mount -vt sysfs sysfs $LFS/sys
mount -vt tmpfs tmpfs $LFS/run
if [ -h $LFS/dev/shm ]; then
  install -v -d -m 1777 $LFS$(realpath /dev/shm)
else
  mount -vt tmpfs -o nosuid,nodev tmpfs $LFS/dev/shm
fi

# 7.4. Entering the Chroot Environment (as root)
chroot "$LFS" /usr/bin/env -i \
    HOME=/root                  \
    TERM="$TERM"                \
    PS1='(lfs chroot) \u:\w\$ ' \
    PATH=/usr/bin:/usr/sbin     \
    MAKEFLAGS="-j$(nproc)"      \
    TESTSUITEFLAGS="-j$(nproc)" \
    /bin/bash --login

# From this point onwards, all remaining commands for the initial build are executed inside the chroot environment as 'root'.

# 7.5. Creating Directories
mkdir -pv /boot /home /mnt /opt /srv
mkdir -pv /etc/{opt,sysconfig}
mkdir -pv /lib/firmware
mkdir -pv /media/{floppy,cdrom}
mkdir -pv /usr/local/{bin,lib,sbin,src}
mkdir -pv /usr/lib/locale
mkdir -pv /usr/local/share/{color,dict,doc,info,locale,man}
mkdir -pv /usr/local/share/misc,terminfo,zoneinfo}
mkdir -pv /usr/local/share/man/man{1..8}
mkdir -pv /var/{cache,local,log,mail,opt,spool}
mkdir -pv /var/lib/{color,misc,locate}
ln -sfv /run /var/run
ln -sfv /run/lock /var/lock
install -dv -m 0750 /root
install -dv -m 1777 /tmp /var/tmp

# 7.6. Creating Essential Files and Symlinks
ln -sv /proc/self/mounts /etc/mtab
cat > /etc/hosts << "EOF"
127.0.0.1 localhost $(hostname)
::1       localhost ip6-localhost ip6-loopback
ff02::1   ip6-allnodes
ff02::2   ip6-allrouters
EOF
cat > /etc/passwd << "EOF"
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/dev/null:/usr/bin/false
daemon:x:6:6:Daemon User:/dev/null:/usr/bin/false
messagebus:x:18:18:D-Bus Message Daemon User:/run/dbus:/usr/bin/false
systemd-journal-gateway:x:73:73:systemd Journal Gateway:/:/usr/bin/false
systemd-journal-remote:x:74:74:systemd Journal Remote:/:/usr/bin/false
systemd-journal-upload:x:75:75:systemd Journal Upload:/:/usr/bin/false
systemd-network:x:76:76:systemd Network Management:/:/usr/bin/false
systemd-resolve:x:77:77:systemd Resolver:/:/usr/bin/false
systemd-timesync:x:78:78:systemd Time Synchronization:/:/usr/bin/false
systemd-coredump:x:79:79:systemd Core Dumper:/:/usr/bin/false
uuidd:x:80:80:UUID Generation Daemon User:/dev/null:/usr/bin/false
systemd-oom:x:81:81:systemd Out Of Memory Daemon:/:/usr/bin/false
nobody:x:65534:65534:Unprivileged User:/dev/null:/usr/bin/false
EOF
cat > /etc/group << "EOF"
root:x:0:
bin:x:1:daemon
sys:x:2:
kmem:x:3:
tape:x:4:
tty:x:5:
daemon:x:6:
floppy:x:7:
disk:x:8:
lp:x:9:
dialout:x:10:
audio:x:11:
video:x:12:
utmp:x:13:
cdrom:x:15:
adm:x:16:
messagebus:x:18:
systemd-journal:x:23:
input:x:24:
mail:x:34:
kvm:x:61:
systemd-journal-gateway:x:73:
systemd-journal-remote:x:74:
systemd-journal-upload:x:75:
systemd-network:x:76:
systemd-resolve:x:77:
systemd-timesync:x:78:
systemd-coredump:x:79:
uuidd:x:80:
systemd-oom:x:81:
wheel:x:97:
users:x:999:
nogroup:x:65534:
EOF
echo "tester:x:101:101::/home/tester:/bin/bash" >> /etc/passwd
echo "tester:x:101:" >> /etc/group
install -o tester -d /home/tester
exec /usr/bin/bash --login
touch /var/log/{btmp,lastlog,faillog,wtmp}
chgrp -v utmp /var/log/lastlog
chmod -v 664 /var/log/lastlog
chmod -v 600 /var/log/btmp

# 7.7. Gettext-0.24
cd /sources
tar -xf gettext-0.24.tar.xz
cd gettext-0.24
./configure --disable-shared
make
cp -v gettext-tools/src/{msgfmt,msgmerge,xgettext} /usr/bin
cd /sources
rm -rf gettext-0.24

# 7.8. Bison-3.8.2
cd /sources
tar -xf bison-3.8.2.tar.xz
cd bison-3.8.2
./configure --prefix=/usr \
            --docdir=/usr/share/doc/bison-3.8.2
make
make install
cd /sources
rm -rf bison-3.8.2

# 7.9. Perl-5.40.1
cd /sources
tar -xf perl-5.40.1.tar.xz
cd perl-5.40.1
sh Configure -des \
  -D prefix=/usr \
  -D vendorprefix=/usr \
  -D useshrplib \
  -D privlib=/usr/lib/perl5/5.40/core_perl \
  -D archlib=/usr/lib/perl5/5.40/core_perl \
  -D sitelib=/usr/lib/perl5/5.40/site_perl \
  -D sitearch=/usr/lib/perl5/5.40/site_perl \
  -D vendorlib=/usr/lib/perl5/5.40/vendor_perl \
  -D vendorarch=/usr/lib/perl5/5.40/vendor_perl
make
make install
cd /sources
rm -rf perl-5.40.1

# 7.10. Python-3.13.2
cd /sources
tar -xf Python-3.13.2.tar.xz
cd Python-3.13.2
./configure --prefix=/usr \
            --enable-shared \
            --without-ensurepip
make
make install
cd /sources
rm -rf Python-3.13.2

# 7.11. Texinfo-7.2
cd /sources
tar -xf texinfo-7.2.tar.xz
cd texinfo-7.2
./configure --prefix=/usr
make
make install
cd /sources
rm -rf texinfo-7.2

# 7.12. Util-linux-2.40.4
cd /sources
tar -xf util-linux-2.40.4.tar.xz
cd util-linux-2.40.4
mkdir -pv /var/lib/hwclock
./configure --libdir=/usr/lib \
            --runstatedir=/run \
            --disable-chfn-chsh \
            --disable-login \
            --disable-nologin \
            --disable-su \
            --disable-setpriv \
            --disable-runuser \
            --disable-pylibmount \
            --disable-static \
            --disable-liblastlog2 \
            --without-python \
            ADJTIME_PATH=/var/lib/hwclock/adjtime \
            --docdir=/usr/share/doc/util-linux-2.40.4
make
make install
cd /sources
rm -rf util-linux-2.40.4

# 7.13. Cleaning up and Saving the Temporary System
rm -rf /usr/share/{info,man,doc}/*
find /usr/{lib,libexec} -name \*.la -delete
rm -rf /tools

# Optional: Backup the system (exit chroot first, then execute as root on host)
# exit
# cd $LFS
# tar -cJpf $HOME/lfs-temp-tools-12.3-systemd.tar.xz .
# Re-enter chroot after backup
# chroot "$LFS" /usr/bin/env -i \
#    HOME=/root                  \
#    TERM="$TERM"                \
#    PS1='(lfs chroot) \u:\w\$ ' \
#    PATH=/usr/bin:/usr/sbin     \
#    MAKEFLAGS="-j$(nproc)"      \
#    TESTSUITEFLAGS="-j$(nproc)" \
#    /bin/bash --login


# 8.3. Man-pages-6.12
cd /sources
tar -xf man-pages-6.12.tar.xz
cd man-pages-6.12
rm -v man3/crypt*
make -R GIT=false prefix=/usr install
cd /sources
rm -rf man-pages-6.12

# 8.4. Iana-Etc-20250123
cd /sources
tar -xf iana-etc-20250123.tar.gz
cd iana-etc-20250123
cp services protocols /etc
cd /sources
rm -rf iana-etc-20250123

# 8.5. Glibc-2.41
cd /sources
tar -xf glibc-2.41.tar.xz
cd glibc-2.41
patch -Np1 -i ../glibc-2.41-fhs-1.patch
mkdir -v build
cd build
echo "rootsbindir=/usr/sbin" > configparms
../configure --prefix=/usr \
             --disable-werror \
             --enable-kernel=5.4 \
             --enable-stack-protector=strong \
             --disable-nscd \
             libc_cv_slibdir=/usr/lib
make
# make check # Optional: Run tests if desired
touch /etc/ld.so.conf
sed '/test-installation/s@$(PERL)@echo not running@' -i ../Makefile
make install
sed '/RTLDLIST=/s@/usr@@g' -i /usr/bin/ldd
localedef -i C -f UTF-8 C.UTF-8
localedef -i cs_CZ -f UTF-8 cs_CZ.UTF-8
localedef -i de_DE -f ISO-8859-1 de_DE
localedef -i de_DE@euro -f ISO-8859-15 de_DE@euro
localedef -i de_DE -f UTF-8 de_DE.UTF-8
localedef -i el_GR -f ISO-8859-7 el_GR
localedef -i en_GB -f ISO-8859-1 en_GB
localedef -i en_GB -f UTF-8 en_GB.UTF-8
localedef -i en_HK -f ISO-8859-1 en_HK
localedef -i en_PH -f ISO-8859-1 en_PH
localedef -i en_US -f ISO-8859-1 en_US
localedef -i en_US -f UTF-8 en_US.UTF-8
localedef -i es_ES -f ISO-8859-15 es_ES@euro
localedef -i es_MX -f ISO-8859-1 es_MX
localedef -i fa_IR -f UTF-8 fa_IR
localedef -i fr_FR -f ISO-8859-1 fr_FR
localedef -i fr_FR@euro -f ISO-8859-15 fr_FR@euro
localedef -i fr_FR -f UTF-8 fr_FR.UTF-8
localedef -i is_IS -f ISO-8859-1 is_IS
localedef -i is_IS -f UTF-8 is_IS.UTF-8
localedef -i it_IT -f ISO-8859-1 it_IT
localedef -i it_IT -f ISO-8859-15 it_IT@euro
localedef -i it_IT -f UTF-8 it_IT.UTF-8
localedef -i ja_JP -f EUC-JP ja_JP
localedef -i ja_JP -f SHIFT_JIS ja_JP.SJIS 2> /dev/null || true
localedef -i ja_JP -f UTF-8 ja_JP.UTF-8
localedef -i nl_NL@euro -f ISO-8859-15 nl_NL@euro
localedef -i ru_RU -f KOI8-R ru_RU.KOI8-R
localedef -i ru_RU -f UTF-8 ru_RU.UTF-8
localedef -i se_NO -f UTF-8 se_NO.UTF-8
localedef -i ta_IN -f UTF-8 ta_IN.UTF-8
localedef -i tr_TR -f UTF-8 tr_TR.UTF-8
localedef -i zh_CN -f GB18030 zh_CN.GB18030
localedef -i zh_HK -f BIG5-HKSCS zh_HK.BIG5-HKSCS
localedef -i zh_TW -f UTF-8 zh_TW.UTF-8

# Alternatively, install all locales:
# make localedata/install-locales
# localedef -i C -f UTF-8 C.UTF-8
# localedef -i ja_JP -f SHIFT_JIS ja_JP.SJIS 2> /dev/null || true

# 8.5.2.1. Adding nsswitch.conf
cat > /etc/nsswitch.conf << "EOF"
# Begin /etc/nsswitch.conf
passwd: files systemd
group: files systemd
shadow: files systemd
hosts: mymachines resolve [!UNAVAIL=return] files myhostname dns
networks: files
protocols: files
services: files
ethers: files
rpc: files
# End /etc/nsswitch.conf
EOF

# 8.5.2.2. Adding Time Zone Data
tar -xf ../tzdata2025a.tar.gz
ZONEINFO=/usr/share/zoneinfo
mkdir -pv $ZONEINFO/{posix,right}
for tz in etcetera southamerica northamerica europe africa antarctica \
          asia australasia backward; do
  zic -L /dev/null -d $ZONEINFO ${tz}
  zic -L /dev/null -d $ZONEINFO/posix ${tz}
  zic -L leapseconds -d $ZONEINFO/right ${tz}
done
cp -v zone.tab zone1970.tab iso3166.tab $ZONEINFO
zic -d $ZONEINFO -p America/New_York
unset ZONEINFO tz
# Determine your local timezone using tzselect, then link it:
# ln -sfv /usr/share/zoneinfo/<xxx> /etc/localtime

# 8.5.2.3. Configuring the Dynamic Loader
cat > /etc/ld.so.conf << "EOF"
# Begin /etc/ld.so.conf
/usr/local/lib
/opt/lib
EOF
cat >> /etc/ld.so.conf << "EOF"
# Add an include directory
include /etc/ld.so.conf.d/*.conf
EOF
mkdir -pv /etc/ld.so.conf.d
cd /sources
rm -rf glibc-2.41

# 8.6. Zlib-1.3.1
cd /sources
tar -xf zlib-1.3.1.tar.gz
cd zlib-1.3.1
./configure --prefix=/usr
make
make check
make install
rm -fv /usr/lib/libz.a
cd /sources
rm -rf zlib-1.3.1

# 8.7. Bzip2-1.0.8
cd /sources
tar -xf bzip2-1.0.8.tar.gz
cd bzip2-1.0.8
patch -Np1 -i ../bzip2-1.0.8-install_docs-1.patch
sed -i 's@\(ln -s -f \)$(PREFIX)/bin/@\1@' Makefile
sed -i "s@(PREFIX)/man@(PREFIX)/share/man@g" Makefile
make -f Makefile-libbz2_so
make clean
make
make PREFIX=/usr install
cp -av libbz2.so.* /usr/lib
ln -sv libbz2.so.1.0.8 /usr/lib/libbz2.so
cp -v bzip2-shared /usr/bin/bzip2
for i in /usr/bin/{bzcat,bunzip2}; do
  ln -sfv bzip2 $i
done
rm -fv /usr/lib/libbz2.a
cd /sources
rm -rf bzip2-1.0.8

# 8.8. Xz-5.6.4
cd /sources
tar -xf xz-5.6.4.tar.xz
cd xz-5.6.4
./configure --prefix=/usr \
            --disable-static \
            --docdir=/usr/share/doc/xz-5.6.4
make
make check
make install
cd /sources
rm -rf xz-5.6.4

# 8.9. Lz4-1.10.0
cd /sources
tar -xf lz4-1.10.0.tar.gz
cd lz4-1.10.0
make BUILD_STATIC=no PREFIX=/usr
make -j1 check
make BUILD_STATIC=no PREFIX=/usr install
cd /sources
rm -rf lz4-1.10.0

# 8.10. Zstd-1.5.7
cd /sources
tar -xf zstd-1.5.7.tar.gz
cd zstd-1.5.7
make prefix=/usr
make check
make prefix=/usr install
rm -v /usr/lib/libzstd.a
cd /sources
rm -rf zstd-1.5.7

# 8.11. File-5.46
cd /sources
tar -xf file-5.46.tar.gz
cd file-5.46
./configure --prefix=/usr
make
make check
make install
cd /sources
rm -rf file-5.46

# 8.12. Readline-8.2.13
cd /sources
tar -xf readline-8.2.13.tar.gz
cd readline-8.2.13
sed -i '/MV.*old/d' Makefile.in
sed -i '/{OLDSUFF}/c:' support/shlib-install
sed -i 's/-Wl,-rpath,[^ ]*//' support/shobj-conf
./configure --prefix=/usr \
            --disable-static \
            --with-curses \
            --docdir=/usr/share/doc/readline-8.2.13
make SHLIB_LIBS="-lncursesw"
make install
# Optional: Install documentation
# install -v -m644 doc/*.{ps,pdf,html,dvi} /usr/share/doc/readline-8.2.13
cd /sources
rm -rf readline-8.2.13

# 8.13. M4-1.4.19
cd /sources
tar -xf m4-1.4.19.tar.xz
cd m4-1.4.19
./configure --prefix=/usr
make
make check
make install
cd /sources
rm -rf m4-1.4.19

# 8.14. Bc-7.0.3
cd /sources
tar -xf bc-7.0.3.tar.xz
cd bc-7.0.3
CC=gcc ./configure --prefix=/usr -G -O3 -r
make
make test
make install
cd /sources
rm -rf bc-7.0.3

# 8.15. Flex-2.6.4
cd /sources
tar -xf flex-2.6.4.tar.gz
cd flex-2.6.4
./configure --prefix=/usr \
            --docdir=/usr/share/doc/flex-2.6.4 \
            --disable-static
make
make check
make install
ln -sv flex /usr/bin/lex
ln -sv flex.1 /usr/share/man/man1/lex.1
cd /sources
rm -rf flex-2.6.4

# 8.16. Tcl-8.6.16
cd /sources
tar -xf tcl8.6.16-src.tar.gz
cd tcl8.6.16
SRCDIR=$(pwd)
cd unix
./configure --prefix=/usr \
            --mandir=/usr/share/man \
            --disable-rpath
make
sed -e "s|$SRCDIR/unix|/usr/lib|" \
    -e "s|$SRCDIR|/usr/include|" \
    -i tclConfig.sh
sed -e "s|$SRCDIR/unix/pkgs/tdbc1.1.10|/usr/lib/tdbc1.1.10|" \
    -e "s|$SRCDIR/pkgs/tdbc1.1.10/generic|/usr/include|" \
    -e "s|$SRCDIR/pkgs/tdbc1.1.10/library|/usr/lib/tcl8.6|" \
    -e "s|$SRCDIR/pkgs/tdbc1.1.10|/usr/include|" \
    -i pkgs/tdbc1.1.10/tdbcConfig.sh
sed -e "s|$SRCDIR/unix/pkgs/itcl4.3.2|/usr/lib/itcl4.3.2|" \
    -e "s|$SRCDIR/pkgs/itcl4.3.2/generic|/usr/include|" \
    -e "s|$SRCDIR/pkgs/itcl4.3.2|/usr/include|" \
    -i pkgs/itcl4.3.2/itclConfig.sh
unset SRCDIR
make test
make install
chmod -v u+w /usr/lib/libtcl8.6.so
make install-private-headers
ln -sfv tclsh8.6 /usr/bin/tclsh
mv /usr/share/man/man3/{Thread,Tcl_Thread}.3
# Optional: Install documentation
# cd ..
# tar -xf ../tcl8.6.16-html.tar.gz --strip-components=1
# mkdir -v -p /usr/share/doc/tcl-8.6.16
# cp -v -r ./html/* /usr/share/doc/tcl-8.6.16
cd /sources
rm -rf tcl8.6.16

# 8.17. Expect-5.45.4
cd /sources
tar -xf expect5.45.4.tar.gz
cd expect5.45.4
python3 -c 'from pty import spawn; spawn(["echo", "ok"])'
patch -Np1 -i ../expect-5.45.4-gcc14-1.patch
./configure --prefix=/usr \
            --with-tcl=/usr/lib \
            --enable-shared \
            --disable-rpath \
            --mandir=/usr/share/man \
            --with-tclinclude=/usr/include
make
make test
make install
ln -svf expect5.45.4/libexpect5.45.4.so /usr/lib
cd /sources
rm -rf expect5.45.4

# 8.18. DejaGNU-1.6.3
cd /sources
tar -xf dejagnu-1.6.3.tar.gz
cd dejagnu-1.6.3
mkdir -v build
cd build
../configure --prefix=/usr
makeinfo --html --no-split -o doc/dejagnu.html ../doc/dejagnu.texi
makeinfo --plaintext -o doc/dejagnu.txt ../doc/dejagnu.texi
make check
make install
install -v -dm755 /usr/share/doc/dejagnu-1.6.3
install -v -m644 doc/dejagnu.{html,txt} /usr/share/doc/dejagnu-1.6.3
cd /sources
rm -rf dejagnu-1.6.3

# 8.19. Pkgconf-2.3.0
cd /sources
tar -xf pkgconf-2.3.0.tar.xz
cd pkgconf-2.3.0
./configure --prefix=/usr \
            --disable-static \
            --docdir=/usr/share/doc/pkgconf-2.3.0
make
make install
ln -sv pkgconf /usr/bin/pkg-config
ln -sv pkgconf.1 /usr/share/man/man1/pkg-config.1
cd /sources
rm -rf pkgconf-2.3.0

# 8.20. Binutils-2.44
cd /sources
tar -xf binutils-2.44.tar.xz
cd binutils-2.44
mkdir -v build
cd build
../configure --prefix=/usr \
            --sysconfdir=/etc \
            --enable-ld=default \
            --enable-plugins \
            --enable-shared \
            --disable-werror \
            --enable-64-bit-bfd \
            --enable-new-dtags \
            --with-system-zlib \
            --enable-default-hash-style=gnu
make tooldir=/usr
make -k check
make tooldir=/usr install
rm -rfv /usr/lib/lib{bfd,ctf,ctf-nobfd,gprofng,opcodes,sframe}.a \
        /usr/share/doc/gprofng/
cd /sources
rm -rf binutils-2.44

# 8.21. GMP-6.3.0
cd /sources
tar -xf gmp-6.3.0.tar.xz
cd gmp-6.3.0
./configure --prefix=/usr \
            --enable-cxx \
            --disable-static \
            --docdir=/usr/share/doc/gmp-6.3.0
make
make html
make check 2>&1 | tee gmp-check-log
awk '/# PASS:/{total+=$3} ; END{print total}' gmp-check-log
make install
make install-html
cd /sources
rm -rf gmp-6.3.0

# 8.22. MPFR-4.2.1
cd /sources
tar -xf mpfr-4.2.1.tar.xz
cd mpfr-4.2.1
./configure --prefix=/usr \
            --disable-static \
            --enable-thread-safe \
            --docdir=/usr/share/doc/mpfr-4.2.1
make
make html
make check
make install
make install-html
cd /sources
rm -rf mpfr-4.2.1

# 8.23. MPC-1.3.1
cd /sources
tar -xf mpc-1.3.1.tar.gz
cd mpc-1.3.1
./configure --prefix=/usr \
            --disable-static \
            --docdir=/usr/share/doc/mpc-1.3.1
make
make html
make check
make install
make install-html
cd /sources
rm -rf mpc-1.3.1

# 8.24. Attr-2.5.2
cd /sources
tar -xf attr-2.5.2.tar.gz
cd attr-2.5.2
./configure --prefix=/usr \
            --disable-static \
            --sysconfdir=/etc \
            --docdir=/usr/share/doc/attr-2.5.2
make
make check
make install
cd /sources
rm -rf attr-2.5.2

# 8.25. Acl-2.3.2
cd /sources
tar -xf acl-2.3.2.tar.xz
cd acl-2.3.2
./configure --prefix=/usr \
            --disable-static \
            --docdir=/usr/share/doc/acl-2.3.2
make
make check
make install
cd /sources
rm -rf acl-2.3.2

# 8.26. Libcap-2.73
cd /sources
tar -xf libcap-2.73.tar.xz
cd libcap-2.73
sed -i '/install -m.*STA/d' libcap/Makefile
make prefix=/usr lib=lib
make test
make prefix=/usr lib=lib install
cd /sources
rm -rf libcap-2.73

# 8.27. Libxcrypt-4.4.38
cd /sources
tar -xf libxcrypt-4.4.38.tar.xz
cd libxcrypt-4.4.38
./configure --prefix=/usr \
            --enable-hashes=strong,glibc \
            --enable-obsolete-api=no \
            --disable-static \
            --disable-failure-tokens
make
make check
make install
# Optional: For binary-only applications that require ABI version 1 of libcrypt.so.1
# make distclean
# ./configure --prefix=/usr \
#             --enable-hashes=strong,glibc \
#             --enable-obsolete-api=glibc \
#             --disable-static \
#             --disable-failure-tokens
# make
# cp -av --remove-destination .libs/libcrypt.so.1* /usr/lib
cd /sources
rm -rf libxcrypt-4.4.38

# 8.28. Shadow-4.17.3
cd /sources
tar -xf shadow-4.17.3.tar.xz
cd shadow-4.17.3
sed -i 's/groups$(EXEEXT) //' src/Makefile.in
find man -name Makefile.in -exec sed -i 's/groups\.1 / /' {} \;
find man -name Makefile.in -exec sed -i 's/getspnam\.3 / /' {} \;
find man -name Makefile.in -exec sed -i 's/passwd\.5 / /' {} \;
sed -e 's:#ENCRYPT_METHOD DES:ENCRYPT_METHOD YESCRYPT:' \
    -e 's:/var/spool/mail:/var/mail:'                 \
    -e '/PATH=/{s@/sbin:@@;s@/bin:@@}'               \
    -i etc/login.defs
touch /usr/bin/passwd
./configure --sysconfdir=/etc \
            --disable-static \
            --with-{b,yes}crypt \
            --without-libbsd \
            --with-group-name-max-length=32
make
make exec_prefix=/usr install
make -C man install-man

# 8.28.2. Configuring Shadow
pwconv
grpconv
mkdir -p /etc/default
useradd -D --gid 999
# Optional: Prevent useradd from creating mailboxes
# sed -i '/MAIL/s/yes/no/' /etc/default/useradd

# 8.28.3. Setting the Root Password
passwd root
cd /sources
rm -rf shadow-4.17.3

# 8.29. GCC-14.2.0
cd /sources
tar -xf gcc-14.2.0.tar.xz
cd gcc-14.2.0
case $(uname -m) in
  x86_64)
    sed -e '/m64=/s/lib64/lib/' \
        -i.orig gcc/config/i386/t-linux64
  ;;
esac
mkdir -v build
cd build
../configure --prefix=/usr \
             LD=ld \
             --enable-languages=c,c++ \
             --enable-default-pie \
             --enable-default-ssp \
             --enable-host-pie \
             --disable-multilib \
             --disable-bootstrap \
             --disable-fixincludes \
             --with-system-zlib
make
ulimit -s -H unlimited
sed -e '/cpython/d' -i ../gcc/testsuite/gcc.dg/plugin/plugin.exp
sed -e 's/no-pic /&-no-pie /' -i ../gcc/testsuite/gcc.target/i386/pr113689-1.c
sed -e 's/300000/(1|300000)/' -i ../libgomp/testsuite/libgomp.c-c++-common/pr109062.c
sed -e 's/{ target nonpic } //' \
    -e '/GOTPCREL/d' -i ../gcc/testsuite/gcc.target/i386/fentryname3.c
chown -R tester .
su tester -c "PATH=$PATH make -k check"
../contrib/test_summary
make install
chown -v -R root:root \
  /usr/lib/gcc/$(gcc -dumpmachine)/14.2.0/include{,-fixed}
ln -svr /usr/bin/cpp /usr/lib
ln -sv gcc.1 /usr/share/man/man1/cc.1
ln -sfv ../../libexec/gcc/$(gcc -dumpmachine)/14.2.0/liblto_plugin.so \
  /usr/lib/bfd-plugins/
echo 'int main(){}' > dummy.c
cc dummy.c -v -Wl,--verbose &> dummy.log
readelf -l a.out | grep ': /lib'
grep -E -o '/usr/lib.*/S?crt[1in].*succeeded' dummy.log
grep 'SEARCH.*/usr/lib' dummy.log |sed 's|; |\n|g'
grep "/lib.*/libc.so.6 " dummy.log
grep found dummy.log
rm -v dummy.c a.out dummy.log
mkdir -pv /usr/share/gdb/auto-load/usr/lib
mv -v /usr/lib/*gdb.py /usr/share/gdb/auto-load/usr/lib
cd /sources
rm -rf gcc-14.2.0

# 8.30. Ncurses-6.5
cd /sources
tar -xf ncurses-6.5.tar.gz
cd ncurses-6.5
./configure --prefix=/usr \
            --mandir=/usr/share/man \
            --with-shared \
            --without-debug \
            --without-normal \
            --with-cxx-shared \
            --enable-pc-files \
            --with-pkg-config-libdir=/usr/lib/pkgconfig
make
make DESTDIR=$PWD/dest install
install -vm755 dest/usr/lib/libncursesw.so.6.5 /usr/lib
rm -v dest/usr/lib/libncursesw.so.6.5
sed -e 's/^#if.*XOPEN.*$/#if 1/' \
    -i dest/usr/include/curses.h
cp -av dest/* /
for lib in ncurses form panel menu ; do
  ln -sfv lib${lib}w.so /usr/lib/lib${lib}.so
  ln -sfv ${lib}w.pc /usr/lib/pkgconfig/${lib}.pc
done
ln -sfv libncursesw.so /usr/lib/libcurses.so
# Optional: Install documentation
# cp -v -R doc -T /usr/share/doc/ncurses-6.5
cd /sources
rm -rf ncurses-6.5

# 8.31. Sed-4.9
cd /sources
tar -xf sed-4.9.tar.xz
cd sed-4.9
./configure --prefix=/usr
make
make html
chown -R tester .
su tester -c "PATH=$PATH make check"
make install
install -d -m755 /usr/share/doc/sed-4.9
install -m644 doc/sed.html /usr/share/doc/sed-4.9
cd /sources
rm -rf sed-4.9

# 8.32. Psmisc-23.7
cd /sources
tar -xf psmisc-23.7.tar.xz
cd psmisc-23.7
./configure --prefix=/usr
make
make check
make install
cd /sources
rm -rf psmisc-23.7

# 8.33. Gettext-0.24
cd /sources
tar -xf gettext-0.24.tar.xz
cd gettext-0.24
./configure --prefix=/usr \
            --disable-static \
            --docdir=/usr/share/doc/gettext-0.24
make
make check
make install
chmod -v 0755 /usr/lib/preloadable_libintl.so
cd /sources
rm -rf gettext-0.24

# 8.34. Bison-3.8.2
cd /sources
tar -xf bison-3.8.2.tar.xz
cd bison-3.8.2
./configure --prefix=/usr --docdir=/usr/share/doc/bison-3.8.2
make
make check
make install
cd /sources
rm -rf bison-3.8.2

# 8.35. Grep-3.11
cd /sources
tar -xf grep-3.11.tar.xz
cd grep-3.11
sed -i "s/echo/#echo/" src/egrep.sh
./configure --prefix=/usr
make
make check
make install
cd /sources
rm -rf grep-3.11

# 8.36. Bash-5.2.37
cd /sources
tar -xf bash-5.2.37.tar.gz
cd bash-5.2.37
./configure --prefix=/usr \
            --without-bash-malloc \
            --with-installed-readline \
            --docdir=/usr/share/doc/bash-5.2.37
make
chown -R tester .
su -s /usr/bin/expect tester << "EOF"
set timeout -1
spawn make tests
expect eof
lassign [wait] _ _ _ value
exit $value
EOF
make install
exec /usr/bin/bash --login
cd /sources
rm -rf bash-5.2.37

# 8.37. Libtool-2.5.4
cd /sources
tar -xf libtool-2.5.4.tar.xz
cd libtool-2.5.4
./configure --prefix=/usr
make
make check
make install
rm -fv /usr/lib/libltdl.a
cd /sources
rm -rf libtool-2.5.4

# 8.38. GDBM-1.24
cd /sources
tar -xf gdbm-1.24.tar.gz
cd gdbm-1.24
./configure --prefix=/usr \
            --disable-static \
            --enable-libgdbm-compat
make
make check
make install
cd /sources
rm -rf gdbm-1.24

# 8.39. Gperf-3.1
cd /sources
tar -xf gperf-3.1.tar.gz
cd gperf-3.1
./configure --prefix=/usr --docdir=/usr/share/doc/gperf-3.1
make
make -j1 check
make install
cd /sources
rm -rf gperf-3.1

# 8.40. Expat-2.6.4
cd /sources
tar -xf expat-2.6.4.tar.xz
cd expat-2.6.4
./configure --prefix=/usr \
            --disable-static \
            --docdir=/usr/share/doc/expat-2.6.4
make
make check
make install
# Optional: Install documentation
# install -v -m644 doc/*.{html,css} /usr/share/doc/expat-2.6.4
cd /sources
rm -rf expat-2.6.4

# 8.41. Inetutils-2.6
cd /sources
tar -xf inetutils-2.6.tar.xz
cd inetutils-2.6
sed -i 's/def HAVE_TERMCAP_TGETENT/ 1/' telnet/telnet.c
./configure --prefix=/usr \
            --bindir=/usr/bin \
            --localstatedir=/var \
            --disable-logger \
            --disable-whois \
            --disable-rcp \
            --disable-rexec \
            --disable-rlogin \
            --disable-rsh \
            --disable-servers
make
make check
make install
mv -v /usr/{,s}bin/ifconfig
cd /sources
rm -rf inetutils-2.6

# 8.42. Less-668
cd /sources
tar -xf less-668.tar.gz
cd less-668
./configure --prefix=/usr --sysconfdir=/etc
make
make check
make install
cd /sources
rm -rf less-668

# 8.43. Perl-5.40.1
cd /sources
tar -xf perl-5.40.1.tar.xz
cd perl-5.40.1
export BUILD_ZLIB=False
export BUILD_BZIP2=0
sh Configure -des \
  -D prefix=/usr \
  -D vendorprefix=/usr \
  -D privlib=/usr/lib/perl5/5.40/core_perl \
  -D archlib=/usr/lib/perl5/5.40/core_perl \
  -D sitelib=/usr/lib/perl5/5.40/site_perl \
  -D sitearch=/usr/lib/perl5/5.40/site_perl \
  -D vendorlib=/usr/lib/perl5/5.40/vendor_perl \
  -D vendorarch=/usr/lib/perl5/5.40/vendor_perl \
  -D man1dir=/usr/share/man/man1 \
  -D man3dir=/usr/share/man/man3 \
  -D pager="/usr/bin/less -isR" \
  -D useshrplib \
  -D usethreads
make
TEST_JOBS=$(nproc) make test_harness
make install
unset BUILD_ZLIB BUILD_BZIP2
cd /sources
rm -rf perl-5.40.1

# 8.44. XML::Parser-2.47
cd /sources
tar -xf XML-Parser-2.47.tar.gz
cd XML-Parser-2.47
perl Makefile.PL
make
make test
make install
cd /sources
rm -rf XML-Parser-2.47

# 8.45. Intltool-0.51.0
cd /sources
tar -xf intltool-0.51.0.tar.gz
cd intltool-0.51.0
sed -i 's:\\\${:\\\$\\{:' intltool-update.in
./configure --prefix=/usr
make
make check
make install
install -v -Dm644 doc/I18N-HOWTO /usr/share/doc/intltool-0.51.0/I18N-HOWTO
cd /sources
rm -rf intltool-0.51.0

# 8.46. Autoconf-2.72
cd /sources
tar -xf autoconf-2.72.tar.xz
cd autoconf-2.72
./configure --prefix=/usr
make
make check
make install
cd /sources
rm -rf autoconf-2.72

# 8.47. Automake-1.17
cd /sources
tar -xf automake-1.17.tar.xz
cd automake-1.17
./configure --prefix=/usr --docdir=/usr/share/doc/automake-1.17
make
make -j$(($(nproc)>4?$(nproc):4)) check
make install
cd /sources
rm -rf automake-1.17

# 8.48. OpenSSL-3.4.1
cd /sources
tar -xf openssl-3.4.1.tar.gz
cd openssl-3.4.1
./config --prefix=/usr \
          --openssldir=/etc/ssl \
          --libdir=lib \
          shared \
          zlib-dynamic
make
HARNESS_JOBS=$(nproc) make test
sed -i '/INSTALL_LIBS/s/libcrypto.a libssl.a//' Makefile
make MANSUFFIX=ssl install
mv -v /usr/share/doc/openssl /usr/share/doc/openssl-3.4.1
# Optional: Install additional documentation
# cp -vfr doc/* /usr/share/doc/openssl-3.4.1
cd /sources
rm -rf openssl-3.4.1

# 8.49. Libelf from Elfutils-0.192
cd /sources
tar -xf elfutils-0.192.tar.bz2
cd elfutils-0.192
./configure --prefix=/usr \
            --disable-debuginfod \
            --enable-libdebuginfod=dummy
make
make check
make -C libelf install
install -vm644 config/libelf.pc /usr/lib/pkgconfig
rm /usr/lib/libelf.a
cd /sources
rm -rf elfutils-0.192

# 8.50. Libffi-3.4.7
cd /sources
tar -xf libffi-3.4.7.tar.gz
cd libffi-3.4.7
./configure --prefix=/usr \
            --disable-static \
            --with-gcc-arch=native
make
make check
make install
cd /sources
rm -rf libffi-3.4.7

# 8.51. Python-3.13.2
cd /sources
tar -xf Python-3.13.2.tar.xz
cd Python-3.13.2
./configure --prefix=/usr \
            --enable-shared \
            --with-system-expat \
            --enable-optimizations
make
make test TESTOPTS="--timeout 120"
make install
# Optional: Suppress pip warnings
# cat > /etc/pip.conf << EOF
# [global]
# root-user-action = ignore
# disable-pip-version-check = true
# EOF
# Optional: Install preformatted documentation
# install -v -dm755 /usr/share/doc/python-3.13.2/html
# tar --strip-components=1 \
#    --no-same-owner \
#    --no-same-permissions \
#    -C /usr/share/doc/python-3.13.2/html \
#    -xvf ../python-3.13.2-docs-html.tar.bz2
cd /sources
rm -rf Python-3.13.2 python-3.13.2-docs-html.tar.bz2

# 8.52. Flit-Core-3.11.0
cd /sources
tar -xf flit_core-3.11.0.tar.gz
cd flit_core-3.11.0
pip3 wheel -w dist --no-cache-dir --no-build-isolation --no-deps $PWD
pip3 install --no-index --find-links dist flit_core
cd /sources
rm -rf flit_core-3.11.0

# 8.53. Wheel-0.45.1
cd /sources
tar -xf wheel-0.45.1.tar.gz
cd wheel-0.45.1
pip3 wheel -w dist --no-cache-dir --no-build-isolation --no-deps $PWD
pip3 install --no-index --find-links dist wheel
cd /sources
rm -rf wheel-0.45.1

# 8.54. Setuptools-75.8.1
cd /sources
tar -xf setuptools-75.8.1.tar.gz
cd setuptools-75.8.1
pip3 wheel -w dist --no-cache-dir --no-build-isolation --no-deps $PWD
pip3 install --no-index --find-links dist setuptools
cd /sources
rm -rf setuptools-75.8.1

# 8.55. Ninja-1.12.1
cd /sources
tar -xf ninja-1.12.1.tar.gz
cd ninja-1.12.1
# Optional: Enable NINJAJOBS environment variable
# sed -i '/int Guess/a \
#   int j = 0;\
#   char* jobs = getenv( "NINJAJOBS" );\
#   if ( jobs != NULL ) j = atoi( jobs );\
#   if ( j > 0 ) return j;\
# ' src/ninja.cc
python3 configure.py --bootstrap --verbose
install -vm755 ninja /usr/bin/
install -vDm644 misc/bash-completion /usr/share/bash-completion/completions/ninja
install -vDm644 misc/zsh-completion /usr/share/zsh/site-functions/_ninja
cd /sources
rm -rf ninja-1.12.1

# 8.56. Meson-1.7.0
cd /sources
tar -xf meson-1.7.0.tar.gz
cd meson-1.7.0
pip3 wheel -w dist --no-cache-dir --no-build-isolation --no-deps $PWD
pip3 install --no-index --find-links dist meson
install -vDm644 data/shell-completions/bash/meson /usr/share/bash-completion/completions/meson
install -vDm644 data/shell-completions/zsh/_meson /usr/share/zsh/site-functions/_meson
cd /sources
rm -rf meson-1.7.0

# 8.57. Kmod-34
cd /sources
tar -xf kmod-34.tar.xz
cd kmod-34
mkdir -p build
cd build
meson setup --prefix=/usr .. \
            --sbindir=/usr/sbin \
            --buildtype=release \
            -D manpages=false
ninja
ninja install
cd /sources
rm -rf kmod-34

# 8.58. Coreutils-9.6
cd /sources
tar -xf coreutils-9.6.tar.xz
cd coreutils-9.6
patch -Np1 -i ../coreutils-9.6-i18n-1.patch
autoreconf -fv
automake -af
FORCE_UNSAFE_CONFIGURE=1 ./configure \
  --prefix=/usr \
  --enable-no-install-program=kill,uptime
make
make NON_ROOT_USERNAME=tester check-root
groupadd -g 102 dummy -U tester
chown -R tester .
su tester -c "PATH=$PATH make -k RUN_EXPENSIVE_TESTS=yes check" < /dev/null
groupdel dummy
make install
mv -v /usr/bin/chroot /usr/sbin
mv -v /usr/share/man/man1/chroot.1 /usr/share/man/man8/chroot.8
sed -i 's/"1"/"8"/' /usr/share/man/man8/chroot.8
cd /sources
rm -rf coreutils-9.6

# 8.59. Check-0.15.2
cd /sources
tar -xf check-0.15.2.tar.gz
cd check-0.15.2
./configure --prefix=/usr --disable-static
make
make check
make docdir=/usr/share/doc/check-0.15.2 install
cd /sources
rm -rf check-0.15.2

# 8.60. Diffutils-3.11
cd /sources
tar -xf diffutils-3.11.tar.xz
cd diffutils-3.11
./configure --prefix=/usr
make
make check
make install
cd /sources
rm -rf diffutils-3.11

# 8.61. Gawk-5.3.1
cd /sources
tar -xf gawk-5.3.1.tar.xz
cd gawk-5.3.1
sed -i 's/extras//' Makefile.in
./configure --prefix=/usr
make
chown -R tester .
su tester -c "PATH=$PATH make check"
rm -f /usr/bin/gawk-5.3.1
make install
ln -sv gawk.1 /usr/share/man/man1/awk.1
# Optional: Install documentation
# install -vDm644 doc/{awkforai.txt,*.{eps,pdf,jpg}} -t /usr/share/doc/gawk-5.3.1
cd /sources
rm -rf gawk-5.3.1

# 8.62. Findutils-4.10.0
cd /sources
tar -xf findutils-4.10.0.tar.xz
cd findutils-4.10.0
./configure --prefix=/usr --localstatedir=/var/lib/locate
make
chown -R tester .
su tester -c "PATH=$PATH make check"
make install
cd /sources
rm -rf findutils-4.10.0

# 8.63. Groff-1.23.0
cd /sources
tar -xf groff-1.23.0.tar.gz
cd groff-1.23.0
PAGE=<paper_size> ./configure --prefix=/usr
make
make check
make install
cd /sources
rm -rf groff-1.23.0

# 8.64. GRUB-2.12
cd /sources
tar -xf grub-2.12.tar.xz
cd grub-2.12
unset {C,CPP,CXX,LD}FLAGS
echo depends bli part_gpt > grub-core/extra_deps.lst
./configure --prefix=/usr \
            --sysconfdir=/etc \
            --disable-efiemu \
            --disable-werror
make
# make check # Not recommended
make install
mv -v /etc/bash_completion.d/grub /usr/share/bash-completion/completions
cd /sources
rm -rf grub-2.12

# 8.65. Gzip-1.13
cd /sources
tar -xf gzip-1.13.tar.xz
cd gzip-1.13
./configure --prefix=/usr
make
make check
make install
cd /sources
rm -rf gzip-1.13

# 8.66. IPRoute2-6.13.0
cd /sources
tar -xf iproute2-6.13.0.tar.xz
cd iproute2-6.13.0
sed -i /ARPD/d Makefile
rm -fv man/man8/arpd.8
make NETNS_RUN_DIR=/run/netns
make SBINDIR=/usr/sbin install
# Optional: Install documentation
# install -vDm644 COPYING README* -t /usr/share/doc/iproute2-6.13.0
cd /sources
rm -rf iproute2-6.13.0

# 8.67. Kbd-2.7.1
cd /sources
tar -xf kbd-2.7.1.tar.xz
cd kbd-2.7.1
patch -Np1 -i ../kbd-2.7.1-backspace-1.patch
sed -i '/RESIZECONS_PROGS=/s/yes/no/' configure
sed -i 's/resizecons.8 //' docs/man/man8/Makefile.in
./configure --prefix=/usr --disable-vlock
make
make check
make install
# Optional: Install documentation
# cp -R -v docs/doc -T /usr/share/doc/kbd-2.7.1
cd /sources
rm -rf kbd-2.7.1

# 8.68. Libpipeline-1.5.8
cd /sources
tar -xf libpipeline-1.5.8.tar.gz
cd libpipeline-1.5.8
./configure --prefix=/usr
make
make check
make install
cd /sources
rm -rf libpipeline-1.5.8

# 8.69. Make-4.4.1
cd /sources
tar -xf make-4.4.1.tar.gz
cd make-4.4.1
./configure --prefix=/usr
make
chown -R tester .
su tester -c "PATH=$PATH make check"
make install
cd /sources
rm -rf make-4.4.1

# 8.70. Patch-2.7.6
cd /sources
tar -xf patch-2.7.6.tar.xz
cd patch-2.7.6
./configure --prefix=/usr
make
make check
make install
cd /sources
rm -rf patch-2.7.6

# 8.71. Tar-1.35
cd /sources
tar -xf tar-1.35.tar.xz
cd tar-1.35
FORCE_UNSAFE_CONFIGURE=1 \
./configure --prefix=/usr
make
make check
make install
make -C doc install-html docdir=/usr/share/doc/tar-1.35
cd /sources
rm -rf tar-1.35

# 8.72. Texinfo-7.2
cd /sources
tar -xf texinfo-7.2.tar.xz
cd texinfo-7.2
./configure --prefix=/usr
make
make check
make install
# Optional: Install TeX components
# make TEXMF=/usr/share/texmf install-tex
# Optional: Recreate info dir file
# pushd /usr/share/info
#   rm -v dir
#   for f in *; do install-info $f dir 2>/dev/null; done
# popd
cd /sources
rm -rf texinfo-7.2

# 8.73. Vim-9.1.1166
cd /sources
tar -xf vim-9.1.1166.tar.gz
cd vim-9.1.1166
echo '#define SYS_VIMRC_FILE "/etc/vimrc"' >> src/feature.h
./configure --prefix=/usr
make
chown -R tester .
sed '/test_plugin_glvs/d' -i src/testdir/Make_all.mak
su tester -c "TERM=xterm-256color LANG=en_US.UTF-8 make -j1 test" \
  &> vim-test.log
make install
ln -sv vim /usr/bin/vi
for L in /usr/share/man/{,*/}man1/vim.1; do
  ln -sv vim.1 $(dirname $L)/vi.1
done
ln -sv ../vim/vim91/doc /usr/share/doc/vim-9.1.1166

# 8.73.2. Configuring Vim
cat > /etc/vimrc << "EOF"
" Begin /etc/vimrc
" Ensure defaults are set before customizing settings, not after
source $VIMRUNTIME/defaults.vim
let skip_defaults_vim=1
set nocompatible
set backspace=2
set mouse=
syntax on
if (&term == "xterm") || (&term == "putty")
  set background=dark
endif
" End /etc/vimrc
EOF
cd /sources
rm -rf vim-9.1.1166

# 8.74. MarkupSafe-3.0.2
cd /sources
tar -xf markupsafe-3.0.2.tar.gz
cd MarkupSafe-3.0.2
pip3 wheel -w dist --no-cache-dir --no-build-isolation --no-deps $PWD
pip3 install --no-index --find-links dist Markupsafe
cd /sources
rm -rf MarkupSafe-3.0.2

# 8.75. Jinja2-3.1.5
cd /sources
tar -xf jinja2-3.1.5.tar.gz
cd Jinja2-3.1.5
pip3 wheel -w dist --no-cache-dir --no-build-isolation --no-deps $PWD
pip3 install --no-index --find-links dist Jinja2
cd /sources
rm -rf Jinja2-3.1.5

# 8.76. Systemd-257.3
cd /sources
tar -xf systemd-257.3.tar.gz
cd systemd-257.3
sed -e 's/GROUP="render"/GROUP="video"/' \
    -e 's/GROUP="sgx", //' \
    -i rules.d/50-udev-default.rules.in
mkdir -p build
cd build
meson setup .. \
  --prefix=/usr \
  --buildtype=release \
  -D default-dnssec=no \
  -D firstboot=false \
  -D install-tests=false \
  -D ldconfig=false \
  -D sysusers=false \
  -D rpmmacrosdir=no \
  -D homed=disabled \
  -D userdb=false \
  -D man=disabled \
  -D mode=release \
  -D pamconfdir=no \
  -D dev-kvm-mode=0660 \
  -D nobody-group=nogroup \
  -D sysupdate=disabled \
  -D ukify=disabled \
  -D docdir=/usr/share/doc/systemd-257.3
ninja
echo 'NAME="Linux From Scratch"' > /etc/os-release
ninja test
ninja install
tar -xf ../../systemd-man-pages-257.3.tar.xz \
    --no-same-owner --strip-components=1 \
    -C /usr/share/man
systemd-machine-id-setup
systemctl preset-all
cd /sources
rm -rf systemd-257.3 systemd-man-pages-257.3.tar.xz

# 8.77. D-Bus-1.16.0
cd /sources
tar -xf dbus-1.16.0.tar.xz
cd dbus-1.16.0
mkdir build
cd build
meson setup --prefix=/usr --buildtype=release --wrap-mode=nofallback ..
ninja
ninja test
ninja install
ln -sfv /etc/machine-id /var/lib/dbus
cd /sources
rm -rf dbus-1.16.0

# 8.78. Man-DB-2.13.0
cd /sources
tar -xf man-db-2.13.0.tar.xz
cd man-db-2.13.0
./configure --prefix=/usr \
            --docdir=/usr/share/doc/man-db-2.13.0 \
            --sysconfdir=/etc \
            --disable-setuid \
            --enable-cache-owner=bin \
            --with-browser=/usr/bin/lynx \
            --with-vgrind=/usr/bin/vgrind \
            --with-grap=/usr/bin/grap
make
make check
make install
cd /sources
rm -rf man-db-2.13.0

# 8.79. Procps-ng-4.0.5
cd /sources
tar -xf procps-ng-4.0.5.tar.xz
cd procps-ng-4.0.5
./configure --prefix=/usr \
            --docdir=/usr/share/doc/procps-ng-4.0.5 \
            --disable-static \
            --disable-kill \
            --enable-watch8bit \
            --with-systemd
make
chown -R tester .
su tester -c "PATH=$PATH make check"
make install
cd /sources
rm -rf procps-ng-4.0.5

# 8.80. Util-linux-2.40.4
cd /sources
tar -xf util-linux-2.40.4.tar.xz
cd util-linux-2.40.4
./configure --bindir=/usr/bin \
            --libdir=/usr/lib \
            --runstatedir=/run \
            --sbindir=/usr/sbin \
            --disable-chfn-chsh \
            --disable-login \
            --disable-nologin \
            --disable-su \
            --disable-setpriv \
            --disable-runuser \
            --disable-pylibmount \
            --disable-liblastlog2 \
            --disable-static \
            --without-python \
            ADJTIME_PATH=/var/lib/hwclock/adjtime \
            --docdir=/usr/share/doc/util-linux-2.40.4
make
# Optional: Run tests
# touch /etc/fstab
# chown -R tester .
# su tester -c "make -k check"
make install
cd /sources
rm -rf util-linux-2.40.4

# 8.81. E2fsprogs-1.47.2
cd /sources
tar -xf e2fsprogs-1.47.2.tar.gz
cd e2fsprogs-1.47.2
mkdir -v build
cd build
../configure --prefix=/usr \
            --sysconfdir=/etc \
            --enable-elf-shlibs \
            --disable-libblkid \
            --disable-libuuid \
            --disable-uuidd \
            --disable-fsck
make
make check
make install
rm -fv /usr/lib/{libcom_err,libe2p,libext2fs,libss}.a
gunzip -v /usr/share/info/libext2fs.info.gz
install-info --dir-file=/usr/share/info/dir /usr/share/info/libext2fs.info
# Optional: Create additional documentation
# makeinfo -o doc/com_err.info ../lib/et/com_err.texinfo
# install -v -m644 doc/com_err.info /usr/share/info
# install-info --dir-file=/usr/share/info/dir /usr/share/info/com_err.info
# Optional: Modify mke2fs.conf
# sed 's/metadata_csum_seed,//' -i /etc/mke2fs.conf
cd /sources
rm -rf e2fsprogs-1.47.2

# 8.83. Stripping (Optional)
save_usrlib="$(cd /usr/lib; ls ld-linux*[^g])
  libc.so.6
  libthread_db.so.1
  libquadmath.so.0.0.0
  libstdc++.so.6.0.33
  libitm.so.1.0.0
  libatomic.so.1.2.0"
cd /usr/lib
for LIB in $save_usrlib; do
  objcopy --only-keep-debug --compress-debug-sections=zlib $LIB $LIB.dbg
  cp $LIB /tmp/$LIB
  strip --strip-unneeded /tmp/$LIB
  objcopy --add-gnu-debuglink=$LIB.dbg /tmp/$LIB
  install -vm755 /tmp/$LIB /usr/lib
  rm /tmp/$LIB
done
online_usrbin="bash find strip"
online_usrlib="libbfd-2.44.so
  libsframe.so.1.0.0
  libhistory.so.8.2
  libncursesw.so.6.5
  libm.so.6
  libreadline.so.8.2
  libz.so.1.3.1
  libzstd.so.1.5.7
  $(cd /usr/lib; find libnss*.so* -type f)"
for BIN in $online_usrbin; do
  cp /usr/bin/$BIN /tmp/$BIN
  strip --strip-unneeded /tmp/$BIN
  install -vm755 /tmp/$BIN /usr/bin
  rm /tmp/$BIN
done
for LIB in $online_usrlib; do
  cp /usr/lib/$LIB /tmp/$LIB
  strip --strip-unneeded /tmp/$LIB
  install -vm755 /tmp/$LIB /usr/lib
  rm /tmp/$LIB
done
for i in $(find /usr/lib -type f -name \*.so* ! -name \*dbg) \
  $(find /usr/lib -type f -name \*.a) \
  $(find /usr/{bin,sbin,libexec} -type f); do
  case "$online_usrbin $online_usrlib $save_usrlib" in
  *$(basename $i)* )
  ;;
  * ) strip --strip-unneeded $i
  ;;
  esac
done
unset BIN LIB save_usrlib online_usrbin online_usrlib

# 8.84. Cleaning Up
rm -rf /tmp/{*,.*}
find /usr/lib /usr/libexec -name \*.la -delete
find /usr -depth -name $(uname -m)-lfs-linux-gnu\* | xargs rm -rf
userdel -r tester
rm -rf /sources/*

# 9.2.1. Network Interface Configuration Files
# Example: Static IP Configuration
# Replace <network-device-name> with your actual network device name (e.g., enp2s1, eth0)
# Replace <Your Domain Name> with your domain name (e.g., example.org)
# Replace IP addresses with your network's configuration.
cat > /etc/systemd/network/10-eth-static.network << "EOF"
[Match]
Name=<network-device-name>
[Network]
Address=192.168.0.2/24
Gateway=192.168.0.1
DNS=192.168.0.1
Domains=<Your Domain Name>
EOF

# Example: DHCP Configuration
# Replace <network-device-name> with your actual network device name (e.g., enp2s1, eth0)
cat > /etc/systemd/network/10-eth-dhcp.network << "EOF"
[Match]
Name=<network-device-name>
[Network]
DHCP=ipv4
[DHCPv4]
UseDomains=true
EOF

# Optional: Disable systemd-networkd-wait-online if not using systemd-networkd
# systemctl disable systemd-networkd-wait-online

# 9.2.2. Creating the /etc/resolv.conf File
# If using systemd-resolved, no manual /etc/resolv.conf is needed.
# If using static /etc/resolv.conf (replace values as appropriate):
# cat > /etc/resolv.conf << "EOF"
# # Begin /etc/resolv.conf
# domain <Your Domain Name>
# nameserver <IP address of your primary nameserver>
# nameserver <IP address of your secondary nameserver>
# # End /etc/resolv.conf
# EOF

# 9.2.3. Configuring the system hostname
echo "<lfs>" > /etc/hostname # Replace <lfs> with your desired hostname

# 9.2.4. Customizing the /etc/hosts File
# Replace <192.168.0.2> with your IP and <FQDN> with your Fully Qualified Domain Name
cat > /etc/hosts << "EOF"
# Begin /etc/hosts
<192.168.0.2> <FQDN> [alias1] [alias2] ...
::1       ip6-localhost ip6-loopback
ff02::1   ip6-allnodes
ff02::2   ip6-allrouters
# End /etc/hosts
EOF

# 9.4.1. Dealing with Duplicate Devices (Optional)
# Example rule for webcam and TV tuner (modify for your devices):
# cat > /etc/udev/rules.d/83-duplicate_devs.rules << "EOF"
# # Persistent symlinks for webcam and tuner
# KERNEL=="video*", ATTRS{idProduct}=="1910", ATTRS{idVendor}=="0d81", SYMLINK+="webcam"
# KERNEL=="video*", ATTRS{device}=="0x036f", ATTRS{vendor}=="0x109e", SYMLINK+="tvtuner"
# EOF

# 9.5. Configuring the System Clock
# If hardware clock is set to local time:
cat > /etc/adjtime << "EOF"
0.0 0 0.0
0
LOCAL
EOF
# If hardware clock is set to UTC, systemd-timedated will adjust automatically if /etc/adjtime isn't present.
# To change timezone: timedatectl set-timezone TIMEZONE

# 9.5.1. Network Time Synchronization (Optional)
# To disable systemd-timesyncd:
# systemctl disable systemd-timesyncd

# 9.6. Configuring the Linux Console
# Example for Lat2-Terminus16 font (modify as needed):
echo FONT=Lat2-Terminus16 > /etc/vconsole.conf

# Example for German keyboard and console:
# cat > /etc/vconsole.conf << "EOF"
# KEYMAP=de-latin1
# FONT=Lat2-Terminus16
# EOF

# 9.7. Configuring the System Locale
# Replace <ll>_<CC>.<charmap><@modifiers> with your desired locale
cat > /etc/locale.conf << "EOF"
LANG=<ll>_<CC>.<charmap><@modifiers>
EOF
cat > /etc/profile << "EOF"
# Begin /etc/profile
for i in $(locale); do
  unset ${i%=*}
done
if [[ "$TERM" = linux ]]; then
  export LANG=C.UTF-8
else
  source /etc/locale.conf
  for i in $(locale); do
  key=${i%=*}
  if [[ -v $key ]]; then
  export $key
  fi
  done
fi
# End /etc/profile
EOF

# 9.8. Creating the /etc/inputrc File
cat > /etc/inputrc << "EOF"
# Begin /etc/inputrc
# Modified by Chris Lynn <roryo@roryo.dynup.net>
# Allow the command prompt to wrap to the next line
set horizontal-scroll-mode Off
# Enable 8-bit input
set meta-flag On
set input-meta On
# Turns off 8th bit stripping
set convert-meta Off
# Keep the 8th bit for display
set output-meta On
# none, visible or audible
set bell-style none
# All of the following map the escape sequence of the value
# contained in the 1st argument to the readline specific functions
"\eOd": backward-word
"\eOc": forward-word
# for linux console
"\e[1~": beginning-of-line
"\e[4~": end-of-line
"\e[5~": beginning-of-history
"\e[6~": end-of-history
"\e[3~": delete-char
"\e[2~": quoted-insert
# for xterm
"\eOH": beginning-of-line
"\eOF": end-of-line
# for Konsole
"\e[H": beginning-of-line
"\e[F": end-of-line
# End /etc/inputrc
EOF

# 9.9. Creating the /etc/shells File
cat > /etc/shells << "EOF"
# Begin /etc/shells
/bin/sh
/bin/bash
# End /etc/shells
EOF

# 9.10.2. Disabling Screen Clearing at Boot Time (Optional)
# mkdir -pv /etc/systemd/system/getty@tty1.service.d
# cat > /etc/systemd/system/getty@tty1.service.d/noclear.conf << EOF
# [Service]
# TTYVTDisallocate=no
# EOF

# 9.10.3. Disabling tmpfs for /tmp (Optional)
# ln -sfv /dev/null /etc/systemd/system/tmp.mount

# 9.10.4. Configuring Automatic File Creation and Deletion (Optional)
# mkdir -p /etc/tmpfiles.d
# cp /usr/lib/tmpfiles.d/tmp.conf /etc/tmpfiles.d

# 9.10.8. Working with Core Dumps (Optional: Limiting disk usage)
# mkdir -pv /etc/systemd/coredump.conf.d
# cat > /etc/systemd/coredump.conf.d/maxuse.conf << EOF
# [Coredump]
# MaxUse=5G
# EOF

# 10.2. Creating the /etc/fstab File
# Replace <xxx> and <yyy> with your partition names, and <fff> with your filesystem type.
cat > /etc/fstab << "EOF"
# Begin /etc/fstab
# file system mount-point type options dump fsck
# order
/dev/<xxx>  / <fff>  defaults 1 1
/dev/<yyy>  swap swap pri=1 0 0
# End /etc/fstab
EOF

# 10.3. Linux-6.13.4
cd /sources
tar -xf linux-6.13.4.tar.xz
cd linux-6.13.4
make mrproper
# make menuconfig # Or make defconfig, then menuconfig to adjust
# Example for defconfig:
# make defconfig
# Example for menuconfig:
# LANG=<host_LANG_value> LC_ALL= make menuconfig

make
make modules_install
cp -iv arch/x86/boot/bzImage /boot/vmlinuz-6.13.4-lfs-12.3-systemd
cp -iv System.map /boot/System.map-6.13.4
cp -iv .config /boot/config-6.13.4
cp -r Documentation -T /usr/share/doc/linux-6.13.4
# Optional: Change ownership if retaining source tree
# chown -R 0:0 .

# 10.3.2. Configuring Linux Module Load Order
install -v -m755 -d /etc/modprobe.d
cat > /etc/modprobe.d/usb.conf << "EOF"
# Begin /etc/modprobe.d/usb.conf
install ohci_hcd /sbin/modprobe ehci_hcd ; /sbin/modprobe -i ohci_hcd ; true
install uhci_hcd /sbin/modprobe ehci_hcd ; /sbin/modprobe -i uhci_hcd ; true
# End /etc/modprobe.d/usb.conf
EOF
cd /sources
rm -rf linux-6.13.4

# 10.4.3. Setting Up the Configuration
# Replace /dev/sda with your boot drive (usually the drive containing your LFS partition).
# If booted with UEFI and want to use i386-pc target: grub-install --target i386-pc /dev/sda
grub-install /dev/sda

# 10.4.4. Creating the GRUB Configuration File
# Replace (hd0,2) with your LFS root partition GRUB designator.
# Replace /dev/sda2 with your LFS root partition device node.
cat > /boot/grub/grub.cfg << "EOF"
# Begin /boot/grub/grub.cfg
set default=0
set timeout=5
insmod part_gpt
insmod ext2
set root=(hd0,2)
set gfxpayload=1024x768x32
menuentry "GNU/Linux, Linux 6.13.4-lfs-12.3-systemd" {
  linux /boot/vmlinuz-6.13.4-lfs-12.3-systemd root=/dev/sda2 ro
}
EOF

# 11.1. The End
echo 12.3-systemd > /etc/lfs-release
cat > /etc/lsb-release << "EOF"
DISTRIB_ID="Linux From Scratch"
DISTRIB_RELEASE="12.3-systemd"
DISTRIB_CODENAME="<your name here>" # Customize this
DISTRIB_DESCRIPTION="Linux From Scratch"
EOF
cat > /etc/os-release << "EOF"
NAME="Linux From Scratch"
VERSION="12.3-systemd"
ID=lfs
PRETTY_NAME="Linux From Scratch 12.3-systemd"
VERSION_CODENAME="<your name here>" # Customize this
HOME_URL="https://www.linuxfromscratch.org/lfs/"
RELEASE_TYPE="stable"
EOF

# Final steps before rebooting (still in chroot)
# Exit chroot
exit

# Back on host system (as root)
umount -v $LFS/dev/pts
mountpoint -q $LFS/dev/shm && umount -v $LFS/dev/shm
umount -v $LFS/dev
umount -v $LFS/run
umount -v $LFS/proc
umount -v $LFS/sys
# If multiple partitions were used for LFS, unmount them before unmounting $LFS:
# umount -v $LFS/home
umount -v $LFS

# Reboot the system!
reboot
