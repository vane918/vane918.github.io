---
layout: post
title:  "FireFly RK3399 Android8.1系统Camera HAL层内存泄漏问题分析"
date:   2023-02-22 00:00:00
catalog:  true
tags:
    - android
    - Camera HAL
    - memory leak
---

## 1. 问题背景

在FireFly RK3399 开发板搭载Android8.1系统，插入UVC设备做开关压力测试，发现压测1万多次后，UVC设备打开失败，log显示申请内存失败。dumpsys meminfo发现Camera HAL层已泄漏2Gb。

## 2. 问题分析

### 2.1 dumpsys meminfo初步分析

dumpsys meminfo信息如下：

```
Total PSS by process:
  2,248,200K: android.hardware.camera.provider@2.4-service (pid 238)
     71,868K: system (pid 473)
```

初步知道是android.hardware.camera.provider@2.4-service进程内存泄漏，该进程是Android Camera HAL层。在Android源码的hardware目录下使用命令“grep -nr --color "android.hardware.camera.provider@2.4-service" 搜索该进程所在位置如下：

hardware/interfaces/camera/provider/2.4/default/Android.bp

```makefile
...
cc_binary {
    name: "android.hardware.camera.provider@2.4-service",
    defaults: ["hidl_defaults"],
    proprietary: true,
    relative_install_path: "hw",
    srcs: ["service.cpp"],
    compile_multilib: "32",
    init_rc: ["android.hardware.camera.provider@2.4-service.rc"],
    shared_libs: [
        "libhidlbase",
        "libhidltransport",
        "libbinder",
        "liblog",
        "libutils",
        "android.hardware.camera.device@1.0",
        "android.hardware.camera.device@3.2",
        "android.hardware.camera.device@3.3",
        "android.hardware.camera.provider@2.4",
        "android.hardware.camera.common@1.0",
    ],
}
```

RK3399开发板对应的源码在下面的路径：hardware/rockchip/camera/CameraHal

### 2.2 分析logcat

找到内存泄漏进程的源码后，为了进一步缩小范围发生内存泄漏的代码，首先分析每次UVC open-close时的log，如下：

打开UVC的log：

