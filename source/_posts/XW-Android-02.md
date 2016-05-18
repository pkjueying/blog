---
title: XW-Android-02
toc: true
comment: true
date: 2013-12-29 11:44:12
categories: [old blog]
tags: [android]
description: DO YOUR WORK

---



在一个新的项目中，在拿到板之前，如果有固件，则获取到相关的配置，并测试相关的外围看是否正常，根据原理图以及相关的资料进行相应的修改，调试相关的驱动等等。。。

### wifi模块

在设置中的wifi模块，首先需要判断硬件是否存在故障；其次在配置文件中注意硬件的高低电平极其引脚；最后注意相关驱动的加载。
首先排除硬件，对于相关的配置下面已rtl8188etv为例：

1. 在.config中需要配置如下选项，将wifi driver 编译为模块CONFIG_RTL8188EU = m
2. 在BoardConfig.mk中
```tex
\# 1. Wifi Configuration
BOARD_WIFI_VENDOR := realtek
\#BOARD_WIFI_VENDOR := broadcom

\# 1.1 realtek wifi support
ifeq ($(BOARD_WIFI_VENDOR), realtek)
    WPA_SUPPLICANT_VERSION := VER_0_8_X
    BOARD_WPA_SUPPLICANT_DRIVER := NL80211
    BOARD_WPA_SUPPLICANT_PRIVATE_LIB := lib_driver_cmd_rtl
    BOARD_HOSTAPD_DRIVER        := NL80211
    BOARD_HOSTAPD_PRIVATE_LIB   := lib_driver_cmd_rtl

   # SW_BOARD_USR_WIFI := rtl8192cu
   # BOARD_WLAN_DEVICE := rtl8192cu

    SW_BOARD_USR_WIFI := rtl8188eu
    BOARD_WLAN_DEVICE := rtl8188eu
```
3. 在init.sun7i.rc中
```tex
\# 1.1 realtek wifi sta service
service wpa_supplicant /system/bin/wpa_supplicant -iwlan0 -Dnl80211 -c/data/misc/wifi/wpa_supplicant.conf -e/data/misc/wifi/entropy.bin
	class main
	socket wpa_wlan0 dgram 660 wifi wifi
	disabled
	oneshot
	
\# 1.2 realtek wifi sta p2p concurrent service
service p2p_supplicant /system/bin/wpa_supplicant \
	-ip2p0 -Dnl80211 -c/data/misc/wifi/p2p_supplicant.conf -e/data/misc/wifi/entropy.bin -N \
	-iwlan0 -Dnl80211 -c/data/misc/wifi/wpa_supplicant.conf
	class main
	socket wpa_wlan0 dgram 660 wifi wifi
	disabled
	oneshot
```

4. 在sys_config.fex中
```tex
[wifi_para]
wifi_used = 1
wifi_sdc_id = 3
wifi_usbc_id = 2
wifi_usbc_type = 1
wifi_mod_sel = 6
wifi_power = ""
```

### 耳机

关于耳机主要是由于时钟频率以及相关的配置，通常在sys_config.fex中进行

```tex
[audio_para]
audio_used = 1
audio_pa_ctrl = port:PH15<1><default><default><0>
audio_earphone_ctrl  = port:PH15<1><default><default><0>
headphone_vol = 0x3f
```

如果没有声音或者是一些杂音则很可能是由于相关的驱动没有加载成功，要在init.XXXX中进行相关的设置:一般--->insmod /system/vendor/xxx.ko


### 修改桌面workspace以及AppsCustomize的单元大小

