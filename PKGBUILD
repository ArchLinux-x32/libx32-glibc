# $Id: PKGBUILD 73695 2012-07-14 13:05:07Z allan $
# Maintainer: Jan Alexander Steffens (heftig) <jan.steffens@gmail.com>
# Contributor: Jan de Groot <jgc@archlinux.org>
# Contributor: Allan McRae <allan@archlinux.org>

# toolchain build order: linux-api-headers->glibc->binutils->gcc->binutils->glibc
# NOTE: valgrind requires rebuilt with each major glibc version

_pkgbasename=glibc
pkgname=lib32-$_pkgbasename
pkgver=2.16.0
pkgrel=2
pkgdesc="GNU C Library for multilib"
arch=('x86_64')
url="http://www.gnu.org/software/libc"
license=('GPL' 'LGPL')
makedepends=('gcc-multilib>=4.7')
options=('!strip' '!emptydirs')
source=(http://ftp.gnu.org/gnu/libc/${_pkgbasename}-${pkgver}.tar.xz{,.sig}
        glibc-2.15-fix-res_query-assert.patch
        glibc-2.15-revert-c5a0802a.patch
        lib32-glibc.conf)
md5sums=('80b181b02ab249524ec92822c0174cf7'
         '2a1221a15575820751c325ef4d2fbb90'
         '31f415b41197d85d3bbee3d1eecd06a3'
         '0a0383d50d63f1c02919fe9943b82014'
         '6e052f1cb693d5d3203f50f9d4e8c33b')

build() {
  cd ${srcdir}/${_pkgbasename}-${pkgver}

  # fix res_query assertion
  # http://sourceware.org/bugzilla/show_bug.cgi?id=13013
  patch -p1 -i ${srcdir}/glibc-2.15-fix-res_query-assert.patch

  # revert commit c5a0802a - causes various hangs
  # https://bugzilla.redhat.com/show_bug.cgi?id=552960
  patch -p1 -i ${srcdir}/glibc-2.15-revert-c5a0802a.patch

  cd ${srcdir}
  mkdir glibc-build
  cd glibc-build


  # Hack to fix NPTL issues with Xen, only required on 32bit platforms
  # TODO: make separate glibc-xen package for i686
  export CFLAGS="${CFLAGS} -mno-tls-direct-seg-refs"

  export CC="gcc -m32"
  export CXX="g++ -m32"
  echo "slibdir=/usr/lib32" >> configparms

  # remove hardening options from CFLAGS for building libraries
  CFLAGS=${CFLAGS/-fstack-protector/}
  CFLAGS=${CFLAGS/-D_FORTIFY_SOURCE=2/}

  ${srcdir}/${_pkgbasename}-${pkgver}/configure --prefix=/usr \
      --libdir=/usr/lib32 --libexecdir=/usr/lib32 \
      --with-headers=/usr/include \
      --enable-add-ons=nptl,libidn \
      --enable-obsolete-rpc \
      --enable-kernel=2.6.32 \
      --enable-bind-now --disable-profile \
      --enable-stackguard-randomization \
      --enable-multi-arch i686-unknown-linux-gnu

  # build libraries with hardening disabled
  echo "build-programs=no" >> configparms
  make
  
  # re-enable hardening for programs
  sed -i "/build-programs=/s#no#yes#" configparms
  echo "CC += -fstack-protector -D_FORTIFY_SOURCE=2" >> configparms
  echo "CXX += -fstack-protector -D_FORTIFY_SOURCE=2" >> configparms
  make

  # remove harding in preparation to run test-suite
  sed -i '2,4d' configparms
}

check() {
  cd ${srcdir}/glibc-build
  make -k check
}

package() {
  cd ${srcdir}/glibc-build
  make install_root=${pkgdir} install

  rm -rf ${pkgdir}/{etc,sbin,usr/{bin,sbin,share},var}

  # We need one 32 bit specific header file
  find ${pkgdir}/usr/include -type f -not -name stubs-32.h -delete

  # Do not strip the following files for improved debugging support
  # ("improved" as in not breaking gdb and valgrind...):
  #   ld-${pkgver}.so
  #   libc-${pkgver}.so
  #   libpthread-${pkgver}.so
  #   libthread_db-1.0.so

  cd $pkgdir
  strip $STRIP_BINARIES usr/lib32/getconf/*

  strip $STRIP_STATIC usr/lib32/*.a

  strip $STRIP_SHARED usr/lib32/{libanl,libBrokenLocale,libcidn,libcrypt}-*.so \
                      usr/lib32/libnss_{compat,db,dns,files,hesiod,nis,nisplus}-*.so \
                      usr/lib32/{libdl,libm,libnsl,libresolv,librt,libutil}-*.so \
                      usr/lib32/{libmemusage,libpcprofile,libSegFault}.so \
                      usr/lib32/{pt_chown,{audit,gconv}/*.so}

  # Dynamic linker
  mkdir ${pkgdir}/usr/lib
  ln -s ../lib32/ld-linux.so.2 ${pkgdir}/usr/lib/

  # Add lib32 paths to the default library search path
  install -Dm644 "$srcdir/lib32-glibc.conf" "$pkgdir/etc/ld.so.conf.d/lib32-glibc.conf"

  # Symlink /usr/lib32/locale to /usr/lib/locale
  ln -s ../lib/locale "$pkgdir/usr/lib32/locale"
}
