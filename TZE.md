# Instructions for TZE testing

## Zcash TZE PR

PR: https://github.com/zcash/zcash/pull/4480  

repo: https://github.com/nuttycom/zcash.git  
branch: zip-tzes  

## Build time requirements

### Pre build script

https://github.com/zcash/zcash/pull/4464/files  

curl -LO https://raw.githubusercontent.com/zcash/zcash/f085f5e232d4a04169fc35c150d5c8dc32b58a9b/zcutil/tnbox/tnbox.py


### Confirm changes to
chainparams.h
consesnsus.cpp

### Build command 
CONFIGURE_FLAGS="--enable-online-rust" ./zcutil/build.sh -j$(nproc)  