```
10:03:32.847 486-548/system_process D/UsbDeviceManager: usb camera removed
10:03:32.849 486-561/system_process I/EventHub: Removing device USB 2.0 Camera due to epoll hang-up event.
10:03:32.849 486-561/system_process I/EventHub: Removed device: path=/dev/input/event3 name=USB 2.0 Camera id=815 fd=109 classes=0x80000401
10:03:32.855 360-466/? W/CameraProviderManager: addProviderLocked: Camera provider HAL with name 'legacy/0' already registered
10:03:32.855 360-466/? I/CameraProviderManager: Connecting to new camera provider: legacy/0, isRemote? 1
10:03:32.855 360-414/? W/CameraProviderManager: addProviderLocked: Camera provider HAL with name 'legacy/0' already registered
10:03:32.855 235-1889/? D/CamProvider@2.4-impl: usb camera hotpluged,reinit camera provider
10:03:32.855 235-1889/? D/vndksupport: Loading /vendor/lib/hw/camera.rk30board.so from current namespace instead of sphal namespace.
10:03:32.859 235-1889/? D/CameraHal: createInstance(1015): open xml file(/vendor/etc/cam_board.xml) success
10:03:32.859 235-1889/? E/CameraHal: cam_board.xml version(v0x0.0x0.0x0) != xml parser version(v0x0.0xf.0x0)
10:03:32.859 235-1889/? D/CameraHal:  Cam_board.xml Version Check: 
10:03:32.859 235-1889/? D/CameraHal:     /vendor/etc/cam_board.xml : v0.0xf.0
10:03:32.859 235-1889/? D/CameraHal:     CameraHal_board_xml_parser: v0x0.0xf.0x0
10:03:32.859 235-1889/? D/CameraHal: ParserSensorInfo(170): SensorName(OV13850)
10:03:32.859 235-1889/? D/CameraHal: ParserSensorInfo(176): lensName(50013A1)
...
10:03:32.881 235-1889/? D/CameraHal: number of camdevice (6)
10:03:32.881 235-1889/? D/CameraHal: now DV size(7)
10:03:32.881 235-1889/? D/CameraHal: camera_get_number_of_cameras(780): board profiles cam num 6
10:03:32.881 235-1889/? E/CameraHal: camera_get_number_of_cameras(788): Open /dev/video0 failed! strr: No such file or directory
10:03:32.881 235-1889/? E/CameraHal: camera_get_number_of_cameras(798): vanelst connectUvc:0, nCamDev:6
10:03:32.881 235-1889/? D/CameraHal: camera_get_number_of_cameras(808): load sensor name(OV13850) connect 0
10:03:32.881 235-1889/? D/CameraHal: camera_get_number_of_cameras(808): load sensor name(OV13850) connect 0
10:03:32.881 235-1889/? D/CameraHal: camera_get_number_of_cameras(808): load sensor name(OV8858) connect 0
10:03:32.881 235-1889/? D/CameraHal: camera_get_number_of_cameras(808): load sensor name(OV8858) connect 0
10:03:32.881 235-1889/? D/CameraHal: camera_get_number_of_cameras(808): load sensor name(OV5640) connect 0
10:03:32.881 235-1889/? D/CameraHal: camera_get_number_of_cameras(808): load sensor name(S5K4EC) connect 0
10:03:32.881 235-1889/? E/CameraHal: camera_get_number_of_cameras(867): Open /dev/video0 failed! strr: No such file or directory
10:03:32.882 235-1889/? D/CameraHal: camera_get_number_of_cameras(1271): camera_get_number_of_cameras(1271): Current board have 0 cameras attached.
10:03:32.882 235-1889/? I/CamProvider@2.4-impl: Loaded "RK29_ICS_CameraHal_Module" camera module
10:03:32.882 235-1889/? I/CamProvider@2.4-impl: setUpVendorTags: No vendor tags defined for this device.
10:03:32.883 235-1889/? D/CameraHal: createInstance(1015): open xml file(/vendor/etc/cam_board.xml) success
10:03:32.884 235-1889/? E/CameraHal: cam_board.xml version(v0x0.0x0.0x0) != xml parser version(v0x0.0xf.0x0)
10:03:32.884 235-1889/? D/CameraHal:  Cam_board.xml Version Check: 
10:03:32.884 235-1889/? D/CameraHal:     /vendor/etc/cam_board.xml : v0.0xf.0
10:03:32.884 235-1889/? D/CameraHal:     CameraHal_board_xml_parser: v0x0.0xf.0x0
10:03:32.884 235-1889/? D/CameraHal: ParserSensorInfo(170): SensorName(OV13850)
10:03:32.884 235-1889/? D/CameraHal: ParserSensorInfo(176): lensName(50013A1)
...
```

关闭UVC的log：