找到设备对应的桌面布局文件，比如我的1024*600像素的一般在values-sw600dp下
```xml
<!-- AppsCustomize -->
    <dimen name="apps_customize_cell_width">96dp</dimen>
    <dimen name="apps_customize_cell_height">96dp</dimen>
    <dimen name="apps_customize_pageLayoutPaddingLeft">12dp</dimen>
    <dimen name="apps_customize_pageLayoutPaddingRight">12dp</dimen>
    <dimen name="apps_customize_tab_bar_height">60dp</dimen>
    <dimen name="apps_customize_tab_bar_margin_top">8dp</dimen>
    <dimen name="apps_customize_widget_cell_width_gap">20dp</dimen>
    <dimen name="apps_customize_widget_cell_height_gap">24dp</dimen>
    <dimen name="app_widget_preview_label_margin_top">8dp</dimen>
    <dimen name="app_widget_preview_label_margin_left">@dimen/app_widget_preview_padding_left</dimen>
    <dimen name="app_widget_preview_label_margin_right">@dimen/app_widget_preview_padding_right</dimen>
<!-- Workspace cell size -->
    <dimen name="workspace_cell_width_land">88dp</dimen>
    <dimen name="workspace_cell_width_port">96dp</dimen>
    <dimen name="workspace_cell_height_land">88dp</dimen>
    <dimen name="workspace_cell_height_port">96dp</dimen>
    <dimen name="workspace_width_gap_land">80dp</dimen>
    <dimen name="workspace_width_gap_port">2dp</dimen>
    <dimen name="workspace_height_gap_land">2dp</dimen>
    <dimen name="workspace_height_gap_port">70dp</dimen>
```

### 修改默认输入法

例如默认为google拼音

1、Z:\exdroid4.2-a20\android4.2\frameworks\base\packages\SettingsProvider\res\values\defaults.xml
中添加
```xml <string name="def_input_method">com.google.android.inputmethod.pinyin/.PinyinIME</string>```

2、Z:\exdroid4.2-a20\android4.2\frameworks\base\packages\SettingsProvider\src\com\android\providers\settings\DatabaseHelper.java

在
```java private void loadSecureSettings(SQLiteDatabase db) {
}```
中加上
```java loadStringSetting( stmt, Settings.Secure.DEFAULT_INPUT_METHOD,R.string.def_input_method);```
就可以了

或者是
在```java private void loadSecureSettings(SQLiteDatabase db)可以改为
            //def input method
            loadSetting(stmt, Settings.Secure.DEFAULT_INPUT_METHOD,
                    SystemProperties.get("def_input_method",
                            mContext.getResources().getString(R.string.def_input_method)));```
                            
可以实现在build.prop中配置输入，比如配置Google拼音，在build.prop中加入
def_input_method=com.android.inputmethod.pinyin/.PinyinIME
即可,这样处理可以方便使用输入法修改。

