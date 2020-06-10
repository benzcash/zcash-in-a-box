# Instructions for TZE testing

## Zcash TZE PR

PR: https://github.com/zcash/zcash/pull/4480  

repo: https://github.com/nuttycom/zcash.git  
branch: zip-tzes  

## Build time requirements

### Pre build script

https://github.com/zcash/zcash/pull/4464/files  

### Confirm changes to
chainparams.h

### Build command 
CONFIGURE_FLAGS="--enable-online-rust" ./zcutil/build.sh -j$(nproc)  

