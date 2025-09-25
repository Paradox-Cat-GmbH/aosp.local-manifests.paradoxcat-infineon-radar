# aosp.local-manifests.paradoxcat-infineon-radar

local_manifests repository that manages all AOSP repositories at Paradox Cat related to Infineon Radars.

## How to use

### Step 1: Get the source code

After syncing any AOSP source tree (tested with LineageOS 21 and 22), get our additions:

```bash
$ cp paradoxcat-infineon-radar.xml [your_aosp]/.repo/local_manifests
$ repo sync -c
```

### Step 2: Obtain Infineon Radar Development Kit and build its libraries from source code

Due to absence of any license terms, we cannot redistribute the Infineon Radar Development Kit.

Download is available after account creation here: [Infineon Developer Center](https://softwaretools.infineon.com/tools/com.ifx.tb.tool.ifxradarsdk)

The only version as of this writing is a Windows installer, which installs:

* Radar Fusion GUI (tool to setup and debug a sensor, including firmware updates)
* Documentation

Follow documentation to obtain all libraries for your desired architecture.
There are some prebuilt libraries, but for linux it only provides:

* x86_64
* arm (32-bit)

In our case, we needed `aarch64`, so we cross-compiled from MacOS.

To see the logs from Infineon libraries, you must build the **debug** versions.

### Step 3: Populate HAL repository with Infineon RDK libraries

* Copy all `*.so` files to `vendor/infineon/hardware/interfaces/radar/prebuilt/libs/`
* Copy all `include` directories to `vendor/infineon/hardware/interfaces/radar/prebuilt/include/`

This is the structure that we expect as of RDK version 3.5.1:

```
vendor/infineon/hardware/interfaces/radar/aidl/prebuilt$ tree
.
├── Android.bp
├── include
│   ├── ifxAdvancedMotionSensing
│   │   └── AdvancedMotionSensing.h
│   ├── ifxAlgo
│   │   ├── 2DMTI.h
│   │   ├── Algo.h
│   │   ├── DBSCAN.h
│   ...
└── libs
    ├── libinfineon_xensiv_android.so
    ├── liblib_avian.so
    ├── libsdk_algo.so
    ├── libsdk_avian.so
    ├── libsdk_base.so
    ├── libsdk_cw.so
    ├── libsdk_fmcw.so
    ├── libsdk_ltr11.so
    ├── libsdk_mimose.so
    ├── libsdk_radar_device_common.so
    ├── libsdk_radar.so
    └── libstrata_shared-d.so

18 directories, 90 files
```

You may need to adjust `vendor/infineon/hardware/interfaces/radar/aidl/prebuilt/Android.bp` depending on the differences in a newer package.

Also, please check if `ifxAvian/DeviceConfig.h` still corresponds to `vendor/infineon/hardware/interfaces/radar/aidl/vendor/infineon/radar/SensorConfig.aidl`.

### Step 4: Add HAL and system service to the device configuration

Locate your `device.mk` file, e.g., `device/samsung/gts4lv-common/gts4lv.mk` and add the following lines to it:

```make
# Infineon Radar HAL
PRODUCT_PACKAGES += vendor.infineon.radar@1.0-service
BOARD_VENDOR_SEPOLICY_DIRS += vendor/infineon/hardware/interfaces/radar/aidl/default/sepolicy

# Infineon Radar Service and Manager
PRODUCT_PACKAGES += \
    InfineonRadarService \
    InfineonRadarManager
```

### Step 5: Ensure that USB permissions are set correctly

Locate your `ueventd.rc` overlay file, e.g. `device/samsung/gts4lv-common/init/ueventd.qcom.rc`, and add the following lines to it:

```make
# Infineon sensor
/dev/ttyACM*    0660    system  system
```

To verify if this change has effect, verify permissions after starting your image and connecting the sensor via USB:

```bash
$ adb shell ls -l /dev/ttyACM0
```

### Step 6: Add Infineon HAL to your Device Framework Compatibility Matrix

Locate your `framework_compatibility_matrix.xml`, e.g., `device/samsung/gts4lv-common/framework_compatibility_matrix.xml` and add the following lines to it:

```xml
<hal format="aidl" optional="true">
    <name>vendor.infineon.radar</name>
    <version>1</version>
    <interface>
        <name>IRadarSdk</name>
        <instance>default</instance>
    </interface>
</hal>
```

Device Framework Compatibility Matrix (FCM) is saying what framework (our InfineonRadarService on system partition) expects from device (vendor.infineon.radar HAL on vendor partition).
See https://source.android.com/docs/core/architecture/vintf/comp-matrices#framework-compatibility-matrix

### Step 7: Build as usual

```bash
$ source build/envsetup.sh 
$ # breakfast/lunch/..., e.g., breakfast gts4lv
$ m
```

### Troubleshooting

During HAL development, you may need to set `RELEASE_AIDL_USE_UNFROZEN` soong build variable to true. Otherwise, your HAL may keep crashing with following errors:

```
INFO: Removing HAL from the manifest because it is declaring V1 of a new unfrozen interface which is not allowed in this release configuration: vendor.infineon.radar
...
servicemanager: Could not find vendor.infineon.radar.IRadarSdk/default in the VINTF manifest. No alternative instances declared in VINTF.
```

Verify if it was set in `out/soong/soong.lineage_gts4lv.variables` or similar.

### Demo application

To see the radar HAL and service in action, you can build a non-system application: https://github.com/Paradox-Cat-GmbH/RadarDemoApp

This application visualizes the data from the radar sensor as range doppler images.
