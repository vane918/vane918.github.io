---
layout: post
title:  "Android9/10返回错误VID/PID的问题分析"
date:   2023-07-14 00:00:00
catalog:  true
tags:
    - android
    - uvc
---

# 背景

Android9/10开发板插入绝大多数UVC设备，使用Android的API UsbDevice返回错误的VID/PID。针对该问题，下面深入Android9源码分析该问题。

涉及到的文件如下：

> frameworks/base/services/usb/java/com/ndroid/server/usb/UsbHostManager.java
>
> frameworks/base/services/usb/java/com/ndroid/server/usb//descriptors/UsbDescriptorParser.java
>
> frameworks/base/services/usb/java/com/ndroid/server/usb//descriptors/UsbDeviceDescriptor.java
>
> frameworks/base/services/usb/java/com/ndroid/server/usb//descriptors/UsbDescriptor.java
>
> frameworks/base/services/usb/java/com/ndroid/server/usb//descriptors/UsbACInterface.java
>
> frameworks/base/services/usb/java/com/ndroid/server/usb//descriptors/Usb10ACHeader.java
>
> frameworks/base/services/core/jni/com_android_server_UsbHostManager.cpp
>
> system/core/libusbhost/usbhost.c

# 1. USB热插拔事件上报流程

Android系统起来后，调用到USB模块UsbHostManager.java的systemReady()函数，该函数开启线程调用JNI函数monitorUsbHostBus监听USB插入或拔出事件，最终是跑到usbhost.c的usb_host_run函数进行监听USB热插拔事件，在usb_host_run传入插入和拔出的回调函数：usb_device_added和usb_device_removed，这两个JNI函数最终上报到UsbHostManager.java的usbDeviceAdded和usbDeviceRemoved函数。

# 2. USB解析描述符

UsbHostManager.java

当有USB设备插入时，最终回调到下面的函数，该函数调用UsbDescriptorParser.java解析描述符。

```java
@SuppressWarnings("unused")
private boolean usbDeviceAdded(String deviceAddress, int deviceClass, int       deviceSubclass, byte[] descriptors) {
	...
    UsbDescriptorParser parser = new UsbDescriptorParser(deviceAddress, descriptors);
    logUsbDevice(parser);
    ...
}
```

UsbDescriptorParser.java

```java
public UsbDescriptorParser(String deviceAddr, byte[] rawDescriptors) {
    mDeviceAddr = deviceAddr;
    mDescriptors = new ArrayList<UsbDescriptor>(DESCRIPTORS_ALLOC_SIZE);
    parseDescriptors(rawDescriptors);
}
/**
 * @hide
 */
public void parseDescriptors(byte[] descriptors) {
    ByteStream stream = new ByteStream(descriptors);
    while (stream.available() > 0) {
        UsbDescriptor descriptor = null;
        try {
            descriptor = allocDescriptor(stream);
        } catch (Exception ex) {
            Log.e(TAG, "Exception allocating USB descriptor.", ex);
        }

        if (descriptor != null) {
            // Parse
            try {
                descriptor.parseRawDescriptors(stream);
                // Clean up
                descriptor.postParse(stream);
            } catch (Exception ex) {
                Log.e(TAG, "Exception parsing USB descriptors.", ex);	
                descriptor.setStatus(UsbDescriptor.STATUS_PARSE_EXCEPTION);
            } finally {
                mDescriptors.add(descriptor);
            }
        }
    }
}
```

上面的parseDescriptors函数主要有两个步骤：

1. 根据入参descriptors描述符调用allocDescriptor创建UsbDescriptor解析类，主要是解析descriptors创建对应的描述符解析器，如UVC、UAC、HID等。
2. 根据步骤1创建的描述符解析器调用parseRawDescriptors函数解析描述符，生成对应的变量。

## 2.1 创建描述符解析器

UsbDescriptorParser.java

