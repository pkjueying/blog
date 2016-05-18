---
title: XW-Android-01
toc: true
comment: true
date: 2013-12-18 20:24:48
categories: [old blog]
tags: [work]
description: DO YOUR WORK

---



### 关机充电图标不显示：

现象：
开机后，插入充电器，再按正常流程关机。 关机后，短按power键，屏幕无充电图标显示(此时充电器还是插入的)。

解决方法：
配置 [pmu_para] power_start = 0 即可


### data分区扩到4G 恢复出厂设置就没作用了

解决方法：

```d

diff --git a/drivers/mtd/nand/sunxi_nand.c b/drivers/mtd/nand/sunxi_nand.c
old mode 100644
new mode 100755
index 10b672a..469ce7c
--- a/drivers/mtd/nand/sunxi_nand.c
+++ b/drivers/mtd/nand/sunxi_nand.c
@ -38,7 +38,7 @ int sunxi_nand_read_opts(nand_info_t *nand, uint offset, size_t *length,
unsigned int nSectNum,  nSectorCnt;
- nSectNum = (unsigned int)(offset / 512);
+ nSectNum = (unsigned int)(offset );
nSectorCnt = (unsigned int)(*length / 512);
\#ifdef DEBUG
printf("sunxi nand read: start sector %x counts %x ", nSectNum, nSectorCnt);
@ -51,7 +51,7 @ int sunxi_nand_write_opts(nand_info_t *nand, uint offset, size_t *length,
unsigned int nSectNum,  nSectorCnt;
- nSectNum = (unsigned int)(offset / 512);
+ nSectNum = (unsigned int)(offset);
nSectorCnt = (unsigned int)(*length / 512);
\#ifdef DEBUG
printf("sunxi nand write: start sector %x counts %x ", nSectNum, nSectorCnt);
diff --git a/drivers/storage_type/sunxi_flash.c b/drivers/storage_type/sunxi_flash.c
index c78c392..2c9cf17 100755
--- a/drivers/storage_type/sunxi_flash.c
+++ b/drivers/storage_type/sunxi_flash.c
@ -66,7 +66,7 @ sunxi_flash_nand_read(unsigned int start_block, unsigned int nblock, void *buffe
tick_printf(FILE, LINE);
        nsize = nblock<<9;
-       ret = sunxi_nand_read_opts(&nand_info[0], start_block<<9, &nsize, buffer, 0);
+       ret = sunxi_nand_read_opts(&nand_info[0], start_block, &nsize, buffer, 0);
        tick_printf(FILE, LINE);
return ret;
@ -77,7 +77,7 @ sunxi_flash_nand_write(unsigned int start_block, unsigned int nblock, void *buff
        unsigned int nsize;
nsize = nblock<<9;
-       return sunxi_nand_write_opts(&nand_info[0], start_block<<9, &nsize, buffer, 0);
+       return sunxi_nand_write_opts(&nand_info[0], start_block, &nsize, buffer, 0);
 }
```
 

### 支持多用户功能

```xml
<integer name="config_multiuserMaximumUsers">8</integer>
```

### 在首次开机时默认将Play商店停用

修改：packages\apps\Provision，在这里将vending应用停用
```java
        PackageManager pm = getPackageManager();
        try {
        	pm.setApplicationEnabledSetting("com.android.vending", 				PackageManager.COMPONENT_ENABLED_STATE_DISABLED_USER, 0);
        } catch(Exception e) {
        	android.util.Log.i("chen", "change fail");
        }         
```
并在Provision应用的AndroidManifest.xml中增加权限
```xml
<uses-permission android:name="android.permission.CHANGE_COMPONENT_ENABLED_STATE" />
```

### 4.1系统在播放视频时，视频切换会出现花屏：

在 device\softwinner\common\hardware\libhardware\hwcomposer\hwcomposer.cpp  里

```cpp
static int hwc_show(sun4i_hwc_context_t *ctx,uint32_t value)
{
	unsigned long               args[4]={0};
    unsigned int                screen_idx;
    int ret = 0;
 
      for(screen_idx=0; screen_idx<2; screen_idx++)
      {
          if(((screen_idx == 0) && (ctx->mode==HWC_MODE_SCREEN0 || ctx->mode==HWC_MODE_SCREEN0_AND_SCREEN1 ))
              || ((screen_idx == 1) && (ctx->mode == HWC_MODE_SCREEN1 || ctx->mode==HWC_MODE_SCREEN0_TO_SCREEN1 || ctx->mode==HWC_MODE_SCREEN0_AND_SCREEN1)))
          {
              if(ctx->video_layerhdl[screen_idx] != 0)
              {
                  if(value == 0)
                  {
                      args[0]                         = screen_idx;
                      args[1]                         = ctx->video_layerhdl[screen_idx];
                      ioctl(ctx->dispfd, DISP_CMD_LAYER_CLOSE,args);
 
                      args[0]                         = screen_idx;
                      args[1]                         = ctx->video_layerhdl[screen_idx];
                      ret = ioctl(ctx->dispfd, DISP_CMD_VIDEO_STOP, args);
 
                  }
                  else
                  {
                      args[0]                         = screen_idx;
                      args[1]                         = ctx->video_layerhdl[screen_idx];
                      ioctl(ctx->dispfd, DISP_CMD_LAYER_OPEN,args);
 
                      args[0]                         = screen_idx;
                      args[1]                         = ctx->video_layerhdl[screen_idx];
                      ret = ioctl(ctx->dispfd, DISP_CMD_VIDEO_START, args);
                  }
              }
          }
      }
 
   return ret;
}
```
 
