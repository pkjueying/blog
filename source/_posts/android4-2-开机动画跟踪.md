---
title: ' android4.2 开机动画跟踪'
comment: true
date: 2016-05-18 17:44:33
categories: [old blog]
tags:
description: Android系统开机动画简析.

---


> Android4.2 源码， UnderStand源码阅读



Android系统开机过程中主要会出现3个动画：

1. Linux系统启动（默认不出现）
2. Android平台启动初始化(默认出现“ANDROID的字样”)
3. ANDROID平台图形系统启动（默认出现含ANDROID字样的闪动动画图片）

下面我们一一来进行跟踪。

首先关于Linux的开机图画在/home2/zfl/a20-4.2/lichee_zfl/linux-3.3/drivers/video/logo/logo.c中。Linux kernel引导启动后，加载该图片。logo.c中定义了nologo,然后在linux_logo * __init_refok fb_find_logo(int depth)方法中根据nologo 来进行判断是否进行显示相关图画。具体相关代码如下：

```cpp

static bool nologo;
module_param(nologo, bool, 0);
MODULE_PARM_DESC(nologo, "Disables startup logo");

/* logo's are marked __initdata. Use __init_refok to tell
 * modpost that it is intended that this function uses data
 * marked __initdata.
 */
const struct linux_logo * __init_refok fb_find_logo(int depth)
{
        const struct linux_logo *logo = NULL;

        if (nologo)
                return NULL;

        if (depth >= 1) {
		\#ifdef CONFIG_LOGO_LINUX_MONO
                /* Generic Linux logo */
                logo = &logo_linux_mono;

		...
}
```

接下来进行开机“ANDROID”字样的跟踪：

ANDROID系统启动后，在/home2/zfl/a20-4.2/android/system/core/init/init.c 中有

```cpp

static int console_init_action(int nargs, char **args)
{
    int fd;
    char tmp[PROP_VALUE_MAX];

    if (console[0]) {
        snprintf(tmp, sizeof(tmp), "/dev/%s", console);
        console_name = strdup(tmp);
    }

    fd = open(console_name, O_RDWR);
    if (fd >= 0)
        have_console = 1;
    close(fd);

    //if( load_565rle_image(INIT_IMAGE_FILE) ) {
    if( load_argb8888_image(INIT_IMAGE_FILE) ) {
        fd = open("/dev/tty0", O_WRONLY);
        if (fd >= 0) {
            const char *msg;
                msg = "\n"
            "\n"
            "\n"
            "\n"
            "\n"
            "\n"
            "\n"  // console is 40 cols x 30 lines
            "\n"
            "\n"
            "\n"
            "\n"
            "\n"
            "\n"
            "\n"
            "             A N D R O I D ";
            write(fd, msg, strlen(msg));
            close(fd);
        }
    }
    return 0;
}
```

而在main()中有相关的调用：

```cpp

  /* skip mounting filesystems in charger mode */
    if (!is_charger) {
        action_for_each_trigger("early-fs", action_add_queue_tail);
        queue_builtin_action(console_init_action, "console_init");
        action_for_each_trigger("fs", action_add_queue_tail);
        action_for_each_trigger("post-fs", action_add_queue_tail);
        action_for_each_trigger("post-fs-data", action_add_queue_tail);
    }
```

上述中的INIT_IMAGE_FILE 在init.h中有相关定义：

```cpp

\#define INIT_IMAGE_FILE "/initlogo.rle"

int load_565rle_image( char *file_name );
int load_argb8888_image(char *fn);
```

接下来进行最后的开机动画即闪动的ANDROID字样的跟踪

在ANDROID4.2中，ANDROID的系统登录动画由前景和背景两站PNG图片组成，在我的系统中，这两张图片位于 /home2/zfl/a20-4.2/android/frameworks/base/core/res/assets/images  如图：

![](/img/animate/20130624115253140.png)


前景图片（android-logo-mask.png）中的ANDROID文字部分镂空，如图:

![](/img/animate/20130624115441921.png)

背景图片则是简单的纹理，如图：

![](/img/animate/20130624115540296.png)

前景图片在最上层显示，程序代码控制背景图片的连续滚动，透过前景图片文字镂空部分进行滚动进而显示纹理，从而显示动画效果。
其相关的代码在：

/home2/zfl/a20-4.2/android/frameworks/base/cmds/bootanimation/BootAnimation.cpp

/home2/zfl/a20-4.2/android/frameworks/base/include/androidfw/AssetManager.h

/home2/zfl/a20-4.2/android/frameworks/base/include/androidfw/Asset.h


.............................待