```java
private UsbDescriptor allocDescriptor(ByteStream stream)
        throws UsbDescriptorsStreamFormatException {
    stream.resetReadCount();
    int length = stream.getUnsignedByte();
    byte type = stream.getByte();

    UsbDescriptor descriptor = null;
    switch (type) {
        /*
         * Standard
         */
        case UsbDescriptor.DESCRIPTORTYPE_DEVICE:
            descriptor = mDeviceDescriptor = new UsbDeviceDescriptor(length, type);
            break;

        case UsbDescriptor.DESCRIPTORTYPE_CONFIG:
            descriptor = mCurConfigDescriptor = new UsbConfigDescriptor(length, type);
            if (mDeviceDescriptor != null) {
                mDeviceDescriptor.addConfigDescriptor(mCurConfigDescriptor);
            } else {
                Log.e(TAG, "Config Descriptor found with no associated Device Descriptor!");
                throw new UsbDescriptorsStreamFormatException(
                        "Config Descriptor found with no associated Device Descriptor!");
            }
            break;

        case UsbDescriptor.DESCRIPTORTYPE_INTERFACE:
            descriptor = mCurInterfaceDescriptor = new UsbInterfaceDescriptor(length, type);
            if (mCurConfigDescriptor != null) {
                mCurConfigDescriptor.addInterfaceDescriptor(mCurInterfaceDescriptor);
            } else {
                Log.e(TAG, "Interface Descriptor found with no associated Config Descriptor!");
                throw new UsbDescriptorsStreamFormatException(
                        "Interface Descriptor found with no associated Config Descriptor!");
            }
            break;

        case UsbDescriptor.DESCRIPTORTYPE_ENDPOINT:
            descriptor = new UsbEndpointDescriptor(length, type);
            if (mCurInterfaceDescriptor != null) {
                mCurInterfaceDescriptor.addEndpointDescriptor(
                        (UsbEndpointDescriptor) descriptor);
            } else {
                Log.e(TAG,
                        "Endpoint Descriptor found with no associated Interface Descriptor!");
                throw new UsbDescriptorsStreamFormatException(
                        "Endpoint Descriptor found with no associated Interface Descriptor!");
            }
            break;

        /*
         * HID
         */
        case UsbDescriptor.DESCRIPTORTYPE_HID:
            descriptor = new UsbHIDDescriptor(length, type);
            break;

        /*
         * Other
         */
        case UsbDescriptor.DESCRIPTORTYPE_INTERFACEASSOC:
            descriptor = new UsbInterfaceAssoc(length, type);
            break;

        /*
         * Audio Class Specific
         */
        case UsbDescriptor.DESCRIPTORTYPE_AUDIO_INTERFACE:
            descriptor = UsbACInterface.allocDescriptor(this, stream, length, type);
            break;
        case UsbDescriptor.DESCRIPTORTYPE_AUDIO_ENDPOINT:
            descriptor = UsbACEndpoint.allocDescriptor(this, length, type);
            break;
        default:
            break;
    }

    if (descriptor == null) {
        // Unknown Descriptor
        Log.i(TAG, "Unknown Descriptor len: " + length + " type:0x"
                + Integer.toHexString(type));
        descriptor = new UsbUnknown(length, type);
    }

    return descriptor;
}
```

上面的allocDescriptor函数主要是根据描述符判断是什么类型的描述符，如Device Descriptor、Configuration Descriptor、Interface Descriptor、Endpoint Descriptor等。

入参的stream是底层回调USB的描述符，该描述符是一个十进制的数组，里面包含USB设备所有的子描述符的信息，如Device Descriptor、Configuration Descriptor、Interface Descriptor、Endpoint Descriptor等。里面的内容解析按照USB协议文档进行解析。USB描述符数组前18个字节是Device Descriptor，如下面的例子：18, 1, 0, 2, -17, 2, 1, 64, -93, 5, 48, -110, 0, 1, 2, 1, 0, 1，解析如下：