在 hwc_setparameter 函数加上
```cpp
}else if(param == HWC_LAYER_SHOW){
ret = hwc_show(ctx,value);
}
```

### 添加新的产品编译项：

以abcd为例

```bash
cp -r crane-m1003h6 crane-abcd
cd crane-abcd
grep -r -l "m1003h6" ./* | xargs sed -i 's/m1003h6/abcd/g'
mv crane_m1003h6.mk crane_abcd.mk
```

### 播放Mp3，屏幕暗下去以后，调节音量键无效

去掉isScreenOn的判断

```d
a/policy/src/com/android/internal/policy/impl/PhoneWindowManager.java
+++ b/policy/src/com/android/internal/policy/impl/PhoneWindowManager.java
@@ -3352,7 +3352,7 @@ public class PhoneWindowManager implements WindowManagerPolicy {
             case KeyEvent.KEYCODE_VOLUME_MUTE: {
                 if (keyCode == KeyEvent.KEYCODE_VOLUME_DOWN) {
                     if (down) {
-                        if (isScreenOn && !mVolumeDownKeyTriggered
+                        if (!mVolumeDownKeyTriggered
                                 && (event.getFlags() & KeyEvent.FLAG_FALLBACK) == 0) {
                             mVolumeDownKeyTriggered = true;
                             mVolumeDownKeyTime = event.getDownTime();
@@ -3366,7 +3366,7 @@ public class PhoneWindowManager implements WindowManagerPolicy {
                     }
                 } else if (keyCode == KeyEvent.KEYCODE_VOLUME_UP) {
                     if (down) {
-                        if (isScreenOn && !mVolumeUpKeyTriggered
+                        if (!mVolumeUpKeyTriggered
                                 && (event.getFlags() & KeyEvent.FLAG_FALLBACK) == 0) {
                             mVolumeUpKeyTriggered = true;
                             cancelPendingPowerKeyAction();
```

不让按键进入suspend

```d

--- a/drivers/input/keyboard/sun4i-keyboard.c
+++ b/drivers/input/keyboard/sun4i-keyboard.c
@@ -183,7 +183,7 @@ static void sun4i_keyboard_suspend(struct early_suspend *h)
     
        if (NORMAL_STANDBY == standby_type) {
\-               writel(0,KEY_BASSADDRESS + LRADC_CTRL);
\+               //writel(0,KEY_BASSADDRESS + LRADC_CTRL);
        /*process for super standby*/   
        } else if(SUPER_STANDBY == standby_type) {
                ;
@@ -466,5 +466,5 @@ MODULE_AUTHOR(" <@>");
```

### Google Play 搜不到某个应用

google play搜不到某个应用的分析和解决方法

   搜不到应用是设备缺少该应用所必须的权限，即使安装了也可能出现部分功能不能正常使用的情况。如何让A1X搜到应用，下面以搜索IDTV应用为例：
   
