# Сборка audiowaveform 1.10.1 под все платформы

## Требования

- **audiowfbuild** — Incus контейнер Ubuntu ARM64 (audiowfbuild.labelstage.com)
- **Mac** — Apple Silicon для darwin-arm64
- SSH доступ к контейнеру настроен в `~/.ssh/config`

## 1. Linux ARM64 (нативная сборка на контейнере)

```bash
ssh audiowfbuild
```

### Установка зависимостей
```bash
apt update && apt install -y build-essential cmake git \
  libboost-program-options-dev libboost-filesystem-dev libboost-regex-dev \
  libgd-dev libpng-dev zlib1g-dev
```

### Сборка статических библиотек (без MPEG)
```bash
mkdir -p /root/static-libs && cd /root/static-libs

# libogg
curl -sL https://github.com/xiph/ogg/releases/download/v1.3.5/libogg-1.3.5.tar.xz | tar xJ
cd libogg-1.3.5 && ./configure --prefix=/root/static-libs/install --disable-shared --enable-static && make -j$(nproc) && make install && cd ..

# libvorbis
curl -sL https://github.com/xiph/vorbis/releases/download/v1.3.7/libvorbis-1.3.7.tar.xz | tar xJ
cd libvorbis-1.3.7 && ./configure --prefix=/root/static-libs/install --disable-shared --enable-static --with-ogg=/root/static-libs/install && make -j$(nproc) && make install && cd ..

# flac
curl -sL https://github.com/xiph/flac/releases/download/1.4.3/flac-1.4.3.tar.xz | tar xJ
cd flac-1.4.3 && ./configure --prefix=/root/static-libs/install --disable-shared --enable-static --with-ogg=/root/static-libs/install && make -j$(nproc) && make install && cd ..

# opus
curl -sL https://github.com/xiph/opus/releases/download/v1.4/opus-1.4.tar.gz | tar xz
cd opus-1.4 && ./configure --prefix=/root/static-libs/install --disable-shared --enable-static && make -j$(nproc) && make install && cd ..

# libmad
curl -sL https://downloads.sourceforge.net/mad/libmad-0.15.1b.tar.gz | tar xz
cd libmad-0.15.1b && autoreconf -fi && ./configure --prefix=/root/static-libs/install --disable-shared --enable-static && make -j$(nproc) && make install && cd ..

# libid3tag
curl -sL https://downloads.sourceforge.net/mad/libid3tag-0.15.1b.tar.gz | tar xz
cd libid3tag-0.15.1b && autoreconf -fi && ./configure --prefix=/root/static-libs/install --disable-shared --enable-static && make -j$(nproc) && make install && cd ..

# zlib
curl -sL https://zlib.net/zlib-1.3.1.tar.gz | tar xz
cd zlib-1.3.1 && ./configure --prefix=/root/static-libs/install --static && make -j$(nproc) && make install && cd ..

# libpng
curl -sL https://download.sourceforge.net/libpng/libpng-1.6.43.tar.xz | tar xJ
cd libpng-1.6.43 && ./configure --prefix=/root/static-libs/install --disable-shared --enable-static CPPFLAGS="-I/root/static-libs/install/include" LDFLAGS="-L/root/static-libs/install/lib" && make -j$(nproc) && make install && cd ..

# libgd
curl -sL https://github.com/libgd/libgd/releases/download/gd-2.3.3/libgd-2.3.3.tar.xz | tar xJ
cd libgd-2.3.3 && ./configure --prefix=/root/static-libs/install --disable-shared --enable-static --with-png=/root/static-libs/install --with-zlib=/root/static-libs/install --without-jpeg --without-tiff --without-webp --without-freetype --without-fontconfig && make -j$(nproc) && make install && cd ..

# libsndfile (БЕЗ MPEG!)
curl -sL https://github.com/libsndfile/libsndfile/releases/download/1.2.2/libsndfile-1.2.2.tar.xz | tar xJ
cd libsndfile-1.2.2 && mkdir build && cd build && cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_SHARED_LIBS=OFF \
  -DENABLE_MPEG=OFF \
  -DCMAKE_PREFIX_PATH=/root/static-libs/install \
  -DCMAKE_INSTALL_PREFIX=/root/static-libs/install \
  -DBUILD_TESTING=OFF -DBUILD_PROGRAMS=OFF -DBUILD_EXAMPLES=OFF && \
make -j$(nproc) && make install && cd ../..

# boost (static)
curl -sL https://boostorg.jfrog.io/artifactory/main/release/1.83.0/source/boost_1_83_0.tar.gz | tar xz
cd boost_1_83_0 && ./bootstrap.sh --prefix=/root/static-libs/install --with-libraries=program_options,filesystem,regex && ./b2 install link=static runtime-link=static -j$(nproc) && cd ..
```