```
    ---------------------- Device Descriptor ----------------------
bLength                  : 0x12 (18 bytes)
bDescriptorType          : 0x01 (Device Descriptor)
bcdUSB                   : 0x200 (USB Version 2.00)
bDeviceClass             : 0xEF (Miscellaneous)
bDeviceSubClass          : 0x02
bDeviceProtocol          : 0x01 (IAD - Interface Association Descriptor)
bMaxPacketSize0          : 0x40 (64 bytes)
idVendor                 : 0x05A3 (TransDimension-NH LLC)
idProduct                : 0x9230
bcdDevice                : 0x0100
iManufacturer            : 0x02 (String Descriptor 2)
 *!*ERROR  String descriptor not found
iProduct                 : 0x01 (String Descriptor 1)
 *!*ERROR  String descriptor not found
iSerialNumber            : 0x00 (No String Descriptor)
bNumConfigurations       : 0x01 (1 Configuration)
```

在Android9/10系统中由于该函数创建错了描述符解析器，导致后面解析USB的VID/PID出现了错误。准确来说是在解析到Video Control Interface Header Descriptor时，其bDescriptorType是0x24，但0x24同时也是UsbDescriptor.DESCRIPTORTYPE_AUDIO_INTERFACE类型，因此解析到下面的内容时，调用了了descriptor = UsbACInterface.allocDescriptor，将Video Control Interface Header Descriptor的数据误认为是UAC音频设备的描述符数据，创建了UAC音频设备的描述符解析器。

13, 36, 1, 0, 1, 77, 0, -64, -31, -28, 0, 1, 1

UsbACInterface.java

```java
/**
 * Allocates an audio class interface subtype based on subtype and subclass.
 */
public static UsbDescriptor allocDescriptor(UsbDescriptorParser parser, ByteStream stream,
        int length, byte type) {
    byte subtype = stream.getByte();
    UsbInterfaceDescriptor interfaceDesc = parser.getCurInterface();
    int subClass = interfaceDesc.getUsbSubclass();
    switch (subClass) {
        case AUDIO_AUDIOCONTROL:
            return allocAudioControlDescriptor(
                    parser, stream, length, type, subtype, subClass);

        case AUDIO_AUDIOSTREAMING:
            return allocAudioStreamingDescriptor(
                    parser, stream, length, type, subtype, subClass);

        case AUDIO_MIDISTREAMING:
            return allocMidiStreamingDescriptor(length, type, subtype, subClass);

        default:
            Log.w(TAG, "Unknown Audio Class Interface Subclass: 0x"
                    + Integer.toHexString(subClass));
            return null;
    }
}
```

根据13, 36, 1, 0, 1, 77, 0, -64, -31, -28, 0, 1, 1描述符数据，与AUDIO_AUDIOCONTROL匹配。

```java
private static UsbDescriptor allocAudioControlDescriptor(UsbDescriptorParser parser,
        ByteStream stream, int length, byte type, byte subtype, int subClass) {
    switch (subtype) {
        case ACI_HEADER:
        {
            int acInterfaceSpec = stream.unpackUsbShort();
            parser.setACInterfaceSpec(acInterfaceSpec);
            if (acInterfaceSpec == UsbDeviceDescriptor.USBSPEC_2_0) {
                return new Usb20ACHeader(length, type, subtype, subClass, acInterfaceSpec);
            } else {
                return new Usb10ACHeader(length, type, subtype, subClass, acInterfaceSpec);
            }
        }

        ...
}
```

subtype为ACI_HEADER，最终创建了Usb10ACHeader类作为描述符解析器。Usb10ACHeader的最上层的父类是UsbDescriptor。

## 2.2 利用描述符解析器解析描述符

回到UsbDescriptorParser.java的parseDescriptors函数，执行完descriptor = allocDescriptor(stream)后，继续执行descriptor.parseRawDescriptors(stream)函数进行解析描述符，descriptor是上面new出来的Usb10ACHeader，因此descriptor.parseRawDescriptors(stream)函数调用的是Usb10ACHeader的parseRawDescriptors。

Usb10ACHeader.java

```java
@Override
public int parseRawDescriptors(ByteStream stream) {

    mTotalLength = stream.unpackUsbShort();
    if (mADCRelease >= 0x200) {
        mControls = stream.getByte();
    } else {
        mNumInterfaces = stream.getByte();
        mInterfaceNums = new byte[mNumInterfaces];
        for (int index = 0; index < mNumInterfaces; index++) {
            mInterfaceNums[index] = stream.getByte();
        }
    }

    return mLength;
}
```