ps:在build.prop中加配置可在device配置中的mk文件(比如nuclear_xxx.mk）中处理

3.将该输入法的apk放置/system/app下，如果有lib则将其放入/system/lib/目录下

验证：
   mmm ./frameworks/base/packages/SettingsProvider/ -j16
   在/data/data/com.android.providers.settings/databases 下，sqlite3 settings.db 用select * from secure ;看见2|default_input_method|com.google.android.inputmethod.pinyin/.PinyinIME 即正确了




### 邮箱默认签名

```d
zfl@Exdroid:~/android4.2-a20/android/android4.2/packages/apps/Email/src/com/android/email/activity$ git diff MessageCompose.java 
diff --git a/src/com/android/email/activity/MessageCompose.java b/src/com/android/email/activity/MessageCompose.java
index 0ac7301..d728117 100644
--- a/src/com/android/email/activity/MessageCompose.java
+++ b/src/com/android/email/activity/MessageCompose.java
@@ -2330,6 +2330,19 @@ public class MessageCompose extends Activity implements OnClickListener, OnFocus
      *     null or has no signature, {@code null} is returned.
      */
     private static String getAccountSignature(Account account) {
-        return (account == null) ? null : account.mSignature;
+//        return (account == null) ? null : account.mSignature;
+// change by zfl 
+       String sinature = "From SANEI N60" ;
+        if(account == null){
+               return sinature;
+        }
+        if(account.mSignature == null){
+               return sinature;
+        }
+        if(account.mSignature.isEmpty()){
+               account.mSignature = sinature ;
+        }
+        return account.mSignature;
+//end by zfl 
     }
 }
```

### 去掉相机的全景模式

单
```d
diff --git a/src/com/android/camera/VideoController.java b/src/com/android/camera/VideoController.java
old mode 100644
new mode 100755
index d84c1ad..7b4f5e3
--- a/src/com/android/camera/VideoController.java
+++ b/src/com/android/camera/VideoController.java
@@ -75,7 +75,10 @@ public class VideoController extends PieController
                 }
             }
         });
-        mRenderer.addItem(item);
+        int n = CameraHolder.instance().getNumberOfCameras();
+        if (n > 1) {
+            mRenderer.addItem(item);
+        }
         mOtherKeys = new String[] {
                 CameraSettings.KEY_VIDEO_EFFECT,
                 CameraSettings.KEY_VIDEO_TIME_LAPSE_FRAME_INTERVAL,
```

双：
```d
zfl@Exdroid:~/android4.2-a20/android/android4.2/packages/apps/Camera/src/com/android/camera$ git diff CameraActivity.java
diff --git a/src/com/android/camera/CameraActivity.java b/src/com/android/camera/CameraActivity.java
index ec2c2d2..c91dd54 100755
--- a/src/com/android/camera/CameraActivity.java
+++ b/src/com/android/camera/CameraActivity.java
@@ -85,7 +85,9 @@ public class CameraActivity extends ActivityBase
                 && CameraHolder.instance().getFrontCameraId() == 0) {
             DRAW_IDS = DRAW_IDS_WITHOUT_PAN;
         } else  {
-            DRAW_IDS = DRAW_IDS_WITH_PAN;
+//            DRAW_IDS = DRAW_IDS_WITH_PAN;
+//zfl 
+             DRAW_IDS = DRAW_IDS_WITHOUT_PAN;
         }
         mDrawables = new Drawable[DRAW_IDS.length];
         for (int i = 0; i < DRAW_IDS.length; i++) {
```

### 相机字体关闭打开显示不全

```d
zfl@Exdroid:~/android4.2-a20/android/android4.2/packages/apps/Camera$ git diff ./res/values/dimens.xml
diff --git a/res/values/dimens.xml b/res/values/dimens.xml
index cb38007..ce45209 100644
--- a/res/values/dimens.xml
+++ b/res/values/dimens.xml
@@ -23,7 +23,8 @@
     <dimen name="setting_item_text_size">18sp</dimen>
     <dimen name="setting_knob_width">20dp</dimen>
     <dimen name="setting_knob_text_size">20dp</dimen>
-    <dimen name="setting_item_text_width">95dp</dimen>
+<!--  zfl  <dimen name="setting_item_text_width">95dp</dimen> -->
+    <dimen name="setting_item_text_width">115dp</dimen>
     <dimen name="setting_popup_window_width">240dp</dimen>
     <dimen name="setting_item_list_margin">14dp</dimen>
     <dimen name="indicator_bar_width">48dp</dimen>
```

### 修改桌面布局

实现办法

1. 下载 GetLayout.apk
2. adb push它们到平板电脑的/system/app中，然后后启平板电脑。
3. 在应用列表中点击GetLayout应用图标以开启该应用：

程序会自动运行，显示一个Toast提示成功生成“/data/data/com.xw.getlayout/files/home_layout.xml”的消息，然后程序自动退出。
将GetLayout.apk生成的home_layout.xml从平板电脑中抽取到本地：

```bash adb pull /data/data/com.xw.getlayout/files/home_layout.xml ~/ ```

4. 用上面克隆出来的home_layout.xml来编译Laucher2：
用home_layout.xml覆盖Launcher应用中所有的“default_workspace.xml”：

```bash
cp -rf ~/home_layout.xml packages/apps/Launcher2/res/xml/default_workspace.xml
cp -rf ~/home_layout.xml packages/apps/Launcher2/res/xml-sw480dp/default_workspace.xml
cp -rf ~/home_layout.xml packages/apps/Launcher2/res/xml-sw600dp/default_workspace.xml
cp -rf ~/home_layout.xml packages/apps/Launcher2/res/xml-sw720dp/default_workspace.xml

chmod 777 packages/apps/Launcher2/res/xml/default_workspace.xml
chmod 777 packages/apps/Launcher2/res/xml-sw480dp/default_workspace.xml
chmod 777 packages/apps/Launcher2/res/xml-sw600dp/default_workspace.xml
chmod 777 packages/apps/Launcher2/res/xml-sw720dp/default_workspace.xml
```

//重新编译Laucher2:
```bash mmm -B packages/apps/Launcher2 -j16  ```

//将新生成的Launcher2.apk安装到平板上：
```bash
adb push out/target/product/wing-k70/system/app/Launcher2.apk /system/app
```

### 修改系统内存

已1G为例：

1. 在BoardConfig.mk文件中 设置BOARD_USERDATAIMAGE_PARTITION_SIZE := 1073741824
2. 在sys_partion.fex中设置 
```tex
;------------------------------>mmcblk0p8/nande
[partition]
name         = data
    size         = 2097152
    user_type    = 0x2
```

### 视频缩放模式默认全屏

```d
zfl@Exdroid:~/android4.2-a20/android/android4.2/packages/apps/Gallery2$ git diff src/com/android/gallery3d/app/MovieViewControl.java
diff --git a/src/com/android/gallery3d/app/MovieViewControl.java b/src/com/android/gallery3d/app/MovieViewControl.java
index ea7087d..a01b9a0 100755
--- a/src/com/android/gallery3d/app/MovieViewControl.java
+++ b/src/com/android/gallery3d/app/MovieViewControl.java
@@ -1024,7 +1024,8 @@ VideoView.OnSubFocusItems{
mControlFocus = EDITOR_ZOOM;

mDialogTitle.setText(R.string.zoom_title);
- int currentMode = sp.getInt(EDITOR_ZOOM, 0);
+// int currentMode = sp.getInt(EDITOR_ZOOM, 0);
+ int currentMode = sp.getInt(EDITOR_ZOOM, 1); // zfl
int[] list = mRes.getIntArray(R.array.screen_zoom_values);
for(mListFocus = 0; mListFocus < list.length; mListFocus++) {
if(currentMode == list[mListFocus]) {
@@ -1144,7 +1145,8 @@ VideoView.OnSubFocusItems{
mVideoView.setSubPosition(offset);

/* zoom mode */
- int zoom = sp.getInt(EDITOR_ZOOM, 0);
+// int zoom = sp.getInt(EDITOR_ZOOM, 0);
+ int zoom = sp.getInt(EDITOR_ZOOM, 1); //zfl
mVideoView.setZoomMode(zoom);
```

### 锁屏界面将向下的解锁换成向右的

```d
zfl@Exdroid:~/android4.2-a20/android/android4.2/frameworks/base/core/res/res$ git diff values-sw600dp-land/arrays.xml
diff --git a/core/res/res/values-sw600dp-land/arrays.xml b/core/res/res/values-sw600dp-land/arrays.xml
index 5550216..2eabf66 100644
--- a/core/res/res/values-sw600dp-land/arrays.xml
+++ b/core/res/res/values-sw600dp-land/arrays.xml
@@ -22,35 +22,35 @@
     <!-- Resources for GlowPadView in LockScreen -->
     <array name="lockscreen_targets_when_silent">
         <item>@drawable/ic_lockscreen_unlock</item>
-        <item>@null</item>
+        <item>@drawable/ic_action_assist_generic</item>
         <item>@drawable/ic_lockscreen_soundon</item>
         <item>@null</item>
     </array>
 
     <array name="lockscreen_target_descriptions_when_silent">
         <item>@string/description_target_unlock</item>
-        <item>@null</item>
+        <item>@string/description_target_search</item>
         <item>@string/description_target_soundon</item>
         <item>@null</item>
     </array>
 
     <array name="lockscreen_direction_descriptions">
         <item>@string/description_direction_right</item>
-        <item>@null</item>
+        <item>@string/description_direction_up</item>
         <item>@string/description_direction_left</item>
         <item>@null</item>
     </array>
 
     <array name="lockscreen_targets_when_soundon">
         <item>@drawable/ic_lockscreen_unlock</item>
-        <item>@null</item>
+        <item>@drawable/ic_action_assist_generic</item>
         <item>@drawable/ic_lockscreen_silent</item>
         <item>@null</item>
     </array>
 
     <array name="lockscreen_target_descriptions_when_soundon">
         <item>@string/description_target_unlock</item>
-        <item>@null</item>
+        <item>@string/description_target_search</item>
         <item>@string/description_target_silent</item>
         <item>@null</item>
     </array>
     
```

### 默认动态壁纸

以蘑菇云为例，如下
![](/img/xw/20130909135608859.png)

### 分区相关的东东

主要涉及到两个文件：sys_partition以及BoardConfig   相关修改见下图：
![](/img/xw/20130910111249375.png)

![](/img/xw/20130910111305531.png)

结果如下图：
![](/img/xw/20130910111350093.png)

### 在build.prop中用属性值来默认动态壁纸

首先我们在build.prop文件中添加已属性值：修改pack_before.sh脚本添加两行,如下图：
![](/img/xw/20130911150346765.png)

其中红色的箭头就是动态壁纸相关的包名以及服务名
然后我们在/frameworks/base/services/java/com/android/server下的WallpaperManagerService.java 文件中作如下修改即可：
![](/img/xw/20130911150810187.png)

最后烧写固件
打印的Log日志如下：
![](/img/xw/20130911151027828.png)

### 相机拍照存储中的“打开-关闭”显示不全

```d 
zfl@Exdroid:~/common_a20/android/packages/apps/Camera$ git diff res/values/dimens.xml
diff --git a/res/values/dimens.xml b/res/values/dimens.xml
old mode 100644
new mode 100755
index cb38007..ec1bacc
--- a/res/values/dimens.xml
+++ b/res/values/dimens.xml
@@ -23,7 +23,8 @@
     <dimen name="setting_item_text_size">18sp</dimen>
     <dimen name="setting_knob_width">20dp</dimen>
     <dimen name="setting_knob_text_size">20dp</dimen>
-    <dimen name="setting_item_text_width">95dp</dimen>
+<!--  zfl  <dimen name="setting_item_text_width">95dp</dimen> -->
+    <dimen name="setting_item_text_width">120dp</dimen>
     <dimen name="setting_popup_window_width">240dp</dimen>
     <dimen name="setting_item_list_margin">14dp</dimen>
```

### 后置摄像头拍照画面拉伸

```d
zfl@Exdroid:/media/home2/zfl/common_a20/android/packages/apps/Camera$ git diff res/values/arrays.xml
diff --git a/res/values/arrays.xml b/res/values/arrays.xml
index c6a823b..7c30b77 100644
--- a/res/values/arrays.xml
+++ b/res/values/arrays.xml
@@ -29,9 +29,9 @@

6

- @string/pref_video_quality_default
+ 5

- 4
+ @string/pref_video_quality_default



zfl@Exdroid:/media/home2/zfl/common_a20/android/packages/apps/Camera$ git diff res/values/strings.xml
diff --git a/res/values/strings.xml b/res/values/strings.xml
index 23ba8f8..9861be7 100644
--- a/res/values/strings.xml
+++ b/res/values/strings.xml
@@ -86,7 +86,7 @@

Video quality

- 5
+ 4

HD 1080p
```

### 在init.sunxx.rc中新建目录以及复制相关的文件到该目录（比如竞技摩托的lib库）

下面就已竞技摩托为例：场景是在固件中将竞技摩托放在system 的 app下，不能卸载，由于该app的lib库是在特定的目录下寻找，找不到直接报错，而不会再到system/lib下继续寻找，所以我们只能将其相关的Lib库放置在其目录下：
```d
zfl@Exdroid:~/common_a20/android/device/softwinner/wingchiphd-other$ git diff init.sun7i.rc
diff --git a/init.sun7i.rc b/init.sun7i.rc
index 786309d..14f3e4e 100755
--- a/init.sun7i.rc
+++ b/init.sun7i.rc
@@ -42,7 +42,7 @@ on fs
     exec /system/bin/logwrapper /system/bin/e2fsck -y /dev/block/cache
     exec /system/bin/busybox mount -t ext4 -o noatime,nosuid,nodev,barrier=0,journal_checksum,noauto_da_alloc /dev/block/cache /cac
 
-    format_userdata /dev/block/UDISK CT720
+    format_userdata /dev/block/UDISK CT720K
 # zfl
     format_userdata /dev/block/private PRIVATE
     mkdir /mnt/private 0770 system system
@@ -61,6 +61,22 @@ on boot
        insmod /system/vendor/modules/sw_device.ko
        chmod 777 /data/misc
 
+mkdir /data/app-lib/RiptideGP
+chown root root /data/app-lib/RiptideGP
+chmod 0755 /data/app-lib/RiptideGP
+
+copy /system/lib/libBlue.so  /data/app-lib/RiptideGP/libBlue.so
+chown root root /data/app-lib/RiptideGP/libBlue.so
+chmod 0755 /data/app-lib/RiptideGP/libBlue.so
+
+copy /system/lib/libfmodex.so  /data/app-lib/RiptideGP/libfmodex.so
+chown root root /data/app-lib/RiptideGP/libfmodex.so
+chmod 0755 /data/app-lib/RiptideGP/libfmodex.so
+
+copy /system/lib/libfmodevent.so /data/app-lib/RiptideGP/libfmodevent.so
+chown root root /data/app-lib/RiptideGP/libfmodevent.so
+chmod 0755 /data/app-lib/RiptideGP/libfmodevent.so
```

### 按键音

```d
zfl@Exdroid:~/common_a20/android/frameworks/base$ git diff policy/src/com/android/internal/policy/impl/PhoneWindowManager.java
diff --git a/policy/src/com/android/internal/policy/impl/PhoneWindowManager.java b/policy/src/com/android/internal/policy/impl/Phone
index fc9bd0f..8635010 100755
--- a/policy/src/com/android/internal/policy/impl/PhoneWindowManager.java
+++ b/policy/src/com/android/internal/policy/impl/PhoneWindowManager.java
@@ -3565,6 +3565,12 @@ public class PhoneWindowManager implements WindowManagerPolicy {
             return 0;
         }
         final boolean down = event.getAction() == KeyEvent.ACTION_DOWN;
+// zfl
+       if(!down) {
+               AudioManager am = (AudioManager)mContext.getSystemService(Context.AUDIO_SERVICE);
+               am.playSoundEffect(android.view.SoundEffectConstants.CLICK);
+       }
+
         final boolean canceled = event.isCanceled();


桌面Workspace文本大小设置

zfl@Exdroid:~/common_a20/android/packages/apps/Launcher2$ git diff src/com/android/launcher2/BubbleTextView.java
diff --git a/src/com/android/launcher2/BubbleTextView.java b/src/com/android/launcher2/BubbleTextView.java
index 610aedc..c86b49e 100755
--- a/src/com/android/launcher2/BubbleTextView.java
+++ b/src/com/android/launcher2/BubbleTextView.java
@@ -55,6 +55,8 @@ public class BubbleTextView extends TextView {
     private int mPressedOutlineColor;
     private int mPressedGlowColor;
 
+       private static final int bubbleTextInt = 14 ; 
+
     private boolean mBackgroundSizeChanged;
     private Drawable mBackground;
 
@@ -79,6 +81,9 @@ public class BubbleTextView extends TextView {
     private void init() {
         mLongPressHelper = new CheckLongPressHelper(this);
         mBackground = getBackground();
+       //zfl
+       super.setTextSize(bubbleTextInt) ;
+
```

### 在配置中修改浏览器模式

首先在build.prop中设置属性：ro.browser.user_agent
然后在java中修改相关代码即可：

```d
zfl@Exdroid:~/common_a20/android/packages/apps/Browser$ git diff ./src/com/android/browser/preferences/BrowserModeFragment.java    
diff --git a/src/com/android/browser/preferences/BrowserModeFragment.java b/src/com/android/browser/preferences/BrowserModeFragment.
index ab7a411..9e4bc40 100755
--- a/src/com/android/browser/preferences/BrowserModeFragment.java
+++ b/src/com/android/browser/preferences/BrowserModeFragment.java
@@ -45,8 +45,12 @@ public class BrowserModeFragment extends Fragment implements
                LayoutParams lp = new LayoutParams(LayoutParams.MATCH_PARENT,
                                LayoutParams.WRAP_CONTENT);
                radioGroup.setLayoutParams(lp);
-               String def = mPref.getString(PreferenceKeys.PREF_USER_AGENT,
-                               "3");
+
+               //add by zfl 
+               String user_agent = android.os.SystemProperties.get("ro.browser.user_agent" , "3") ;
+
+               String def = mPref.getString(PreferenceKeys.PREF_USER_AGENT, user_agent);
+
                if (mEntries != null && mEntryValues != null) {
```

### 去掉相机拍照时手势的缩放功能

```d
zfl@Exdroid:~/common_a20/android/packages/apps/Camera$ git diff src/com/android/camera/ui/ZoomRenderer.java
diff --git a/src/com/android/camera/ui/ZoomRenderer.java b/src/com/android/camera/ui/ZoomRenderer.java
index 10c5e80..6643634 100644
--- a/src/com/android/camera/ui/ZoomRenderer.java
+++ b/src/com/android/camera/ui/ZoomRenderer.java
@@ -50,8 +50,8 @@ public class ZoomRenderer extends OverlayRenderer
     private Rect mTextBounds;
 
     public interface OnZoomChangedListener {
-        void onZoomStart();
-        void onZoomEnd();
+       void onZoomStart();
+       void onZoomEnd();
         void onZoomValueChanged(int index);  // only for immediate zoom
     }
 
@@ -125,7 +125,7 @@ public class ZoomRenderer extends OverlayRenderer
 
     @Override
     public boolean onScale(ScaleGestureDetector detector) {
-        final float sf = detector.getScaleFactor();
+      /*  final float sf = detector.getScaleFactor();
         float circle = (int) (mCircleSize * sf * sf);
         circle = Math.max(mMinCircle, circle);
         circle = Math.min(mMaxCircle, circle);
@@ -134,25 +134,28 @@ public class ZoomRenderer extends OverlayRenderer
             int zoom = mMinZoom + (int) ((mCircleSize - mMinCircle) * (mMaxZoom - mMinZoom) / (mMaxCircle - mMinCircle));
             mListener.onZoomValueChanged(zoom);
         }
+       */
         return true;
     }
 
     @Override
     public boolean onScaleBegin(ScaleGestureDetector detector) {
-        setVisible(true);
+  /*      setVisible(true);
         if (mListener != null) {
             mListener.onZoomStart();
         }
         update();
+       */
         return true;
     }
 
     @Override
     public void onScaleEnd(ScaleGestureDetector detector) {
-        setVisible(false);
+ /*       setVisible(false);
         if (mListener != null) {
             mListener.onZoomEnd();
         }
+*/
     }
 
```






