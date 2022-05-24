---
layout: post
title:  "RK3399 Android系统插入UVC Camera导致系统开机失败"
date:   2022-05-24 00:00:00
catalog:  true
tags:
    - android
    - 稳定性
    - camera
---

# 背景

某些板厂的RK3399 android7.1系统在插入我们的深度UVC Camera设备后，再次开机android系统启动失败，一直停留在开机动画界面。

# 分析开机logcat

```
--------- beginning of crash
01-18 09:05:33.565   248   248 F libc    : Fatal signal 6 (SIGABRT), code -6 in tid 248 (main)
01-18 09:05:33.565   221   221 W         : debuggerd: handling request: pid=248 uid=9999 gid=9999 tid=248
01-18 09:05:33.572   484   484 I AppOps  : No existing app ops /data/system/appops.xml; starting empty
01-18 09:05:33.573   576   576 E         : debuggerd: attached task 248 does not match request: expected pid=248,uid=9999,gid=9999, actual pid=248,uid=0,gid=0
01-18 09:05:33.574   221   221 W         : debuggerd: resuming target 248
01-18 09:05:33.575   249   249 I art     : Thread[1,tid=249,Native,Thread*=0xf4005400,peer=0x12c070d0,"main"] recursive attempt to load library "/system/lib/libmedia_jni.so"
01-18 09:05:33.576   249   249 D MtpDeviceJNI: register_android_mtp_MtpDevice
01-18 09:05:33.576   249   249 I art     : Thread[1,tid=249,Native,Thread*=0xf4005400,peer=0x12c070d0,"main"] recursive attempt to load library "/system/lib/libmedia_jni.so"
01-18 09:05:33.576   249   249 I art     : Thread[1,tid=249,Native,Thread*=0xf4005400,peer=0x12c070d0,"main"] recursive attempt to load library "/system/lib/libmedia_jni.so"
01-18 09:05:33.578   249   249 D MediaProfiles: getInstance
01-18 09:05:33.578   249   249 D MediaProfiles: CameraGroupFound(624): media_profiles_id: 0x0
01-18 09:05:33.578   249   249 D MediaProfiles: getInstance(718): Create instance from /data/camera/media_profiles.xml 
01-18 09:05:33.581   249   249 F MediaProfiles: frameworks/av/media/libmedia/MediaProfiles.cpp:546 CHECK(refIndex != -1) failed.
01-18 09:05:33.581   249   249 F libc    : Fatal signal 6 (SIGABRT), code -6 in tid 249 (main)
01-18 09:05:33.581   220   220 W         : debuggerd: handling request: pid=249 uid=9999 gid=9999 tid=249
01-18 09:05:33.584   579   579 E         : debuggerd: attached task 249 does not match request: expected pid=249,uid=9999,gid=9999, actual pid=249,uid=0,gid=0
01-18 09:05:33.585   220   220 W         : debuggerd: resuming target 249
01-18 09:05:33.587   251   251 I ServiceManager: Waiting for service sensorservice...
01-18 09:05:33.595   260   260 V NatController: runCmd(/system/bin/iptables -w -F natctrl_FORWARD) res=0
01-18 09:05:33.606   260   260 V NatController: runCmd(/system/bin/ip6tables -w -F natctrl_FORWARD) res=0
01-18 09:05:33.616   235   235 I ServiceManager: service 'media.player' died
01-18 09:05:33.616   235   235 I ServiceManager: service 'media.resource_manager' died
01-18 09:05:33.616   235   235 I ServiceManager: service 'media.audio_flinger' died
01-18 09:05:33.616   235   235 I ServiceManager: service 'media.audio_policy' died
01-18 09:05:33.616   235   235 I ServiceManager: service 'media.sound_trigger_hw' died
01-18 09:05:33.616   235   235 I ServiceManager: service 'media.radio' died
01-18 09:05:33.653   253   253 E         : eof
01-18 09:05:33.653   253   253 E         : failed to read size
01-18 09:05:33.653   253   253 I         : closing connection
01-18 09:05:36.737   589   589 I Netd    : Netd 1.0 starting
01-18 09:05:36.737   589   589 D TetherController: Setting IP forward enable = 0
01-18 09:05:36.827   587   587 I cameraserver: ServiceManager: 0xef599420
01-18 09:05:36.827   587   587 I CameraService: CameraService started (pid=587)
01-18 09:05:36.827   587   587 I CameraService: CameraService process starting
01-18 09:05:36.828   587   587 W BatteryNotifier: batterystats service unavailable!
01-18 09:05:36.828   587   587 W BatteryNotifier: batterystats service unavailable!
```