描述符数据：13, 36, 1, 0, 1, 77, 0, -64, -31, -28, 0, 1, 1

在执行到mInterfaceNums = new byte[mNumInterfaces]时，由于mNumInterfaces的值是 -64，导致了异常，但UsbDescriptorParser.java的parseDescriptors函数捕获了该异常，因此parseDescriptors函数继续解析USB描述符。但此刻是从上面的描述符数据-31开始执行的，所以导致了后面的解析都出现问题。

# 3. 问题根因及解决方案

上面的源码分析可知，该问题的根因主要是Android系统误将Video Control Interface Header Descriptor类型的描述符解析为UAC音频的描述符，从而导致解析出现了异常，导致了后面的解析都出现了问题。

前面有说到并不是所有的UVC设备在Android9/10上的VID/PID都出现了错误，这是因为USB描述符数组数据是按将Device Descriptor的描述符数据放在最前面，而VID和PID是在Device Descriptor数据中解析的，如果解析到Video Control Interface Header Descriptor描述符数据发生了异常，从异常的描述符数据继续往后解析，如果USB描述符数据中没有再次匹配到Device Descriptor类型，就使用第一次解析的Device Descriptor中的VID和PID,因此就不会发生VID和PID异常的问题。

解决方案很简单，因为是误认为Video Control Interface Header Descriptor描述符数据为UAC音频描述符，因此在匹配到UsbDescriptor.DESCRIPTORTYPE_AUDIO_INTERFACE时，再次做一次确认是否是真正的UAC设备，修复的代码如下：

UsbDescriptorParser.java

```java
private UsbDescriptor allocDescriptor(ByteStream stream)
        throws UsbDescriptorsStreamFormatException {
    ...

       /*
        * Audio Class Specific
        */
       case UsbDescriptor.DESCRIPTORTYPE_AUDIO_INTERFACE:
          //add by wyh for fixing parser uvc's vid:pid error on 2023.07.17---start
+         //descriptor = UsbACInterface.allocDescriptor(this, stream, length,type);
+         if (mCurInterfaceDescriptor.getUsbClass() == UsbDescriptor.CLASSID_AUDIO){
+           descriptor = UsbACInterface.allocDescriptor(this, stream, length,type);
+         }
+		  //add by wyh for fixing parser uvc's vid:pid error on 2023.07.17---end
          break;
       case UsbDescriptor.DESCRIPTORTYPE_AUDIO_ENDPOINT:
+          //add by wyh for fixing parser uvc's vid:pid error on 2023.07.17---start
+          //descriptor = UsbACEndpoint.allocDescriptor(this, length, type);
+         if (mCurInterfaceDescriptor.getUsbClass() == UsbDescriptor.CLASSID_AUDIO) {
+              descriptor = UsbACEndpoint.allocDescriptor(this, length, type);
+          }
+		   //add by wyh for fixing parser uvc's vid:pid error on 2023.07.17---end
           break;
        default:
            break;
    }

    if (descriptor == null) {
        // Unknown Descriptor
        Log.i(TAG, "Unknown Descriptor len: " + length + " type:0x"
                + Integer.toHexString(type));
        descriptor = new UsbUnknown(length, type);
    }

    return descriptor;
}
```

再次使用mCurInterfaceDescriptor.getUsbClass() == UsbDescriptor.CLASSID_AUDIO判断是否是UAC音频的描述符数据，如果不是，就走到下面new UsbUnknown，UsbUnknown没有parseRawDescriptors函数，因此会调用其父类UsbDescriptor的parseRawDescriptors函数，如下：

```java
/**
 * Reads data fields from specified raw-data stream.
 */
public int parseRawDescriptors(ByteStream stream) {
    int numRead = stream.getReadCount();
    int dataLen = mLength - numRead;
    if (dataLen > 0) {
        mRawData = new byte[dataLen];
        for (int index = 0; index < dataLen; index++) {
            mRawData[index] = stream.getByte();
        }
    }
    return mLength;
}
```

UsbDescriptor的parseRawDescriptors函数没做解析动作。
