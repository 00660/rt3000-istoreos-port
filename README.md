# RT3000 iStoreOS GitHub build benchmark

This repository contains the RT3000 porting patch, the build configuration from the Linux build host, and a manually dispatched GitHub Actions workflow. The workflow applies these inputs to iStoreOS revision `96c7500f01945e2652ff71478aa74afea5503ccd`, verifies the RT3000 profile, iStore app, and Docker exclusions, then records runner resources and build time.

The firmware artifact is for comparison and inspection only. The workflow does not flash a device.