logcat文件搜索beginning of crash，上面的log显示是media进程发生了crash，导致ServiceManager: Waiting for service sensorservice，然后系统不停重新启动media进程，然后media进程启动后又发生了crash，一直循环。

具体发生crash的原因上面的log已经打印了：

```
Thread[1,tid=249,Native,Thread*=0xf4005400,peer=0x12c070d0,"main"] recursive attempt to load library "/system/lib/libmedia_jni.so"
01-18 09:05:33.578   249   249 D MediaProfiles: getInstance
01-18 09:05:33.578   249   249 D MediaProfiles: CameraGroupFound(624): media_profiles_id: 0x0
01-18 09:05:33.578   249   249 D MediaProfiles: getInstance(718): Create instance from /data/camera/media_profiles.xml 
01-18 09:05:33.581   249   249 F MediaProfiles: frameworks/av/media/libmedia/MediaProfiles.cpp:546 CHECK(refIndex != -1) failed.
01-18 09:05:33.581   249   249 F libc    : Fatal signal 6 (SIGABRT), code -6 in tid 249 (main)
01-18 09:05:33.581   220   220 W         : debuggerd: handling request: pid=249 uid=9999 gid=9999 tid=249
```

猜测是在执行MediaProfiles: frameworks/av/media/libmedia/MediaProfiles.cpp:546 CHECK(refIndex != -1) failed后发生的crash。

MediaProfiles.cpp源码如下：

```cpp
int refIndex = getRequiredProfileRefIndex(cameraId);
CHECK(refIndex != -1);
RequiredProfileRefInfo *info =
                    &mRequiredProfileRefs[refIndex].mRefs[j];
```

即refIndex的值为-1，mRequiredProfileRefs[refIndex].mRefs[j]发生了crash。下面分析具体的流程。

# MediaProfiles.cpp初始化

MediaProfiles初始化时序图如下：

![image-20211018221219906](/images/camera/meidia_profiles.png)

```c
/*static*/ MediaProfiles*
MediaProfiles::getInstance()
{
    ALOGD("getInstance");
    ...
            ALOGD("%s(%d): 2 Create instance from %s ",__FUNCTION__,__LINE__,value);
            sInstance = createInstanceFromXmlFile(value);
        }
        CHECK(sInstance != NULL);
        sInstance->checkAndAddRequiredProfilesIfNecessary();
        sIsInitialized = true;
    }

    ALOGD("get Instance:%p", sInstance);

    return sInstance;
}
```

getInstance()函数会调用createInstanceFromXmlFile()函数创建MediaProfiles实例，然后在调用checkAndAddRequiredProfilesIfNecessary()函数做一些检测工作。

## createInstanceFromXmlFile

```cpp
/*static*/ MediaProfiles*
MediaProfiles::createInstanceFromXmlFile(const char *xml)
{
    FILE *fp = NULL;
    CHECK((fp = fopen(xml, "r")));

    XML_Parser parser = ::XML_ParserCreate(NULL);
    CHECK(parser != NULL);

    MediaProfiles *profiles = new MediaProfiles();
    ::XML_SetUserData(parser, profiles);
    ::XML_SetElementHandler(parser, startElementHandler, NULL);

    /*
      FIXME:
      expat is not compiled with -DXML_DTD. We don't have DTD parsing support.

      if (!::XML_SetParamEntityParsing(parser, XML_PARAM_ENTITY_PARSING_ALWAYS)) {
          ALOGE("failed to enable DTD support in the xml file");
          return UNKNOWN_ERROR;
      }

    */

    const int BUFF_SIZE = 512;
    for (;;) {
        void *buff = ::XML_GetBuffer(parser, BUFF_SIZE);
        if (buff == NULL) {
            ALOGE("failed to in call to XML_GetBuffer()");
            delete profiles;
            profiles = NULL;
            goto exit;
        }

        int bytes_read = ::fread(buff, 1, BUFF_SIZE, fp);
        if (bytes_read < 0) {
            ALOGE("failed in call to read");
            delete profiles;
            profiles = NULL;
            goto exit;
        }
        CHECK(::XML_ParseBuffer(parser, bytes_read, bytes_read == 0));

        if (bytes_read == 0) break;  // done parsing the xml file
    }

exit:
    ::XML_ParserFree(parser);
    ::fclose(fp);
    return profiles;
}
```