1. 先在pc上打开google play搜到IDTV这个应用
2. 查看这个应用所需要的权限，需要定位，WIFI等
3. 查看设备所具有的权限etc/permissions/*
4. 比较找到设备所缺少的权限，例如A10默认缺少IDTV所必须的定位权限。
5. 将可疑的权限文件push到etc/permissions/下，恢复出厂设置再次测试能否搜到。
6. 在项目中拷贝所需的权限，例如打开网络定位的功能。

如果难以定位缺少所需的权限，其他手机可搜索到，比较etc/permissions/*下的内容，反复push，恢复出厂设置后测试。
所有权限文件路径：frameworks/native/data/etc/*

### Google Play 上下载的水果忍者运行出错

下载的水果忍者带有广告， 会请求定位服务。没有gps，所以要保证有网络定位功能。

请检查 config_networkLocationProvider 是否为 null

修改： 

```xml
<string name="config_networkLocationProviderPackageName" translatable="false">com.google.android.location</string>
```

### 内存使用状态显示问题：

现象：
在settings->apps->观察internal storage 显示信息（有显示已用空间与可用空间的数值）->点击右边任意一个应用->返回->再观察 internal storage 显示信息（无已用空间与可用空间的数值显示）

解决方法：
```d
diff --git a/src/com/android/settings/applications/ManageApplications.java b/src/com/android/settings/applications/ManageApplications.java
--- a/src/com/android/settings/applications/ManageApplications.java
+++ b/src/com/android/settings/applications/ManageApplications.java
@@ -855,13 +855,16 @@ public class ManageApplications extends Fragment implements
mColorBar.setRatios((totalStorage-freeStorage-appStorage)/(float)totalStorage,
appStorage/(float)totalStorage, freeStorage/(float)totalStorage);
long usedStorage = totalStorage - freeStorage;
- if (mLastUsedStorage != usedStorage) {
+	
+ //if (mLastUsedStorage != usedStorage) {
+ if (true) {
mLastUsedStorage = usedStorage;
String sizeStr = Formatter.formatShortFileSize(getActivity(), usedStorage);
mUsedStorageText.setText(getActivity().getResources().getString(
R.string.service_foreground_processes, sizeStr));
}
- if (mLastFreeStorage != freeStorage) {
+ //if (mLastFreeStorage != freeStorage) {
+ if (true) {
mLastFreeStorage = freeStorage;
String sizeStr = Formatter.formatShortFileSize(getActivity(), freeStorage);
mFreeStorageText.setText(getActivity().getResources().getString(
```


### 隐藏状态栏：

如果有应用程序的源码，可以在需要隐藏状态栏的activity中添加

```xml
<activity android:name="xxx"
		  android:label="@string/app_name"
          android:theme="@android:style/Theme.NoTitleBar.Fullscreen" >                   
```

如果没有应用的源码，在PhoneWindowManager.java文件 finishAnimationLw函数中，针对特殊的应用判断。已有视频播放的参考代码，参照修改即可。


### 预装APK过大，升级固件在一定百分比出错

可以试一下做如下修改

1. 修改 BoardConfig.mk
BOARD_SYSTEMIMAGE_PARTITION_SIZE 要比预装的总空间大。
 
2. 通过更改devices/softwinner/crane_xx/package.sh如下

```bash
mv out/target/product/crane-evb_mmc/system.img out/target/product/crane-evb_mmc/system.img_old
simg2img out/target/product/crane-evb_mmc/system.img_old out/target/product/crane-evb_mmc/system.img
cd $PACKAGE
./pack -c sun4i -p crane -b evb_mmc
cd -
```


### 蓝牙可用设备会显示之前搜索到但已关闭的设备

蓝牙的缓存管理CachedBluetoothDeviceManager中， 只有添加设备， 没有清除。

修改如下：

```java
//在CachedBluetoothDeviceManager中添加删除接口
        void removeAllCacheDevices() {
            synchronized(this) {
                List<CachedBluetoothDevice> mCachedList = mCachedDevices;
                for (int i = mCachedList.size() - 1; i >= 0; i--) {
                    mCachedList.remove(mCachedList.get(0));
                }
            }
        }
//在startScanning的地方调用，调用方式
         mLocalManager = LocalBluetoothManager.getInstance(context);
         if (mLocalManager == null) {
             Log.e(TAG, "Bluetooth is not supported on this device");
             return;
         }
   mLocalManager.getCachedDeviceManager().removeAllCacheDevices();
```

### 修改Launcher显示的列数

将Launcher修改为7列, 如下两种方法均可.

方法1: workspace.xml 文件中, 在 launcher:defaultScreen="2" 的下面增加一行: launcher:cellCountX="7"

方法2: dimens.xml 文件中, 修改workspace_cell_width的值. 改小, 使屏幕宽度除以这个值大于7.

### 状态栏的日期格式不随设置的日期格式改变

修改如下:
文件: frameworks\base\packages\SystemUI\src\com\android\systemui\statusbar\policy\DateView.java
函数: updateClock
CharSequence date = DateFormat.getLongDateFormat(context).format(now); // java语言长日期格式
改为
CharSequence date = DateFormat.getDateFormat(context).format(now);  // 从设置读取日期格式


### 超请播放器分享功能相关问题

现象：
在A13的1.5版本上，客户在美国那边发现在超清播放器里面图片分享的功能，如果只安装了gmail或者email的时候无法使用，但是安装了比如facebook、twitter这些东西之后就可以使用了

解决方法：
在 frameworks\base\core\java\android\widget\ActivityChooserView.java
 public ActivityChooserView(Context context, AttributeSet attrs, int defStyle) {  这个函数大概 239 行，
 
 mAdapter = new ActivityChooserViewAdapter(); 加上
 
 mAdapter.setShowFooterView(true);

### Google Chrome 进Setting 停止运行

现象：
通过play store下载安装Google Chrome。打开该浏览器，点右上角菜单，选择“Settings”，出现停止运行提示。

解决方法：
与尺寸有关,小尺寸有问题.使用的preference_headers.xml不同.

```java
//修改文件 frameworks\base\core\java\android\preference\PreferenceActivity.java
    public Header onGetInitialHeader() {
    // 添加这个for循环
    for (Header h : mHeaders) {
            if (h.fragment != null) {
                return h;
            }
        }
        return mHeaders.get(0);
    }
```

### 升级后首次开机不显示音量图标

现象：
升级后首次开机需要先待机锁屏再解锁，才能显示音量图标

解决方法：
```java
//在 TabletStatusBar.java 文件的 makeStatusBarView 函数中,
mVolumeDownButton = mNavigationArea.findViewById(R.id.volume_down);
mVolumeUpButton = mNavigationArea.findViewById(R.id.volume_up);
// 增加下面这个if分支
if(mContext.getResources().getBoolean(R.bool.hasVolumeButton))
{
mVolumeUpButton.setVisibility(View.VISIBLE);
mVolumeDownButton.setVisibility(View.VISIBLE);
}
```

### 视频实时旋转

```java

//1.frameworks\base\services\surfaceflinger\SurfaceTextureLayer.cpp 中修改一下函数
uint32_t SurfaceTextureLayer::getParameter(uint32_t cmd) 
{
    uint32_t res = 0;
    
/* if(cmd == NATIVE_WINDOW_CMD_GET_SURFACE_TEXTURE_TYPE) {
return 1;
}
*/ 
    sp<Layer> layer(mLayer.promote());
    if (layer != NULL) 
    {
        res = layer->getDisplayParameter(cmd);
    }
    return res;
}
 
 
//2.在frameworks\base\policy\src\com\android\internal\policy\impl\PhoneWindowManager.java  中
rotationForOrientationLw() 函数里 注释掉多余的
// if(MediaPlayer.isPlayingVideo())
// {
// Log.i(TAG, "MediaPlayer.isPlayingVideo");
// if(SystemProperties.getInt("ro.sf.hwrotation",0)==270)
// {
// sensorRotation = Surface.ROTATION_90;
// }
// else
// {
// sensorRotation = Surface.ROTATION_0;
// }
// }
// else
// {
Log.i(TAG, "MediaPlayer.is not PlayingVideo");
sensorRotation = mOrientationListener.getProposedRotation(); // may be -1
// }


