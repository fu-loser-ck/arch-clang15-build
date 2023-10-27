# Maintainer: Evangelos Foutras <evangelos@foutrelis.com>
# Contributor: Jan "heftig" Steffens <jan.steffens@gmail.com>

pkgname=clang15-all
pkgver=15.0.7
pkgrel=1
pkgdesc="C language family frontend for LLVM"
arch=('x86_64')
url="https://clang.llvm.org/"
license=('custom:Apache 2.0 with LLVM Exception')
depends=('gcc' 'gcc-libs' 'perl' 'zlib' 'zstd' 'libffi' 'libedit' 'ncurses' 'libxml2')
makedepends=('cmake' 'ninja' 'zlib' 'zstd' 'perl' 'libffi' 'libedit' 'ncurses'
             'libxml2' 'python' 'python-setuptools' 'python-psutil'
             'python-sphinx' 'python-recommonmark')
optdepends=('openmp: OpenMP support in clang with -fopenmp'
            'python: for scan-view and git-clang-format')
options=('staticlibs' '!lto') # Getting thousands of test failures with LTO
provides=("clang-analyzer=$pkgver" "clang-tools-extra=$pkgver")
conflicts=('clang-analyzer' 'clang-tools-extra')
replaces=('clang-analyzer' 'clang-tools-extra')
_source_base=https://github.com/llvm/llvm-project/releases/download/llvmorg-$pkgver
source=($_source_base/llvm-project-$pkgver.src.tar.xz{,.sig}
        $pkgname-linker-wrapper-tool.patch::https://github.com/llvm/llvm-project/commit/c2aabcfc8395.patch
        $pkgname-structured-bindings-r1.patch::https://github.com/llvm/llvm-project/commit/127bf4438542.patch
        $pkgname-bitfield-value-capture.patch::https://github.com/llvm/llvm-project/commit/a1a71b7dc97b.patch
        enable-fstack-protector-strong-by-default.patch
        llvm-config.h)
sha256sums=('8b5fcb24b4128cf04df1b0b9410ce8b1a729cb3c544e6da885d234280dedeac6'
            'SKIP'
            '75f220b68622a57b49a9480fe2ee321c7ff9b5ce643091b6cb510b9e38400e92'
            '2b613e392b00aebbef27639d8f5c4a3252983e5497f9cff4eca44286ac692aa4'
            '0ae44d6e6f080364c74238b2960a3f23ecdc355ea96997eb2e8180a708007d39'
            '7a9ce949579a3b02d4b91b6835c4fb45adc5f743007572fb0e28e6433e48f3a5'
            '597dc5968c695bbdbb0eac9e8eb5117fcd2773bc91edf5ec103ecffffab8bc48')
validpgpkeys=('474E22316ABF4785A88C6E8EA2C794A986419D8A'  # Tom Stellard <tstellar@redhat.com>
              'D574BD5D1D0E98895E3BF90044F2485E45D59042') # Tobias Hieta <tobias@hieta.se>

prepare() {
  mv llvm-project{-$pkgver.src,}
  cd llvm-project-$pkgver.src
  mkdir build
  cd clang
  mv "$srcdir/clang-tools-extra-$pkgver.src" tools/extra
  patch -Np2 -i ../enable-fstack-protector-strong-by-default.patch

  # https://reviews.llvm.org/D145862
  patch -Np2 -l -i ../$pkgname-linker-wrapper-tool.patch

  # https://reviews.llvm.org/D122768 (needed for Chromium 113)
  sed 's|clang-tools-extra|clang/tools/extra|g' \
    ../$pkgname-structured-bindings-r1.patch | patch -Np2

  # https://reviews.llvm.org/D131202 (regression caused by the above)
  patch -Np2 -i ../$pkgname-bitfield-value-capture.patch

  # Attempt to convert script to Python 3
  2to3 -wn --no-diffs \
    tools/extra/clang-include-fixer/find-all-symbols/tool/run-find-all-symbols.py
}