该函数的工作是从/data/camera/media_profiles.xml中解析相关的项，然后创建对应的变量，具体的工作是调用startElementHandler()完成的。

```cpp
/*static*/ void
MediaProfiles::startElementHandler(void *userData, const char *name, const char **atts)
{
    MediaProfiles *profiles = (MediaProfiles *) userData;
    if (strcmp("Video", name) == 0) {
        createVideoCodec(atts, profiles);
    } else if (strcmp("Audio", name) == 0) {
        createAudioCodec(atts, profiles);
    } else if (strcmp("VideoEncoderCap", name) == 0 &&
               strcmp("true", atts[3]) == 0) {
        profiles->mVideoEncoders.add(createVideoEncoderCap(atts));
    } else if (strcmp("AudioEncoderCap", name) == 0 &&
               strcmp("true", atts[3]) == 0) {
        profiles->mAudioEncoders.add(createAudioEncoderCap(atts));
    } else if (strcmp("VideoDecoderCap", name) == 0 &&
               strcmp("true", atts[3]) == 0) {
        profiles->mVideoDecoders.add(createVideoDecoderCap(atts));
    } else if (strcmp("AudioDecoderCap", name) == 0 &&
               strcmp("true", atts[3]) == 0) {
        profiles->mAudioDecoders.add(createAudioDecoderCap(atts));
    } else if (strcmp("EncoderOutputFileFormat", name) == 0) {
        profiles->mEncoderOutputFileFormats.add(createEncoderOutputFileFormat(atts));
    } else if (strcmp("CamcorderProfiles", name) == 0) {
        profiles->mCurrentCameraId = getCameraId(atts);
        profiles->addStartTimeOffset(profiles->mCurrentCameraId, atts);
    } else if (strcmp("EncoderProfile", name) == 0) {
        profiles->mCamcorderProfiles.add(
            createCamcorderProfile(profiles->mCurrentCameraId, atts, profiles->mCameraIds));
    } else if (strcmp("ImageEncoding", name) == 0) {
        profiles->addImageEncodingQualityLevel(profiles->mCurrentCameraId, atts);
    }
}
```

重点关注EncoderProfile项，/data/camera/media_profiles.xml文件是插入UVC camera后生成的，具体内容部分如下：

```xml
<MediaSettings>
    <!-- Each camcorder profile defines a set of predefined configuration parameters -->
    <CamcorderProfiles cameraId="0">

<!--  
       <EncoderProfile quality="qcif" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="256000"
                   width="176"
                   height="144"
                   frameRate="10" /> 

            <Audio codec="aac"
                   bitRate="61000"
                   sampleRate="44100"
                   channels="1" />

        </EncoderProfile>
-->  
        
<!--  
        <EncoderProfile quality="qvga" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="128000"
                   width="320"
                   height="240"
                   frameRate="30" /> 
            <Audio codec="amrnb"
                   bitRate="12200"
                   sampleRate="8000"
                   channels="1" />
        </EncoderProfile> 
-->  
        
<!--  
        <EncoderProfile quality="cif" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="1200000"
                   width="352"
                   height="288"
                   frameRate="10" /> 
            <Audio codec="aac"
                   bitRate="61000"
                   sampleRate="44100"
                   channels="1" />
        </EncoderProfile>   
-->  
        
        <!--  If your sensor driver don't support 720p and 480p stream, Please fill this element according as
              your sensor max resolution for preview(Not Capture resolution)  -->                  
<!--  
       <EncoderProfile quality="480p" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="256000"
                   width="640"
                   height="480"
                   frameRate="10" /> 

            <Audio codec="aac"
                   bitRate="61000"
                   sampleRate="44100"
                   channels="1" />
        </EncoderProfile>
-->  
        
        <!--  If your sensor driver don't support 480p stream, Please turn off this element -->
          
<!--  
       <EncoderProfile quality="480p" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="256000"
                   width="720"
                   height="480"
                   frameRate="10" /> 

            <Audio codec="aac"
                   bitRate="61000"
                   sampleRate="44100"
                   channels="1" />
        </EncoderProfile>
-->  

        <!--  If your sensor driver don't support 480p stream, Please turn off this element -->
          
<!--  
       <EncoderProfile quality="480p" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="256000"
                   width="800"
                   height="600"
                   frameRate="10" /> 

            <Audio codec="aac"
                   bitRate="61000"
                   sampleRate="44100"
                   channels="1" />
        </EncoderProfile>
-->  


         <!--  If your sensor driver don't support 576p stream, Please turn off this element -->

<!--  
       <EncoderProfile quality="576p" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="256000"
                   width="720"
                   height="576"
                   frameRate="12" />

            <Audio codec="aac"
                   bitRate="61000"
                   sampleRate="44100"
                   channels="1" />
        </EncoderProfile>
-->  

        <!--  If your sensor driver don't support 720p stream, Please turn off this element -->

<!--  
       <EncoderProfile quality="720p" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="3000000"
                   width="1280"
                   height="720"
                   frameRate="10" /> 

            <Audio codec="aac"
                   bitRate="61000"
                   sampleRate="44100"
                   channels="1" />
        </EncoderProfile>
-->  
        
        <!--  If your sensor driver don't support 1080p stream, Please turn off this element -->

<!--  
       <EncoderProfile quality="1080p" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="3000000"
                   width="1920"
                   height="1080"
                   frameRate="0" /> 

            <Audio codec="aac"
                   bitRate="61000"
                   sampleRate="44100"
                   channels="1" />
        </EncoderProfile>
-->  
       
<!--  
       <EncoderProfile quality="timelapseqcif" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="192000"
                   width="176"
                   height="144"
                   frameRate="10" /> 
            
            <Audio codec="aac"
                   bitRate="61000"
                   sampleRate="44100"
                   channels="1" />
        </EncoderProfile>
-->  
        
<!--  
       <EncoderProfile quality="timelapseqvga" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="128000"
                   width="320"
                   height="240"
                   frameRate="30" /> 
            <Audio codec="amrnb"
                   bitRate="12200"
                   sampleRate="8000"
                   channels="1" />
        </EncoderProfile>
-->  

<!--  
       <EncoderProfile quality="timelapsecif" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="1200000"
                   width="352"
                   height="288"
                   frameRate="10" /> 
            
            <Audio codec="aac"
                   bitRate="61000"
                   sampleRate="44100"
                   channels="1" />
        </EncoderProfile>  
-->  
              
        <!--  If your sensor driver don't support 720p and 480p stream, Please fill this element according as
              your sensor max resolution for preview(Not Capture resolution)  --> 
<!--  
        <EncoderProfile quality="timelapse480p" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="1200000"
                   width="640"
                   height="480"
                   frameRate="10" /> 
            
       			<Audio codec="aac"
                   bitRate="61000"
                   sampleRate="44100"
                   channels="1" />	
        </EncoderProfile> 
-->  
        
        <!--  If your sensor driver don't support 480p stream, Please turn off this element  -->  
          
<!--  
       <EncoderProfile quality="timelapse480p" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="256000"
                   width="720"
                   height="480"
                   frameRate="10" /> 

            <Audio codec="aac"
                   bitRate="61000"
                   sampleRate="44100"
                   channels="1" />
        </EncoderProfile>
-->  

        <!--  If your sensor driver don't support 480p stream, Please turn off this element  -->  
          
<!--  
       <EncoderProfile quality="timelapse480p" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="256000"
                   width="800"
                   height="600"
                   frameRate="10" /> 

            <Audio codec="aac"
                   bitRate="61000"
                   sampleRate="44100"
                   channels="1" />
        </EncoderProfile>
-->  


         <!--  If your sensor driver don't support 720p and 480p stream, Please fill this element according as
              your sensor max resolution for preview(Not Capture resolution)  -->
<!--  
        <EncoderProfile quality="timelapse480p" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="1200000"
                   width="800"
                   height="600"
                   frameRate="10" /> 

            <Audio codec="aac"
                   bitRate="61000"
                   sampleRate="44100"
                   channels="1" />
        </EncoderProfile>
-->  

         <!--  If your sensor driver don't support 576p stream, Please fill this element according as
              your sensor max resolution for preview(Not Capture resolution)  -->
<!--  
        <EncoderProfile quality="timelapse576p" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="1200000"
                   width="720"
                   height="576"
                   frameRate="15" />
            
            <Audio codec="aac"
                   bitRate="61000"
                   sampleRate="44100"
                   channels="1" />	
        </EncoderProfile> 
-->  

        <!--  If your sensor driver don't support 720p stream, Please turn off this element -->

<!--  
       <EncoderProfile quality="timelapse720p" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="3000000"
                   width="1280"
                   height="720"
                   frameRate="10" /> 

            <Audio codec="aac"
                   bitRate="61000"
                   sampleRate="44100"
                   channels="1" />
        </EncoderProfile>
-->  

		<!--  If your sensor driver don't support 108p stream, Please turn off this element -->

<!--  
       <EncoderProfile quality="timelapse1080p" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="3000000"
                   width="1920"
                   height="1080"
                   frameRate="0" /> 

            <Audio codec="aac"
                   bitRate="61000"
                   sampleRate="44100"
                   channels="1" />
        </EncoderProfile>
-->  

        <ImageEncoding quality="90" />
        <ImageEncoding quality="80" />
        <ImageEncoding quality="70" />
        <ImageDecoding memCap="20000000" />

</CamcorderProfiles>
```

上面的media_profiles.xml内容是从/system/etc/media_profiles_default.xml拷贝过来的，然后根据插入的UVC Camera属性，如果UVC Camera不支持xml文件的分辨率，就将该分辨率注释。

由于插入的UVC camera不支持上面的标准分辨率：176x144,320x240,352x288,640x480,720x480,800x600,720x576,1280x720,1920x1080,因此这些分辨率都注释了，即EncoderProfile项都注释了。因此下面的条件就没成立，导致createCamcorderProfile没执行，createCamcorderProfile()函数的功能是创建一个MediaProfiles::CamcorderProfile对象和给**mCameraIds**容器添加一个cameraId，这个步骤很重要，后面的检测步骤就是由于检测到mCameraIds没有元素导致了crash。

```
} else if (strcmp("EncoderProfile", name) == 0) {
    profiles->mCamcorderProfiles.add(
        createCamcorderProfile(profiles->mCurrentCameraId, atts, profiles->mCameraIds));
}
```

## checkAndAddRequiredProfilesIfNecessary

```cpp
void MediaProfiles::checkAndAddRequiredProfilesIfNecessary() {
    ...

    for (size_t cameraId = 0; cameraId < mCameraIds.size(); ++cameraId) {
        for (size_t j = 0; j < kNumRequiredProfiles; ++j) {
            int refIndex = getRequiredProfileRefIndex(cameraId);
            CHECK(refIndex != -1);
            RequiredProfileRefInfo *info =
                    &mRequiredProfileRefs[refIndex].mRefs[j];

            ...
}
```

从开机log中得知打印了CHECK(refIndex != -1)，即getRequiredProfileRefIndex的值为-1，源码如下：

```cpp
int MediaProfiles::getRequiredProfileRefIndex(int cameraId) {
    for (size_t i = 0, n = mCameraIds.size(); i < n; ++i) {
        if (mCameraIds[i] == cameraId) {
            return i;
        }
    }
    return -1;
}
```

由于media_profiles.xml没有EncoderProfile节点，导致不执行createCamcorderProfile()函数，从而导致mCameraIds没有元素，进而导致上面的getRequiredProfileRefIndex()返回-1，最终在执行mRequiredProfileRefs[refIndex].mRefs[j]发生了crash。

## 解决方案

该问题是RK3399 android7.1早期基线版本存在的BUG，各板厂修复的情况不一，Firefly板厂在2018年6月5号的提交e25808459b8dec3013caebb7391871f235b5fffa修复了该问题，patch地址：[patch](https://gitlab.com/TeeFirefly/FireNow-Nougat/-/tree/e25808459b8dec3013caebb7391871f235b5fffa)

```cpp
commit e25808459b8dec3013caebb7391871f235b5fffa
Author: Firefly-RK3288 <service@t-firefly.com>
Date:   Tue Jun 5 16:01:05 2018 +0800

    1.fix adb connect, 2.support mic and usb speaker. 3.support mipi dsi. 4.support single lvds . 5.fix ethernet error. 6.support print service. 7.support hdmi some property setting. 8. fix system other bug
    
diff --git a/frameworks/av/media/libmedia/MediaProfiles.cpp b/frameworks/av/media/libmedia/MediaProfiles.cpp
index 60e833c..7e9a73b 100644
--- a/frameworks/av/media/libmedia/MediaProfiles.cpp
+++ b/frameworks/av/media/libmedia/MediaProfiles.cpp
@@ -15,7 +15,7 @@
 ** limitations under the License.
 */
 
-
+#undef NDEBU
 //#define LOG_NDEBUG 0
 #define LOG_TAG "MediaProfiles"
 
@@ -715,8 +715,15 @@ MediaProfiles::getInstance()
                 fclose(fp);  // close the file first.
                 ALOGD("%s(%d): Create instance from %s ",__FUNCTION__,__LINE__,defaultXmlFile);
                 sInstance = createInstanceFromXmlFile(defaultXmlFile);
+                if(sInstance == NULL){
+                    ALOGD("sInstance == NULL");
+                    sInstance = createDefaultInstance();
+                }else{
+                    ALOGD("sInstance != NULL");
+                }
             }
         } else {
+            ALOGD("%s(%d): 2 Create instance from %s ",__FUNCTION__,__LINE__,value);
             sInstance = createInstanceFromXmlFile(value);
         }
         CHECK(sInstance != NULL);
@@ -949,6 +956,7 @@ MediaProfiles::createDefaultImageEncodingQualityLevels(MediaProfiles *profiles)
 /*static*/ MediaProfiles*
 MediaProfiles::createDefaultInstance()
 {
+    ALOGW("createDefaultInstance");
     MediaProfiles *profiles = new MediaProfiles;
     createDefaultCamcorderProfiles(profiles);
     createDefaultVideoEncoders(profiles);
@@ -1002,7 +1010,23 @@ MediaProfiles::createInstanceFromXmlFile(const char *xml)
             goto exit;
         }
 
-        CHECK(::XML_ParseBuffer(parser, bytes_read, bytes_read == 0));
+        //XML_STATUS_OK 1
+        //XML_STATUS_ERROR 0
+        int status = ::XML_ParseBuffer(parser, bytes_read, bytes_read == 0);
+        ALOGE("createInstanceFromXmlFile status:%d",status);
+        if(status != ::XML_STATUS_OK)
+        {
+          if(remove(xml)){
+                ALOGD("Could not delete the file");
+          }else{
+                ALOGD(" delete the file ok");
+          }
+           ::XML_ParserFree(parser);
+           ::fclose(fp);
+            return NULL;
+        }
+        CHECK(status);
+        //CHECK(::XML_ParseBuffer(parser, bytes_read, bytes_read == 0));
 
         if (bytes_read == 0) break;  // done parsing the xml file
     }
```

该patch的修复方案是创建一个默认的createDefaultInstance，不从media_profiles.xml创建。

UVC camera侧厂家修复方案是添加一个media_profiles.xml支持的分辨率：176x144,320x240,352x288,640x480,720x480,800x600,720x576,1280x720,1920x1080。

# 结论

插入不支持176x144,320x240,352x288,640x480,720x480,800x600,720x576,1280x720,1920x1080分辨率的UVC Camera由于MediaProfiles解析media_profiles.xml文件没有EncoderProfile节点，导致不执行createCamcorderProfile()函数，从而导致mCameraIds没有元素，最终发生了crash。