// 3.修改 packages\apps\Gallery2\AndroidManifest.xml  修改
        <activity android:name="com.android.gallery3d.app.MovieActivity"
                android:label="@string/movie_view_label"
android:configChanges="keyboardHidden|orientation"
                android:theme="@style/Theme.Movie" >
```

### 开启wifi连3G会出现wifi和3G两个图标，退出3G后3G图标不消失

修改如红色代码所示：

./frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/policy/NetworkController.java

```java

 void refreshViews() {
        Context context = mContext;
        int combinedSignalIconId = 0;
        int combinedActivityIconId = 0;
        String combinedLabel = "";
        String wifiLabel = "";
        String mobileLabel = "";
        int N;
         if (mWifiConnected && !mDataConnected) {
             mDataSignalIconId = mPhoneSignalIconId = 0;
            mobileLabel = "";
        }
  
      if (!mHasMobileDataFeature) {
            mDataSignalIconId = mPhoneSignalIconId = 0;
            mobileLabel = "";
        } else {
```


### 解锁界面应用图标特大

frameworks\base\core\java\com\android\internal\widget\multiwaveview\MultiWaveView.java
把下面代码红色部分注释掉

```java

public void setTargetResources(ArrayList<Drawable> drawables){       Resources res = getContext().getResources();       int count = drawables.size();  Drawable drawable = null;       ArrayList<TargetDrawable> targetDrawables = new ArrayList<TargetDrawable>(count);  if(count > mMaxAppIconNum)  count = mMaxAppIconNum;        for (int i = 0; i < count; i++) {if(i!=0){            drawable = zoomDrawable(drawables.get(i),mIconSize,mIconSize);}else{   drawable = drawables.get(i);}  public void setTargetResources(ArrayList<Drawable> drawables)
{
Resources res = getContext().getResources();
int count = drawables.size();
Drawable drawable = null;
 ArrayList<TargetDrawable> targetDrawables = new ArrayList<TargetDrawable>(count);
if(count > mMaxAppIconNum)
count = mMaxAppIconNum;
for (int i = 0; i < count; i++) {
// if(i!=0)
// {
// drawable = zoomDrawable(drawables.get(i),mIconSize,mIconSize);
// }else
// {
drawable = drawables.get(i);
//}
}

```

### 高清播放器图片预览时，不是居中

修改
 packages\apps\Gallery2\src\com\android\gallery3d\ui\SlideshowView.java  
 
```java
public void apply(GLCanvas canvas) {
//            float centerX = viewWidth / 2 + mMovingVector.x * mProgress;//            float centerY = viewHeight / 2 + mMovingVector.y * mProgress;                        float centerX = viewWidth / 2 ;            float centerY = viewHeight / 2 ;//            float centerX = viewWidth / 2 + mMovingVector.x * mProgress;//            float centerY = viewHeight / 2 + mMovingVector.y * mProgress;                        float centerX = viewWidth / 2 ;            float centerY = viewHeight / 2 ;约 152 行
    // float centerX = viewWidth / 2 + mMovingVector.x * mProgress;
   //  float centerY = viewHeight / 2 + mMovingVector.y * mProgress;
  float centerX = viewWidth / 2 ;
  float centerY = viewHeight / 2 ;    
```

### 休眠间歇性唤醒

现象：
在standby时，系统间歇性会被唤醒

解决方法：
由于android本身timer机制导致，可以通过如下配置进行优化：

1.  进入menuconfig
2.  选择device drivers -> real time clock
3.  选择android alarm clock wakeup

### 音乐播放器中，均衡器界面不起作用

现象：
音乐播放器，选择 ‘音效’ ，出现 均衡器 界面，界面上所有操作都不起作用

解决方法：
修改
packages\apps\MusicFX\src\com\android\musicfx\ActivityMusic.java 
```java
protected void onResume() { 
//中 460 行 
\\ mIsHeadsetOn = (audioManager.isWiredHeadsetOn() || audioManager.isBluetoothA2dpOn());
// 改为 mIsHeadsetOn = true;

 private void updateUI() {
//中 551 行
/**
final boolean isEnabled = ControlPanelEffect.getParameterBoolean(mContext,                mCallingPackageName, mAudioSession, ControlPanelEffect.Key.global_enabled);final boolean isEnabled = ControlPanelEffect.getParameterBoolean(mContext,                mCallingPackageName, mAudioSession, ControlPanelEffect.Key.global_enabled);final boolean isEnabled = ControlPanelEffect.getParameterBoolean(mContext,                mCallingPackageName, mAudioSession, ControlPanelEffect.Key.global_enabled);  * final boolean isEnabled = ControlPanelEffect.getParameterBoolean(mContext,
* mCallingPackageName, mAudioSession, ControlPanelEffect.Key.global_enabled);
**/
// 改为
  final boolean isEnabled = ControlPanelEffect.getParameterBoolean(mContext,
 mCallingPackageName, mAudioSession, ControlPanelEffect.Key.virt_enabled);
```

### 修改播放器拖动时的状态条宽度

修改
android4.0\packages\apps\Gallery2\res\layout\media_status.xml

```xml
        <SeekBar            android:id="@+id/mediacontroller_progress"            style="?android:attr/progressBarStyleHorizontal"            android:layout_width="0dip"            android:layout_weight="1"            android:paddingTop="30dp"            android:layout_height="60dip" />        <SeekBar            android:id="@+id/mediacontroller_progress"            style="?android:attr/progressBarStyleHorizontal"            android:layout_width="0dip"            android:layout_weight="1"            android:paddingTop="30dp"            android:layout_height="60dip" />        <SeekBar            android:id="@+id/mediacontroller_progress"            style="?android:attr/progressBarStyleHorizontal"            android:layout_width="0dip"            android:layout_weight="1"            android:paddingTop="30dp"            android:layout_height="60dip" />        <SeekBar            android:id="@+id/mediacontroller_progress"            style="?android:attr/progressBarStyleHorizontal"            android:layout_width="0dip"            android:layout_weight="1"            android:paddingTop="30dp"            android:layout_height="60dip" />        <SeekBar            android:id="@+id/mediacontroller_progress"            style="?android:attr/progressBarStyleHorizontal"            android:layout_width="0dip"            android:layout_weight="1"            android:paddingTop="30dp"            android:layout_height="60dip" />        <SeekBar            android:id="@+id/mediacontroller_progress"            style="?android:attr/progressBarStyleHorizontal"            android:layout_width="0dip"            android:layout_weight="1"            android:paddingTop="30dp"            android:layout_height="60dip" />        <SeekBar            android:id="@+id/mediacontroller_progress"            style="?android:attr/progressBarStyleHorizontal"            android:layout_width="0dip"            android:layout_weight="1"            android:paddingTop="30dp"            android:layout_height="60dip" />        <SeekBar            android:id="@+id/mediacontroller_progress"            style="?android:attr/progressBarStyleHorizontal"            android:layout_width="0dip"            android:layout_weight="1"            android:paddingTop="30dp"            android:layout_height="60dip" />        <SeekBar            android:id="@+id/mediacontroller_progress"            style="?android:attr/progressBarStyleHorizontal"            android:layout_width="0dip"            android:layout_weight="1"            android:paddingTop="30dp"            android:layout_height="60dip" />        <SeekBar            android:id="@+id/mediacontroller_progress"            style="?android:attr/progressBarStyleHorizontal"            android:layout_width="0dip"            android:layout_weight="1"            android:paddingTop="30dp"            android:layout_height="60dip" /><SeekBar
     android:id="@+id/mediacontroller_progress"
     style="?android:attr/progressBarStyleHorizontal"
     android:layout_width="0dip"
    android:layout_weight="1"
    android:paddingTop="30dp"
    android:layout_height="60dip" />  
```

数值根据需要进行调整。

### Music播放器无法点击前一首

修改 android4.0\packages\apps\Music\src\com\android\music\MediaPlaybackActivity.java
 431行 onClick（）函数里
 ```java
 if (mService == null) return;
            try {
                    mService.prev();
                }
            } catch (RemoteException ex) {
            }
```

### Google拼音输入法，横屏时有些应用没有中文选项

现象：

google拼音版本1.2版. 在launcher自定义文件夹名称, email输入邮件主题等横屏输入中文时, 不显示中文候选框, 无法输入.

解决方法：

在横屏时全屏来输入, 去掉相应布局文件中的 android:imeOptions="flagNoExtractUi".


### 播放电影字幕与菜单重叠

现象：

播放外带字幕电影，播放电影且已有字幕出现在屏幕上时触摸屏幕调出子菜单，字幕和菜单重叠，下一条字幕出现时则恢复正常。

解决方法：

修改 Gallery2\src\com\android\gallery3d\app\MediaController.java  在initControllerView  函数最后加 mUpSubPos += 10; 即可


### 卡启动，Nand分区识别

现象：

卡启动时，如何识别到nand并对其进行操作

解决方法：

关于要在卡启动识别到nand，有两种情况：

1. nand上已经有分区；
2. nand是裸片；
 
 第一种情况下卡启动时nand驱动能够初始化成功，用户可以通过mount命令挂载nand的分区；
 第二种情况下要先给nand一个虚拟的mbr，使得启动时nand能够初始化成功，
可以通过修改nand驱动的源码达到这一目的。

 A10平台修改linux-3.0/drivers/block/sun4i_nand/nfd/mbr.c
A1X平台修改linux-3.0/drivers/block/sun5i_nand/nfd/mbr.c

具体改动如下：

```cpp
int mbr2disks(struct nand_disk* disk_array)
{
         int part_cnt = 0;
         int part_index;
\#if 0
         if(_get_mbr()){
                   printk("get mbr error\n" );
                   return part_cnt;
         }
         part_index = 0;
 
         for(part_cnt = 0; part_cnt<MAX_PART_COUNT; part_cnt++)
                   part_secur[part_index] = 0;       
         //查找出所有的LINUX盘符
         for(part_cnt = 0; part_cnt < mbr->PartCount && part_cnt < MAX_PART_COUNT; part_cnt++)
         {
             //if((mbr->array[part_cnt].user_type == 2) || (mbr->array[part_cnt].user_type == 0))
             {
                            PRINT("The %d disk name = %s, class name = %s, disk size = %d\n", part_index, mbr->array[part_cnt].name,
                             mbr->array[part_cnt].classname, mbr->array[part_cnt].lenlo);
                            disk_array[part_index].offset = mbr->array[part_cnt].addrlo;
                            disk_array[part_index].size = mbr->array[part_cnt].lenlo;
                            part_secur[part_index] = mbr->array[part_cnt].user_type;
                            part_index ++;
             }
         }
         disk_array[part_index - 1].size = DiskSize - mbr->array[mbr->PartCount - 1].addrlo;
         _free_mbr();
         PRINT("The %d disk size = %lu\n", part_index - 1, disk_array[part_index - 1].size);
         PRINT("part total count = %d\n", part_index);
\#else
         part_index = 1;
         disk_array[0].offset = 0;
         disk_array[0].size = DiskSize;
         part_secur[0] = 0;
\#endif
         return part_index;
}
```
 

### Android4.0 蓝牙键盘不能用

在menuconfig中，打开HIDP protocol support选项为y

### 让多台机器公用一个下载帐号

将授权机器的密钥文件（id_rsa和id_rsa.pub）copy到新机器用户目录下面的.ssh文件夹里，如果没有就新建。
同时将相关文件的权限进行修改：
owner  和group 设置成新机器上的用户
id_rsa文件权限设置成600


### 高清播放器删除最后一张图片，图片再无法移动

改 Gallery2\src\com\android\gallery3d\ui\PhotoView.java 

```java
public void startSlideInAnimation(int direction) {
...
        mTransitionMode = direction; 改成  mTransitionMode = TRANS_NONE;

}
```

### 在系统中去掉蓝牙相关功能

现象：

小机不带蓝牙功能，但系统自带的widget上有蓝牙选项，该如何去掉？

解决方法：

在android4.0\device\softwinner\crane-common\tablet_core_hardware.xml中
注释掉 
```xml <feature name="android.hardware.bluetooth" /> ```

### 默认竖屏模式下，高清播放器休眠后自动播放视频

现象：

默认竖屏模式下，高清播放器休眠后自动播放视频

解决方法：

由于播放器，不支持竖屏模式，所以休眠之后，由于横竖屏切换，导致播放器再次运行 可以适当的延长播放器的暂停时间，以回避这问题。 packages\apps\Gallery2\src\com\android\gallery3d\app\MovieActivity.java

```java
import android.os.PowerManager; 
import android.content.Context; 
import android.os.SystemClock;


private PowerManager mPowerManager; 
@Override
public void onCreate(Bundle icicle) {
	mPowerManager = (PowerManager)getApplicationContext().getSystemService(Context.POWER_SERVICE); 
}

public void onPause() {  
	if(mPowerManager != null){ 
		if(!mPowerManager.isScreenOn()){ 
			SystemClock.sleep(1000);
		}
	}
	mControl.onPause();super.onPause();
}
```


### 字体为超大时，竖屏在解锁界面，解锁的图标显示不全

现象：

800x600,lcd_denisty 160

解决方法：

frameworks/base/core/res/res/layout-sw600dp/keyguard_screen_tab_unlock.xml

适当加大72行的android:layout_weight="1"

这样解锁部分的面积会增加，解决这个问题

其他分辨率需要修改其他layout文件夹下的keyguard_screen_tab_unlock.xml

### 音效里的Normal和Flat功能反

frameworks/base/media/libeffects/lvm/wrapper/Bundle/EffectBundle.h

gEqualizerPresets数组,名字可以随意修改

### 动态壁纸，线性光幕效果有黑块

效果：

将壁纸设为动态壁纸中的“线性光幕效果”时，在主界面旋转屏幕时未显示完整，即往边上拉存在黑块问题。

解决方法：

修改：
android\packages\wallpapers\Basic\src\com\android\wallpaper\nexus\NexusRS.java

```java
    @Override
    public void setOffset(float xOffset, float yOffset, int xPixels, int yPixels) {
        //mXOffset = xOffset; 
        //mScript.set_gXOffset(xOffset);
    }
```

### 修改Setting里面各字体对应的大小

现象：

感觉setting里面字体大小选择的变化幅度太大，希望小不要太小，特大不要太大，每级变化的幅度缩小点该如何修改

解决方法：

android4.0\packages\apps\Settings\res\values\arrays.xml 文件里

```xml
    <string-array name="entryvalues_font_size" translatable="false">        <item>0.85</item>        <item>1.0</item>        <item>1.15</item>        <item>1.30</item>    </string-array>    <string-array name="entryvalues_font_size" translatable="false">        <item>0.85</item>        <item>1.0</item>        <item>1.15</item>        <item>1.30</item>    </string-array>    <string-array name="entryvalues_font_size" translatable="false">        <item>0.85</item>        <item>1.0</item>        <item>1.15</item>        <item>1.30</item>    </string-array>    <string-array name="entryvalues_font_size" translatable="false">        <item>0.85</item>        <item>1.0</item>        <item>1.15</item>        <item>1.30</item>    </string-array>    <string-array name="entryvalues_font_size" translatable="false">        <item>0.85</item>        <item>1.0</item>        <item>1.15</item>        <item>1.30</item>    </string-array>    <string-array name="entryvalues_font_size" translatable="false">        <item>0.85</item>        <item>1.0</item>        <item>1.15</item>        <item>1.30</item>    </string-array>    <string-array name="entryvalues_font_size" translatable="false">        <item>0.85</item>        <item>1.0</item>        <item>1.15</item>        <item>1.30</item>    </string-array>    <string-array name="entryvalues_font_size" translatable="false">        <item>0.85</item>        <item>1.0</item>        <item>1.15</item>        <item>1.30</item>    </string-array>    <string-array name="entryvalues_font_size" translatable="false">        <item>0.85</item>        <item>1.0</item>        <item>1.15</item>        <item>1.30</item>    </string-array>    <string-array name="entryvalues_font_size" translatable="false">        <item>0.85</item>        <item>1.0</item>        <item>1.15</item>        <item>1.30</item>    </string-array>    <string-array name="entryvalues_font_size" translatable="false">        <item>0.85</item>        <item>1.0</item>        <item>1.15</item>        <item>1.30</item>    </string-array>    <string-array name="entryvalues_font_size" translatable="false">
        <item>0.85</item>
        <item>1.0</item>
        <item>1.15</item>
        <item>1.30</item>
    </string-array>
默认是0.15的梯度 改成合适的梯度如
    <string-array name="entryvalues_font_size" translatable="false">
        <item>0.90</item>
        <item>1.0</item>
        <item>1.10</item>
        <item>1.20</item>
    </string-array>
```

### 浏览器为Android模式时在线看视频卡

现象：

A10

1. 在浏览时有时偶尔会退出
2. 浏览器用android模式，看在线视频十几分钟左右后会卡，但用ipad模式不会卡

问题分析：

1. 需要具体分析问题，我们其它客户案上浏览正常
2. android模式采用的flash格式播放，而ipad采用的html5。flash由于自身格式和片源分割的关系，在线视频有机率产生不同步或卡顿问题。该问题由flash自身引起，这也是为何flash会逐渐被html5取代的一个方面。



### Android2.3上开启U盘自扫描

现象；

插入U盘后在播放器中看不到U盘文件，如何才能打开

解决方法：

需修改
frameworks\base\core\java\android\provider\Settings.java
frameworks\base\packages\SettingsProvider\src\com\android\providers\settings\DatabaseHelper.java
packages\providers\MediaProvider\src\com\android\providers\media\MediaScannerReceiver.java
packages\providers\MediaProvider\src\com\android\providers\media\MediaScannerService.java

具体修改如若无法完成，可参考相关补丁包----补丁呢!!!


### 如何降低CPU最高频率

现象：

目前系统为动态调频，如何限定最高CPU频率

解决方法：

降低CPU频率需要作以下两个修改：

1. 在lichee\tools\pack\chips\sun5i\configs\android\xxx\sys_config1.fex文件(“xxx”为对应的方案配置文件目录)中修改启动频率：
       [target]boot_clock =1008
    boot_clock为启动的CPU频率，单位为MHz，根据需要自行修改，该值不能大于最高频率；

2. 在lichee\linux-3.0\arch\arm\mach-sun4i\cpu-freq\cpu-freq.h文件中修改CPU的最高频率：
        #define SUN4I_CPUFREQ_MAX       (1008000000)
       SUN4I_CPUFREQ_MAX为CPU允许运行的最高频率，单位是Hz，根据需要自行修改；







### 修改sys_config.fex，不升级整个固件，就使其生效

1. 在android shell中将/dev/block/nanda mount到某个节点:
    mount -t vfat /dev/block/nanda /mnt/nand
2. 修改sys_config1后build固件，然后在lichee\tools\pack\out\bootfs下找到scrpt.bin和script0.bin
3. 然后用adb连接后，将scritp.bin和script0.bin推到所mount节点的根目录下，替换原有同名文件:
    adb push script*.bin  /mnt/nand/
4. 最后sync重启即可
    adb shell
    sync
    reboot

### 修改浏览器默认浏览模式

在android4.0.1/packages/apps/Browser/res/xml/debug_preferences.xml中将

```xml
<ListPreference
         android:key="user_agent"
         android:title="@string/pref_development_uastring"
         android:entries="@array/pref_development_ua_choices"
         android:entryValues="@array/pref_development_ua_values"
         android:defaultValue="3"/>
```

中修改defaultValue的值,对应如下:
Android  :0
Desktop :1
iPhone:2
iPad :3
Froyo-N1:4
Honeycomb-Xoom:5



### 电源管理不准以及低电不关机


现象：

v1.0和v1.1中感觉电池电量不准，在低电情况下，系统也没有自动关机

解决方法：

需确认sys_config1.fex以下几个地方：

1. pmu_battery_rdc的值为100
2. pmu_battery_cap为正确的电池电量
3. pmu_bat_para的放电曲线校正

对于低电系统没有自动关机问题，一般尝试修改pmu_battery_cap：
pmu_bat_para4            = 0
pmu_bat_para5            = 5

### 如何单独替换内核

现象：

有没有办法不升级整个固件，单独替换内核

解决方法：

```bash
在android shell中将/dev/block/nanda mount到某个节点:
mount -t vfat /dev/block/nanda /mnt/nand
然后用adb连接后，将bImage直接push到所mount节点的linux目录下，替换bImage:
adb push bImage /mnt/nand/linux
最后sync重启即可
adb shell
sync
reboot
```

### 摄像头Zoom功能生效

修改packages/apps/Camera/src/com/android/camera/ui/IndicatorControlWheel.java

dispatchTouchEvent函数，第269行

if ((mZoomControl != null) && (index == 0)) {   

改为

if ((mZoomControl != null) && (index == 0) && mCurrentLevel == 0) {

### 录象时不能选择分辨率

现象：

摄像头录像时，选择分辨率的选项没有

解决实例：

如果想在720p和480p之间选择，在mediaprofile.xml文件中，按如下设置：

```xml
<CamcorderProfiles>
        
        <EncoderProfile quality="720p" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="3000000"
                   width="1280"
                   height="720"
                   frameRate="30" />
            
            <Audio codec="aac"
                   bitRate="128000"
                   sampleRate="44100"
                   channels="2" />
        </EncoderProfile>
        
        <EncoderProfile quality="480p" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="1500000"
                   width="640"
                   height="480"
                   frameRate="25" />
            <Audio codec="aac"
                   bitRate="12200"
                   sampleRate="8000"
                   channels="1" />
        </EncoderProfile>
        <EncoderProfile quality="timelapse720p" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="3000000"
                   width="1280"
                   height="720"
                   frameRate="30" />
            <!-- audio setting is ignored -->
            <Audio codec="aac"
                   bitRate="128000"
                   sampleRate="44100"
                   channels="2" />
      </EncoderProfile>
      
      <EncoderProfile quality="timelapse480p" fileFormat="mp4" duration="30">
            <Video codec="h264"
                   bitRate="1500000"
                   width="640"
                   height="480"
                   frameRate="25" />
            <Audio codec="aac"
                   bitRate="12200"
                   sampleRate="8000"
                   channels="1" />
        </EncoderProfile> 

        <ImageEncoding quality="90" />
        <ImageEncoding quality="80" />
        <ImageEncoding quality="70" />
        <ImageDecoding memCap="20000000" />
        <Camera previewFrameRate="0" />
    </CamcorderProfiles>
```

### 修改蓝牙名称

如果你想修改默认的名字，可以这么做：
文件external/bluetooth/bluez/src/main.c
将main_opts.name  = g_strdup("BlueZ");里面的BlueZ换成你想要的名字即可！

### 为何用UsbManager调用getDeviceList获取设备列表总是空的

这个是因为缺少android.hardware.usb.host权限，

http://www.oschina.net/code/download_src?file=android-4.0.1%2Fdata%2Fetc%2Fandroid.hardware.usb.host.xml

下载该文件，放在/system/etc/permisson/下，可以解决

### 低电自动关机重启

现象：

低电状态下，自动关机后重启，进不了系统，偶尔能进入系统但是触屏没反应。

解决方法：

原因在于源不足以支持系统开机，而此时电池又电量太低，导致不能满足整个系统的开机功耗。解决方法是提高开机的门限电和Android关机门限。

1. boot阶段开机的门限电压设置方法 ：
在sys_config1脚本中
[boot_extend]
vol_threshold = 3500

2. 修改Android 低电关机的门限
```java 
BatteryService.java 
     private final void shutdownIfNoPower() {
        // shut down gracefully if our battery is critically low and we are not powered.
        // wait until the system has booted before attempting to display the shutdown dialog.
        if (mBatteryLevel < 5 && （5为5%即可关机修改 为更大的值，具体值可以根据具体情况而定）
```

### 修改鼠标的按键定义

现象：

鼠标插上以后，左键和右键的功能都是左键的，如何将右键修改为返回

解决方法：

这个是android的标准做法，如果定制，可以修改frameworks/base/services/input/inputreader.cpp文件中的：

```cpp
uint32_t CursorButtonAccumulator::getButtonState() const {
    uint32_t result = 0;
    if (mBtnLeft) {
        result |= AMOTION_EVENT_BUTTON_PRIMARY;
    }
    if (mBtnRight) {
        result |= AMOTION_EVENT_BUTTON_SECONDARY;
    }
    if (mBtnMiddle) {
        result |= AMOTION_EVENT_BUTTON_TERTIARY;
    }
    if (mBtnBack || mBtnSide) {
        result |= AMOTION_EVENT_BUTTON_BACK;
    }
    if (mBtnForward || mBtnExtra) {
        result |= AMOTION_EVENT_BUTTON_FORWARD;
    }
    return result;
}
```

目前因为下面函数

```cpp
static bool isPointerDown(int32_t buttonState) {
    return buttonState &
            (AMOTION_EVENT_BUTTON_PRIMARY | AMOTION_EVENT_BUTTON_SECONDARY
                    | AMOTION_EVENT_BUTTON_TERTIARY);
}
```
 
将鼠标左右键的功能设置成判断鼠标是否点击的操作，所以功能一致，如果需要修改，这需要改一下上面的getButtonState函数中的标志位即可，比如需要将右键改为后退键，只需要
函数中：

```cpp
  if (mBtnRight) {
        result |= AMOTION_EVENT_BUTTON_SECONDARY;
    }
//改为
  if (mBtnRight) {
        result |= AMOTION_EVENT_BUTTON_BACK;
    }
//即可
```

### 修改system分区大小

1. 修改sys_config.fex中的
```tex
    [partition3]
    class_name  = DISK
    name        = system
    size_hi     = 0
    size_lo     = 524288     //此处单位为K
    user_type   = 2
    ro            = 0
```

2. 修改device对应目录下的BoardConfig.mk中的
    BOARD_SYSTEMIMAGE_PARTITION_SIZE := 536870912  //此处单位为byte