build() {
  # ----------
  # Build ALL
  cd llvm-project-$pkgver.src/build

  # Build only minimal debug info to reduce size
  CFLAGS=${CFLAGS/-g /-g1 }
  CXXFLAGS=${CXXFLAGS/-g /-g1 }

  local cmake_args=(
    -G Ninja
    -DCMAKE_BUILD_TYPE=Release
    -DCMAKE_INSTALL_DOCDIR=share/doc
    -DCMAKE_INSTALL_PREFIX=/usr/clang15
    -DCMAKE_SKIP_RPATH=ON
    -DCOMPILER_RT_INSTALL_PATH="/usr/clang15/lib/clang/$pkgver"
    -DCLANG_DEFAULT_PIE_ON_LINUX=ON
    -DCLANG_LINK_CLANG_DYLIB=ON
    -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra;compiler-rt;lld;lldb;libcxx;libcxxabi"
    -DENABLE_LINKER_BUILD_ID=ON
    -DLLVM_BINUTILS_INCDIR=/usr
    -DLLVM_BUILD_DOCS=ON
    -DLLVM_BUILD_LLVM_DYLIB=ON
    -DLLVM_BUILD_TESTS=ON
    -DLLVM_ENABLE_BINDINGS=OFF
    -DLLVM_ENABLE_FFI=ON
    -DLLVM_ENABLE_RTTI=ON
    -DLLVM_ENABLE_SPHINX=ON
    -DLLVM_HOST_TRIPLE=$CHOST
    -DLLVM_INCLUDE_BENCHMARKS=OFF
    -DLLVM_INSTALL_UTILS=ON
    -DLLVM_LINK_LLVM_DYLIB=ON
    -DLLVM_USE_PERF=ON
    -DLLVM_EXTERNAL_LIT="/usr/clang15/bin/lit"
    -DLLVM_MAIN_SRC_DIR="$srcdir/llvm-$pkgver.src"
    -DSPHINX_WARNINGS_AS_ERRORS=OFF
  )

  cmake ../llvm "${cmake_args[@]}"
  ninja
}

check() {
  cd llvm-project-$pkgver.src/build
  LD_LIBRARY_PATH=$PWD/lib ninja check
}

_python_optimize() {
  python -m compileall "$@"
  python -O -m compileall "$@"
  python -OO -m compileall "$@"
}

package() {
  cd llvm-project-$pkgver.src/build

  DESTDIR="$pkgdir" ninja install

  # Include lit for running lit-based tests in other projects
  pushd ../utils/lit
  python3 setup.py install --root="$pkgdir" -O1
  popd

  # The runtime libraries go into llvm-libs
  mv -f "$pkgdir"/usr/clang15/lib/lib{LLVM,LTO,Remarks}*.so* "$srcdir"
  mv -f "$pkgdir"/usr/clang15/lib/LLVMgold.so "$srcdir"

  if [[ $CARCH == x86_64 ]]; then
    # Needed for multilib (https://bugs.archlinux.org/task/29951)
    # Header stub is taken from Fedora
    mv "$pkgdir/usr/clang15/include/llvm/Config/llvm-config"{,-64}.h
    cp "$srcdir/llvm-config.h" "$pkgdir/usr/clang15/include/llvm/Config/llvm-config.h"
  fi

  # Install libs
  cd "$srcdir"
  install -d "$pkgdir/usr/clang15/lib"
  cp -P \
    "$srcdir"/lib{LLVM,LTO,Remarks}*.so* \
    "$srcdir"/LLVMgold.so \
    "$pkgdir/usr/clang15/lib/"

  # Symlink LLVMgold.so from /usr/clang15/lib/bfd-plugins
  # https://bugs.archlinux.org/task/28479
  install -d "$pkgdir/usr/clang15/lib/bfd-plugins"
  ln -s ../LLVMgold.so "$pkgdir/usr/clang15/lib/bfd-plugins/LLVMgold.so"

  install -Dm644 ../llvm/LICENSE.TXT "$pkgdir/usr/clang15/share/licenses/$pkgname/LICENSE"

  # Move scanbuild-py into site-packages and install Python bindings
  local site_packages=$(python -c "import site; print(site.getsitepackages()[0])")
  install -d "$pkgdir/$site_packages"
  mv "$pkgdir"/usr/clang15/lib/{libear,libscanbuild} "$pkgdir/$site_packages/"
  cp -a ../bindings/python/clang "$pkgdir/$site_packages/"

  # Move analyzer scripts out of /usr/clang15/libexec
  mv "$pkgdir"/usr/clang15/libexec/* "$pkgdir/usr/clang15/lib/clang/"
  rmdir "$pkgdir/usr/clang15/libexec"
  sed -i 's|libexec|lib/clang|' \
    "$pkgdir/usr/clang15/bin/scan-build" \
    "$pkgdir/$site_packages/libscanbuild/analyze.py"

  # Compile Python scripts
  _python_optimize "$pkgdir/usr/clang15/share" "$pkgdir/$site_packages"
}

# vim:set ts=2 sw=2 et:
