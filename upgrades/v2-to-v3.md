# v2 to v3 Configuration changes

## Components reference
### **op-succinct-proposer**
Image: [ghcr.io/agglayer/op-succinct/op-succinct:v3.1.0-agglayer](https://github.com/agglayer/op-succinct/pkgs/container/op-succinct%2Fop-succinct/515633556?tag=v3.1.0-agglayer)

What's new:
- [`OP_SUCCINCT_CONFIG_NAME`](https://succinctlabs.github.io/op-succinct/proposer.html#optional-environment-variables) environment variable set to the value of the config name (`_configName` parameter)

```diff
+    OP_SUCCINCT_CONFIG_NAME: "v3.1.0-agglayer"
```

### **aggkit-prover**
Image: [ghcr.io/agglayer/aggkit-prover:1.4.2](https://github.com/agglayer/provers/pkgs/container/aggkit-prover/530717765?tag=1.4.2)

What's new:
- Config change
```diff
     [aggchain-proof-service.aggchain-proof-builder]
     network-id = 0
+    proving-timeout = "1h"

     [aggchain-proof-service.aggchain-proof-builder.primary-prover.network-prover]
     proving-timeout = "1h"

-    [aggchain-proof-service.aggchain-proof-builder.proving-timeout]
-    secs = 3600
-    nanos = 0
```

### **aggsender**
- Image: [ghcr.io/agglayer/aggkit:0.7.1](https://github.com/agglayer/aggkit/pkgs/container/aggkit/549774904?tag=0.7.1)
- Config reference: https://github.com/agglayer/aggkit/blob/v0.7.1/config/default.go
- Release notes: https://github.com/agglayer/aggkit/releases/tag/v0.7.1

What's required since 0.5.4:
- Config change
```diff
     [AggSender]
-    BlockFinality="FinalizedBlock"
-    [AggSender.AgglayerClient]
+    [AggSender.AgglayerClient.GRPC]
     URL = https://
     UseTLS = true

+    [AggSender.OptimisticModeConfig]
+        SovereignRollupAddr = "0x"
+        # By default use the same key that aggsender signs certs
+        TrustedSequencerKey = "0x"
+        OpNodeURL = "https://"
+        RequireKeyMatchTrustedSequencer = true
```