```
10:03:34.942 486-548/system_process D/UsbDeviceManager: usb camera added
10:03:34.951 486-561/system_process D/EventHub: No input device configuration file found for device 'USB 2.0 Camera'.
10:03:34.953 360-466/? W/CameraProviderManager: addProviderLocked: Camera provider HAL with name 'legacy/0' already registered
10:03:34.953 360-466/? I/CameraProviderManager: Connecting to new camera provider: legacy/0, isRemote? 1
10:03:34.953 360-414/? W/CameraProviderManager: addProviderLocked: Camera provider HAL with name 'legacy/0' already registered
10:03:34.954 235-1958/? D/CamProvider@2.4-impl: usb camera hotpluged,reinit camera provider
10:03:34.954 235-1958/? D/vndksupport: Loading /vendor/lib/hw/camera.rk30board.so from current namespace instead of sphal namespace.
10:03:34.959 486-561/system_process I/EventHub: New device: id=816, fd=109, path='/dev/input/event3', name='USB 2.0 Camera', classes=0x80000401, configuration='', keyLayout='/system/usr/keylayout/Generic.kl', keyCharacterMap='/system/usr/keychars/Generic.kcm', builtinKeyboard=false, 
10:03:34.959 486-561/system_process I/InputReader: Device added: id=816, name='USB 2.0 Camera', sources=0x00002103
10:03:34.962 235-1958/? D/CameraHal: createInstance(1015): open xml file(/vendor/etc/cam_board.xml) success
10:03:34.962 235-1958/? E/CameraHal: cam_board.xml version(v0x0.0x0.0x0) != xml parser version(v0x0.0xf.0x0)
10:03:34.962 235-1958/? D/CameraHal:  Cam_board.xml Version Check: 
10:03:34.962 235-1958/? D/CameraHal:     /vendor/etc/cam_board.xml : v0.0xf.0
10:03:34.962 235-1958/? D/CameraHal:     CameraHal_board_xml_parser: v0x0.0xf.0x0
10:03:34.963 235-1958/? D/CameraHal: ParserSensorInfo(170): SensorName(OV13850)
10:03:34.963 235-1958/? D/CameraHal: ParserSensorInfo(176): lensName(50013A1)
...
10:03:34.991 235-1958/? D/CameraHal: number of camdevice (6)
10:03:34.991 235-1958/? D/CameraHal: now DV size(7)
10:03:34.991 235-1958/? D/CameraHal: camera_get_number_of_cameras(780): board profiles cam num 6
10:03:34.991 235-1958/? D/CameraHal: camera_get_number_of_cameras(792): Open /dev/video0 success!
10:03:34.993 235-1958/? E/CameraHal: camera_get_number_of_cameras(788): Open /dev/video1 failed! strr: No such file or directory
10:03:34.993 235-1958/? E/CameraHal: camera_get_number_of_cameras(798): vanelst connectUvc:1, nCamDev:6
10:03:34.993 235-1958/? D/CameraHal: camera_get_number_of_cameras(870): Open /dev/video0 success!
10:03:34.996 235-1958/? E/CameraHal: camera_get_number_of_cameras(867): Open /dev/video1 failed! strr: No such file or directory
10:03:34.996 235-1958/? D/CameraHal: camera_get_number_of_cameras(1271): camera_get_number_of_cameras(1271): Current board have 1 cameras attached.
...
```

使用libusb open UVC设备后，UsbDeviceManager会打印“usb camera removed”，close UVC设备后，打印“usb camera added”。

在open UVC时，libusb调用libusb_claim_interface接口后，会将该UVC设备的所有权从V4L2驱动中转移到libusb中，因此/dev/video节点会消失，UsbDeviceManager会打印“usb camera removed”。而在close UVC时，libusb调用libusb_release_interface接口，将对UVC设备的控制权归还给V4L2，V4L2驱动重新生成/dev/video节点。

我们可以从UsbDeviceManager的打印usb camera removed入手。

### 2.3 源码分析

frameworks/base/services/usb/java/com/android/server/usb/UsbDeviceManager.java

```java
/*
 *Listens for uevent messages from the kernel to monitor the USB state
 */
private final UEventObserver mUEventObserver = new UEventObserver() {
    @Override
    public void onUEvent(UEventObserver.UEvent event) {
        if (DEBUG) Slog.v(TAG, "USB UEVENT: " + event.toString());

        String subSystem = event.get("SUBSYSTEM");
        String devPath = event.get("DEVPATH");

        if (devPath != null && devPath.contains("/devices/platform")) {
            if ("video4linux".equals(subSystem)) {
                Intent intent = new Intent(Intent.ACTION_USB_CAMERA);
                String action = event.get("ACTION");
                ...
                if ("remove".equals(action)){
                    Slog.d(TAG,"usb camera removed");
                    intent.setFlags(Intent.FLAG_USB_CAMERA_REMOVE);
                    SystemProperties.set("persist.sys.usbcamera.status","remove");
                } else if ("add".equals(action)) {
                    Slog.d(TAG,"usb camera added");
                    intent.setFlags(Intent.FLAG_USB_CAMERA_ADD);
                    SystemProperties.set("persist.sys.usbcamera.status","add");
                }

                int num = android.hardware.Camera.getNumberOfCameras();
                mContext.sendBroadcast(intent);
                SystemProperties.set("persist.sys.usbcamera.status","");
                Slog.d(TAG,"usb camera num="+num);
            }
        }
...
```

代码显示收到内核USB的uevent messages后作出的动作。无论是USB removed还是added都会执行android.hardware.Camera.getNumberOfCameras()方法。

frameworks/base/core/java/android/hardware/Camera.java

```java
/**
 * Returns the number of physical cameras available on this device.
 *
 * @return total number of accessible camera devices, or 0 if there are no
 *   cameras or an error was encountered enumerating them.
 */
public native static int getNumberOfCameras();
```

