---
layout: post
tags: Android DeviceId 隐私权限
date: 2022-9-21
thumbnail: assets/img/f001.png
title: Android生成设备唯一ID方案
published: true
---

设备唯一ID对业务有非常重要的作用，比如新用户注册归因和业务风控，都对设备ID的稳定性有着很高的要求。稳定性主要表现在两个方面:
1. 同一台设备，卸载重装APP，生成ID不变；
2. 同一台设备，不同的APP生成的ID是一致的。
要保持这种稳定性，主要由两种策略来实现，一种是在设备上持久化保存ID。一种是通过设备较为稳定的属性作为因子生成ID。

<!--more-->

### Android持久化分析
- SD卡公共目录保存，即使卸载APP也能还原hdid，但依赖业务动态申请SD卡访问权限才能生效。
SDCard访问权限演变：
- Android 6（API 23）以下只需要在清单文件申请权限。
- Android 6（API 23）以上需要APP向业务动态申请访问权限。
- Android 10（API 29）以上需要清单文件中application节点加上。android:requestLegacyExternalStorage="true"属性。
- Android 11 开始APP默认只能访问沙盒目录，无法访问公共SDCard目录，如果以Android10以下为目标平台仍可以兼容访问（参考[《Android 11 中的存储机制更新》](https://developer.android.com/about/versions/11/privacy/storage?hl=zh-cn)）。

#### 总结：
随着系统权限的逐步收紧，SDCard公共目录保存也将逐渐被淘汰，要在卸载App后依然保存数据在手机上基本没有完全可靠的方案。

### 生成因子分析
- Android_id：
  - 唯一决定于应用签名、用户和设备三者的组合
  - AndroidID较为稳定，但无法跨业务，如果业务APP签名不同则产生的设备ID不一样。
  - 参考：[https://developer.android.com/about/versions/oreo/android-8.0-changes.html](https://developer.android.com/about/versions/oreo/android-8.0-changes.html)
  - 参考：[https://www.cnblogs.com/kekec/p/12544674.html](https://www.cnblogs.com/kekec/p/12544674.html)
- Mac地址：
  - 需要申明ACCESS_WIFI_STATE，但Android 10.0后获取到的是随机生成的MAC（参考《Android随机分配 MAC 地址》）。
  - 实测在Android10设备上，切换wifi就会发生改变。
  - Mac在Android8.0以后获取的值不可靠。
- IMEI：
  - Android 10以下系统，需要动态申请电话权限(READ_PHONE_STATE)，
  - Android 10以上无法获取，需要READ_PRIVILEGED_PHONE_STATE权限，但该权限需要系统特许的或者官方应用才能获取
  - 参考：设备标识符官方说明
- OAID
  - 在高版本系统增加作为因子，国内厂商普遍在Android10之后开始支持OAID：
<div>    
	<img src="https://raw.githubusercontent.com/ruilin/blog/master/assets/img/d002.png"/>
</div>

### 最终方案
- 优先使用OAID+Host，可以确保设备ID的稳定性，并且实现跨APP。
- 低版本系统多数设备无法获取OAID，采用Mac + Build.SERIAL 作为因子实现。
- 一旦前面都获取失败，则采用兜底方案：Android_id + Build.Host。

<div>    
	<img src="https://raw.githubusercontent.com/ruilin/blog/master/assets/img/d003.jpg" style="width: 600px"/>
</div>

需要注意的是获取OAID结果回调才生成设备ID，由于OAID在有些设备上获取比较慢，甚至没有回调（内部是通过AIDL跨进程调系统服务获取），因此需要设置一个超时时间（比如1秒），超时则放弃使用，改为兜底方案。

### 验证结果
<div>    
	<img src="https://raw.githubusercontent.com/ruilin/blog/master/assets/img/d005.jpg"/>
</div>