### Сборка audiowaveform
```bash
cd /root && git clone https://github.com/bbc/audiowaveform.git audiowaveform-src
cd audiowaveform-src && git checkout 1.10.1 && mkdir build && cd build

cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_STATIC=1 \
  -DENABLE_TESTS=OFF \
  -DCMAKE_PREFIX_PATH=/root/static-libs/install \
  -DCMAKE_FIND_LIBRARY_SUFFIXES=".a" \
  -DCMAKE_EXE_LINKER_FLAGS="-static-libgcc -static-libstdc++"

make -j$(nproc)

# Проверка
ldd audiowaveform  # должны быть только libc, libm, libpthread, libdl
./audiowaveform --version

# Копирование
mkdir -p /root/binaries
cp audiowaveform /root/binaries/audiowaveform-linux-arm64
```

---

## 2. Linux x64 (кросс-компиляция на ARM64 контейнере)

```bash
ssh audiowfbuild
```

### Установка кросс-компилятора
```bash
apt install -y gcc-x86-64-linux-gnu g++-x86-64-linux-gnu
```

### Сборка x64 статических библиотек
```bash
mkdir -p /root/x64-build && cd /root/x64-build
export CC=x86_64-linux-gnu-gcc
export CXX=x86_64-linux-gnu-g++
export AR=x86_64-linux-gnu-ar
export RANLIB=x86_64-linux-gnu-ranlib
export HOST=x86_64-linux-gnu

# Повторить все библиотеки как выше, но с --host=$HOST и prefix=/root/x64-build/install
# ... (аналогично ARM64, но с кросс-компилятором)

# boost для x64
cd boost_1_83_0
cat > user-config.jam << 'EOF'
using gcc : x64 : x86_64-linux-gnu-g++ ;
EOF
./b2 install --prefix=/root/x64-build/install link=static runtime-link=static toolset=gcc-x64 -j$(nproc)
```

### Сборка audiowaveform x64
```bash
cd /root/audiowaveform-src && rm -rf build && mkdir build && cd build

cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_STATIC=1 \
  -DENABLE_TESTS=OFF \
  -DCMAKE_C_COMPILER=x86_64-linux-gnu-gcc \
  -DCMAKE_CXX_COMPILER=x86_64-linux-gnu-g++ \
  -DCMAKE_PREFIX_PATH=/root/x64-build/install \
  -DCMAKE_FIND_LIBRARY_SUFFIXES=".a" \
  -DCMAKE_EXE_LINKER_FLAGS="-static-libgcc -static-libstdc++"

make -j$(nproc)

cp audiowaveform /root/binaries/audiowaveform-linux-x64
```

---

## 3. macOS ARM64 (на Mac)

### Установка зависимостей
```bash
brew install boost libsndfile libgd libpng zlib libmad libid3tag
```