frameworks/base/core/jni/android_hardware_Camera.cpp

```cpp
static jint android_hardware_Camera_getNumberOfCameras(JNIEnv *env, jobject thiz)
{
    return Camera::getNumberOfCameras();
}
```

frameworks/av/camera/CameraBase.cpp

```cpp
template <typename TCam, typename TCamTraits>
int CameraBase<TCam, TCamTraits>::getNumberOfCameras() {
    const sp<::android::hardware::ICameraService> cs = getCameraService();

    if (!cs.get()) {
        // as required by the public Java APIs
        return 0;
    }
    int32_t count;
    binder::Status res = cs->getNumberOfCameras(
            ::android::hardware::ICameraService::CAMERA_TYPE_BACKWARD_COMPATIBLE,
            &count);
    ...
}
```

frameworks/av/services/camera/libcameraservice/CameraService.cpp

```cpp
Status CameraService::getNumberOfCameras(int32_t type, int32_t* numCameras) {
    ATRACE_CALL();
    Mutex::Autolock l(mServiceLock);
    char value[PROPERTY_VALUE_MAX];
    switch (type) {
        case CAMERA_TYPE_BACKWARD_COMPATIBLE:
            memset(value, 0, sizeof(value));
            property_get("persist.sys.usbcamera.status", value, "");
            if((strcmp(value, "add") == 0)||(strcmp(value, "remove") == 0)){
                //property_set("persist.sys.usbcamera.status", "");
                mCameraProviderManager->updateCameraCount();
                mNumberOfNormalCameras = mCameraProviderManager->getAPI1CompatibleCameraCount();
            }
            *numCameras = mNumberOfNormalCameras;
            ALOGD("CameraService mNumberOfNormalCameras:%d\n", mNumberOfNormalCameras);
            break;
...
}
```

frameworks/av/services/camera/libcameraservice/common/CameraProviderManager.cpp

```cpp
int CameraProviderManager::updateCameraCount(){
    const std::string& newProvider = kLegacyProviderName;
    sp<provider::V2_4::ICameraProvider> interface;
    sp<StatusListener> listener = getStatusListener();
    ServiceInteractionProxy *proxy = &sHardwareServiceInteractionProxy;

    interface = mServiceProxy->getService(newProvider);
    status_t res = OK;
    CameraProviderManager::initialize(listener, proxy);
    sp<ProviderInfo> providerInfo = mProviders.front();
    if (providerInfo != NULL) {
        res = providerInfo->initialize();
    }
    if (res != OK) {
        return res;
    }
    return 0;
}

status_t CameraProviderManager::ProviderInfo::initialize() {
    ...
    // Get initial list of camera devices, if any
    std::vector<std::string> devices;
    hardware::Return<void> ret = mInterface->getCameraIdList([&status, &devices](
            Status idStatus,
            const hardware::hidl_vec<hardware::hidl_string>& cameraDeviceNames) {
        status = idStatus;
        if (status == Status::OK) {
            for (size_t i = 0; i < cameraDeviceNames.size(); i++) {
                devices.push_back(cameraDeviceNames[i]);
            }
        } });
...
}
```

CameraProviderManager::ProviderInfo::initialize()函数首先会通过HIDL跨进程通信方式与Camera HAL层通信，调用mInterface->getCameraIdList，对应的HAL层代码如下：

hardware/interfaces/camera/provider/2.4/default/CameraProvider.cpp

