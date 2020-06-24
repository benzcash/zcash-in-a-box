# How to ensure block snapshot from testnet will load on a BoxedTestnet

Assume a currently running zcashd node (ZCASHD1) on testnet synced to TestnetCurrentHeight.

Stop the instance.

Copy the ZcashDataDir/blocks/* and ZcashDataDir/chainstate/* contents to BoxedZcashDataDir, which is otherwise empty.

From the source code used to build ZCASHD1, **modify** the pchMessageStart values
- src/chainparams.cpp 
```
    class CTestNetParams : public CChainParams {
        public:
            CTestNetParams() {
                pchMessageStart[0] = 0xfa;
                pchMessageStart[1] = 0x1a;
                pchMessageStart[2] = 0xf9;
                pchMessageStart[3] = 0xbf;
            }
```

Build and run the new binary with BoxedZcashDataDir.