### Сборка статических libmad и libid3tag
```bash
mkdir -p /tmp/static-deps && cd /tmp/static-deps

# libmad
curl -sL https://downloads.sourceforge.net/mad/libmad-0.15.1b.tar.gz | tar xz
cd libmad-0.15.1b && autoreconf -fi
./configure --prefix=/tmp/static-deps --disable-shared --enable-static
# Убрать GCC-специфичные флаги
sed -i '' 's/-fforce-addr//g; s/-fthread-jumps//g; s/-fcse-follow-jumps//g; s/-fcse-skip-blocks//g; s/-fexpensive-optimizations//g; s/-fregmove//g; s/-fschedule-insns2//g' Makefile
make -j$(sysctl -n hw.ncpu) && make install && cd ..

# libid3tag
curl -sL https://downloads.sourceforge.net/mad/libid3tag-0.15.1b.tar.gz | tar xz
cd libid3tag-0.15.1b && autoreconf -fi
./configure --prefix=/tmp/static-deps --disable-shared --enable-static
make -j$(sysctl -n hw.ncpu) && make install && cd ..

# libsndfile без MPEG
curl -sL https://github.com/libsndfile/libsndfile/releases/download/1.2.2/libsndfile-1.2.2.tar.xz | tar xJ
cd libsndfile-1.2.2 && mkdir build && cd build
cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_SHARED_LIBS=OFF \
  -DENABLE_MPEG=OFF \
  -DCMAKE_PREFIX_PATH=/opt/homebrew \
  -DCMAKE_INSTALL_PREFIX=/tmp/static-deps \
  -DBUILD_TESTING=OFF -DBUILD_PROGRAMS=OFF -DBUILD_EXAMPLES=OFF \
  -DCMAKE_POLICY_VERSION_MINIMUM=3.5
make -j$(sysctl -n hw.ncpu) && make install && cd ../..
```

### Сборка audiowaveform
```bash
git clone https://github.com/bbc/audiowaveform.git /tmp/audiowaveform-mac
cd /tmp/audiowaveform-mac && git checkout 1.10.1

# Убрать boost_system из CMakeLists.txt (он теперь header-only)
sed -i '' 's/program_options filesystem regex system/program_options filesystem regex/' CMakeLists.txt

mkdir build && cd build
cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_STATIC=1 \
  -DENABLE_TESTS=OFF \
  -DCMAKE_PREFIX_PATH="/tmp/static-deps;/opt/homebrew;/opt/homebrew/opt/zlib" \
  -DZLIB_ROOT=/opt/homebrew/opt/zlib \
  -DCMAKE_POLICY_VERSION_MINIMUM=3.5 \
  -DBoost_NO_BOOST_CMAKE=ON

make -j$(sysctl -n hw.ncpu)

# Проверка - должны быть только системные либы
otool -L audiowaveform
# /usr/lib/libc++.1.dylib
# /usr/lib/libSystem.B.dylib

./audiowaveform --version

cp audiowaveform ~/binaries/audiowaveform-darwin-arm64
```

---

## 4. Windows x64 (скачать официальный билд)

```bash
gh release download 1.10.1 --repo bbc/audiowaveform --pattern "audiowaveform-1.10.1.win64.zip"
unzip audiowaveform-1.10.1.win64.zip
mv audiowaveform.exe ~/binaries/audiowaveform-win32-x64.exe
```

---

## 5. Обновление npm пакета

```bash
cd /path/to/audiowaveform-installer

# Копировать бинарники
cp ~/binaries/audiowaveform-linux-arm64 platforms/linux-arm64/audiowaveform
cp ~/binaries/audiowaveform-linux-x64 platforms/linux-x64/audiowaveform
cp ~/binaries/audiowaveform-darwin-arm64 platforms/darwin-arm64/audiowaveform
cp ~/binaries/audiowaveform-win32-x64.exe platforms/win32-x64/audiowaveform.exe

# Обновить версии в package.json (во всех)
# Публиковать каждую платформу отдельно:
cd platforms/linux-arm64 && npm publish --access public
cd platforms/linux-x64 && npm publish --access public
cd platforms/darwin-arm64 && npm publish --access public
cd platforms/win32-x64 && npm publish --access public

# Основной пакет
cd ../.. && npm publish --access public
```

---

## Проверка бинарников

```bash
# Linux - должны быть только glibc зависимости
ldd audiowaveform-linux-arm64
ldd audiowaveform-linux-x64

# macOS - только системные
otool -L audiowaveform-darwin-arm64

# Запуск
./audiowaveform --version
```