```cpp
Return<void> CameraProvider::getCameraIdList(getCameraIdList_cb _hidl_cb)  {
    std::vector<hidl_string> deviceNameList;
    char value[PROPERTY_VALUE_MAX];

    memset(value, 0, sizeof(value));
    property_get("persist.sys.usbcamera.status", value, "");
    if((strcmp(value, "add") == 0)||(strcmp(value, "remove") == 0)){
        ALOGD("usb camera hotpluged,reinit camera provider");
        mModule.clear();
        deviceNameList.clear();
        initialize();
    }
    for (auto const& deviceNamePair : mCameraDeviceNames) {
        if (mCameraStatusMap[deviceNamePair.first] == CAMERA_DEVICE_STATUS_PRESENT) {
            deviceNameList.push_back(deviceNamePair.second);
        }
    }
    hidl_vec<hidl_string> hidlDeviceNameList(deviceNameList);
    _hidl_cb(Status::OK, hidlDeviceNameList);
    return Void();
}

bool CameraProvider::initialize() {
    camera_module_t *rawModule;
    int err = hw_get_module(CAMERA_HARDWARE_MODULE_ID,
            (const hw_module_t **)&rawModule);
    if (err < 0) {
        ALOGE("Could not load camera HAL module: %d (%s)", err, strerror(-err));
        return true;
    }

    mModule = new CameraModule(rawModule);
    err = mModule->init();
    if (err != OK) {
        ALOGE("Could not initialize camera HAL module: %d (%s)", err, strerror(-err));
        mModule.clear();
        return true;
    }
    ALOGI("Loaded \"%s\" camera module", mModule->getModuleName());
    ...
    mCameraDeviceNames.clear();
    mNumberOfLegacyCameras = mModule->getNumberOfCameras();
    ...
}
```

最终CameraProvider.cpp的CameraProvider::initialize()函数掉了HAL层的mModule->getNumberOfCameras()。getNumberOfCameras()函数是HAL层接口函数，在下面的头文件中声明hardware/libhardware/include/hardware/camera_common.h

```cpp
    /**
     * get_number_of_cameras:
     *
     * Returns the number of camera devices accessible through the camera
     * module.  The camera devices are numbered 0 through N-1, where N is the
     * value returned by this call. The name of the camera device for open() is
     * simply the number converted to a string. That is, "0" for camera ID 0,
     * "1" for camera ID 1.
     *
     * Version information (based on camera_module_t.common.module_api_version):
     *
     * CAMERA_MODULE_API_VERSION_2_3 or lower:
     *
     *   The value here must be static, and cannot change after the first call
     *   to this method.
     *
     * CAMERA_MODULE_API_VERSION_2_4 or higher:
     *
     *   The value here must be static, and must count only built-in cameras,
     *   which have CAMERA_FACING_BACK or CAMERA_FACING_FRONT camera facing values
     *   (camera_info.facing). The HAL must not include the external cameras
     *   (camera_info.facing == CAMERA_FACING_EXTERNAL) into the return value
     *   of this call. Frameworks will use camera_device_status_change callback
     *   to manage number of external cameras.
     */
    int (*get_number_of_cameras)(void);
```

RK平台对应的Camera HAL层代码在hardware/rockchip/camera/CameraHal目录下，get_number_of_cameras函数的具体实现在下面的文件中

hardware/rockchip/camera/CameraHal/CameraHal_Module.cpp

```cpp
camera_module_t HAL_MODULE_INFO_SYM = {
    common: {
         tag: HARDWARE_MODULE_TAG,
         version_major: ((CONFIG_CAMERAHAL_VERSION&0xffff00)>>16),
         version_minor: CONFIG_CAMERAHAL_VERSION&0xff,
         id: CAMERA_HARDWARE_MODULE_ID,
         name: CAMERA_MODULE_NAME,
         author: "RockChip",
         methods: &camera_module_methods,
         dso: NULL, /* remove compilation warnings */
         reserved: {0}, /* remove compilation warnings */
    },
    get_number_of_cameras: camera_get_number_of_cameras,
    get_camera_info: camera_get_camera_info,
    set_callbacks:NULL,
    get_vendor_tag_ops:NULL,
#if defined(ANDROID_5_X)
    open_legacy:NULL,
#endif
#if defined(ANDROID_6_X) || defined(ANDROID_7_X) || defined(ANDROID_8_X)
    set_torch_mode:NULL,
    init:NULL,
#endif
    reserved: {0}   
};
```

### 2.4 CameraHal_Module.cpp::camera_get_number_of_cameras

CameraHal_Module.cpp文件实现了camera_common.h声明的接口，get_number_of_cameras()函数对应的实现是camera_get_number_of_cameras()

```cpp
int camera_get_number_of_cameras(void)
{

    char cam_num[3],i;
    int cam_cnt=0,fd=-1,rk29_cam[CAMERAS_SUPPORT_MAX];
	camera_board_profiles * profiles = NULL;
    size_t nCamDev = 0;
	int connectUvc=0;
    ...
    profiles = camera_board_profiles::getInstance();
    nCamDev = profiles->mDevieVector.size();
	LOGD("board profiles cam num %d\n", nCamDev);
        for (i=0; i<CAMERAS_SUPPORT_MAX; i++) {
            cam_path[0] = 0x00;
            strcat(cam_path, CAMERA_DEVICE_NAME);                                                                                              
            sprintf(cam_num, "%d", i);
            strcat(cam_path,cam_num);
            fd = open(cam_path, O_RDONLY);
            if (fd < 0) {
                LOGE("Open %s failed! strr: %s",cam_path,strerror(errno));
                break;
            }
	    connectUvc = 1;
            LOGD("Open %s success!",cam_path);
	    close(fd);
	}

if (i < 2){
    if (nCamDev>0) {
	    if (connectUvc == 1)
	        goto CSI_PRO;
	    else
	    camera_board_profiles::OpenAndRegistALLSensor(profiles);
        ...
    }
}
CSI_PRO:
   	if(cam_cnt<CAMERAS_SUPPORT_MAX){
		int i=0;
        ...
        for (i=0; i<10; i++) {
            ...
            if ((capability.capabilities & (V4L2_CAP_VIDEO_CAPTURE | V4L2_CAP_STREAMING)) != (V4L2_CAP_VIDEO_CAPTURE | V4L2_CAP_STREAMING)) {
        	    LOGD("Video device(%s): video capture not supported.\n",cam_path);
            } else {
            	rk_cam_total_info* pNewCamInfo = new rk_cam_total_info();
                ...
				if(strcmp((char*)&capability.driver[0],"uvcvideo") == 0)//uvc
				{
					...
					//add usb camera to board_profiles 
					rk_DV_info *pDVResolution = new rk_DV_info();
					...
					for(i=0; i<element_count; i++){
						...
						rk_DV_info *pDVResolution = new rk_DV_info();
						pDVResolution->mWidth = width;
		    	    	pDVResolution->mHeight = height;
		    	    	pDVResolution->mFps = 0;
		    	    	...
						if ((width>sensor_resolution_w) || (height>sensor_resolution_h)) {
							pNewCamInfo->mSoftInfo.mDV_vector.add(pDVResolution);
	                		continue;
	            		}
	            		...
						pNewCamInfo->mSoftInfo.mDV_vector.add(pDVResolution);	        
					}
				}
				...
				profiles->mCurDevice= pNewCamInfo;
            	profiles->mDevieVector.add(pNewCamInfo);
				pNewCamInfo->mDeviceIndex = (profiles->mDevieVector.size()) - 1;
                camInfoTmp[cam_cnt].pcam_total_info = pNewCamInfo;
                cam_cnt++;
                if (cam_cnt >= CAMERAS_SUPPORT_MAX)
                    i = 10;
            }
    loop_continue:
            if (fd > 0) {
                close(fd);
                fd = -1;
            }
            continue;    
        }
   	}
    gCamerasNumber = cam_cnt;
...  
camera_get_number_of_cameras_end:
    LOGD("%s(%d): Current board have %d cameras attached.",__FUNCTION__, __LINE__, gCamerasNumber);
    return gCamerasNumber;
}
```

camera_get_number_of_cameras()函数主要做了一下几个事情：

1. camera_board_profiles::getInstance() new一个camera_board_profiles实例，用于解析/vendor/etc/cam_board.xml文件，该文件是相关摄像头的配置文件。
2. new rk_DV_info和rk_cam_total_info对象并根据UVC摄像头的描述符填充相关的信息。
3. 将new出来的rk_DV_info和rk_cam_total_info对象分别加入camera_board_profiles的vector全局变量中。

内存泄漏就出现在camera_get_number_of_cameras()函数。

**从上面分析得知camera_get_number_of_cameras()会new出来几个对象，而且new出来的对应并没有传递出去，但是没看到delete的地方，因此导致了内存泄漏。**

## 3. 解决方案

### 3.1 初步解决方案

既然知道了内存泄漏的原因，解决的方案就简单了，在camera_get_number_of_cameras()函数结束前将new出来的对象delete即可。解决方案如下：

```cpp
camera_get_number_of_cameras_end:
    LOGD("%s(%d): Current board have %d cameras attached.",__FUNCTION__, __LINE__, gCamerasNumber);
    //add by wyh for fixing memory leak--start
    for (Vector<rk_cam_total_info*>::const_iterator itr=profiles->mDevieVector.begin(); itr!=profiles->mDevieVector.end(); ++itr) {
        for (Vector<rk_DV_info*>::const_iterator itr2=(*itr)->mSoftInfo.mDV_vector.begin(); itr2!=(*itr)->mSoftInfo.mDV_vector.end(); ++itr2) {
            delete *itr2;
		}
		(*itr)->mSoftInfo.mDV_vector.clear();
		delete *itr;
    }
    profiles->mDevieVector.clear();
    delete profiles;
    //add by wyh for fixing memory leak--end

    return gCamerasNumber;
}
```

编译源码进行烧录到板子重新压测，发现对比修改前的每次开关导致的内存泄漏量：200k左右下降到了10k左右，但每次开关UVC还是会有10k左右的内存泄漏，继续修改。

### 3.2 最终方案

前面提到new出来的camera_board_profiles对象会解析/vendor/etc/cam_board.xml文件，该文件里面都是使用的RK3399开发板上没有的camera，/vendor/etc/cam_board.xml文件是在源码编译时将hardware/rockchip/camera/Config/cam_board_rk3399.xml文件烧录到板子的/vendor/etc/cam_board.xml。因此尝试将hardware/rockchip/camera/Config/cam_board_rk3399.xml的CamDevie字段屏蔽，重新编译烧录测试，结果显示每次开关UVC导致的10k的内存泄漏没有了。问题解决。

为什么屏蔽掉cam_board.xml的CamDevie字段就解决了10k的内存泄漏了呢，深挖一下。

```cpp
    if (nCamDev>0) {
	    camera_board_profiles::OpenAndRegistALLSensor(profiles);
```

new出来的camera_board_profiles解析到vendor/etc/cam_board.xml文件有多少个CamDevie就会赋值nCamDev为多少，因此会调用OpenAndRegistALLSensor函数。该函数会根据CamDevie的个数采用dlopen的方式分别调用CamDevie对应的isp库，如cam_board.xml文件的CamDevie的Sensor为OV13850对应libisp_isi_drv_OV13850.so。所以猜测是调用对应的isp库导致了10k的内存泄漏。

将hardware/rockchip/camera/Config/cam_board_rk3399.xml文件还原，但将camera_board_profiles::OpenAndRegistALLSensor(profiles)注释，编译并重新烧录测试，发现10k的内存泄漏也解决了。

最终的方案是采用delete new出来的对象和屏蔽hardware/rockchip/camera/Config/cam_board_rk3399.xml的CamDevie字段的方案。压测1万多次内存正常。如下

```
rk3399_firefly_mid:/ $ dumpsys meminfo
Applications Memory Usage (in Kilobytes):
Uptime: 4982785 Realtime: 4982785

Total PSS by process:
     86,037K: system (pid 483)
     82,551K: com.android.systemui (pid 643)
     63,537K: zygote (pid 355)
     34,127K: com.android.launcher3 (pid 1125 / activities)
     26,210K: media.codec (pid 368)
     25,679K: webview_zygote32 (pid 712)
     22,744K: zygote64 (pid 354)
     22,268K: com.ang.demo.camera (pid 2492 / activities)
     21,281K: com.android.phone (pid 904)
     18,737K: android.hardware.neuralnetworks@1.0-service-armnn (pid 241)
     17,118K: app_process (pid 1730)
     14,621K: com.android.inputmethod.latin (pid 626)
     14,107K: surfaceflinger (pid 247)
     13,773K: app_process (pid 1739)
     11,962K: android.process.media (pid 1235)
     11,933K: com.android.email (pid 1387)
     11,641K: android.hardware.camera.provider@2.4-service (pid 233)
```

## 4. 总结

该问题目前只是在FireFly RK3399 Android8.1系统才会导致该Camera HAL层内存泄漏，只有使用libusb库的方式操作UVC设备才会导致才问题。导致内存泄漏的原因是UVC每次remove和add都会遍历UVC设备，最终在Camera HAL层的camera_get_number_of_cameras()函数出现了内存泄漏，泄漏点一是new了很多对象，但没有执行delete操作。泄漏点二是cam_board.xml的CamDevie对应的isp库发生了10k的内存泄漏。
