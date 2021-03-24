---
title: openssl update
date: 2020-03-04 17:04:51
tags:
- openssl
- cmake
- findpackage
- pkg_search_module
---

# openssl 升级至 1.1.1d

## 下载openssl-1.1.1d

``` bash
cd /usr/local/src/
wget https://www.openssl.org/source/openssl-1.1.1d.tar.gz
tar xf openssl-1.1.1d.tar.gz
```

## 编译openssl

``` bash
sudo yum -y install gcc zlib zlib-devel
cd openssl-1.1.1d
./config --prefix=/usr/local/openssl shared zlib
make && make install
```

## 配置openssl-1.1.1d

``` bash
cp /usr/bin/openssl /usr/bin/openssl.old
ln -sf /usr/local/openssl/bin/openssl /usr/bin/openssl
ln -s /usr/local/openssl/lib/libssl.so.1.1 /usr/lib64/libssl.so.1.1
ln -s /usr/local/openssl/lib/libcrypto.so.1.1 /usr/lib64/libcrypto.so.1.1
sudo echo "/usr/local/openssl/lib/" > /etc/ld.so.conf.d/openssl_1.1.1.conf
sudo ldconfig -v 
/usr/bin/openssl version
```

# cmake 引用openssl-1.1.1d

## find_package 引用 openssl-1.1.1d

``` bash
cmake_minimum_required( VERSION 3.8 )

project( test )
set( OPENSSL_ROOT_DIR "/usr/local/openssl" )
set( OPENSSL_INCLUDE_DIR "/usr/local/openssl/include" )

find_package( OpenSSL REQUIRED )
message( "OPENSSL_VERSION: " ${OPENSSL_VERSION} )
message( "OPENSSL_INCLUDE_DIR: " ${OPENSSL_INCLUDE_DIR} )
message( "OPENSSL_LIBRARIES: " ${OPENSSL_LIBRARIES} )
```

## pkg_search_module 引用 openssl-1.1.1d

```bash
cmake_minimum_required( VERSION 3.8 )

project( test )

find_package( PkgConfig REQUIRED )
set(ENV{PKG_CONFIG_PATH}  "/usr/local/openssl/lib/pkgconfig" )
pkg_search_module( LIBCRYPTO REQUIRED libcrypt )
message( "LIBCRYPTO_VERSION: " ${LIBCRYPTO_VERSION} )

pkg_search_module( LIBSSL REQUIRED libssl )
message( "LIBSSL_VERSION: " ${LIBSSL_VERSION} )
```