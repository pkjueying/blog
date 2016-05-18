---
title: Android 启动过程代码跟踪
comment: true
date: 2016-05-18 16:54:53
categories: [old blog]
tags: 
description: Android 启动过程代码跟踪,  勿喷......

---

> 准备工作：Android源码，UnderStand源码阅读工具



我们知道Android系统在启动时首先会启动Linux系统，引导加载Linux Kernel并启动init进程。Init进程是一个由内核启动的用户级进程，是Android系统的第一个进程。该进程的相关代码在android/system/core/init/init.c。在main函数中，有如下代码：

```cpp

   init_parse_config_file("/init.rc");

    action_for_each_trigger("early-init", action_add_queue_tail);

    ......

    /* execute all the boot actions to get us started */
    action_for_each_trigger("init", action_add_queue_tail);

    /* skip mounting filesystems in charger mode */
    if (!is_charger) {
        action_for_each_trigger("early-fs", action_add_queue_tail);
        action_for_each_trigger("fs", action_add_queue_tail);
        action_for_each_trigger("post-fs", action_add_queue_tail);
        action_for_each_trigger("post-fs-data", action_add_queue_tail);
    }

    .......

    if (is_charger) {
        action_for_each_trigger("charger", action_add_queue_tail);
    } else {
        action_for_each_trigger("early-boot", action_add_queue_tail);
        action_for_each_trigger("boot", action_add_queue_tail);
    }
```


这里会加载init.rc并进行解析，init.rc文件定义了在init进程中需要启动哪些进程服务和执行哪些动作。其详细说明参见android/system/core/init/reademe.txt。init.rc见如下定义：

......
service servicemanager /system/bin/servicemanager
    class core
    user system
    group system
    critical
    onrestart restart zygote
    onrestart restart media
    onrestart restart surfaceflinger
    onrestart restart drm

service vold /system/bin/vold
    class core
    socket vold stream 0660 root mount
    ioprio be 2

........

service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media
    onrestart restart netd

service drm /system/bin/drmserver
    class main
    user drm
    group drm system inet drmrpc sdcard_r

service media /system/bin/mediaserver
    class main

..........


具体解析过程见android/system/core/init/Init_parser.c。解析所得服务添加到service_list中，动作添加到action_list中。主要代码流程如下：
```cpp

for (;;) {
        switch (next_token(&state)) {
        case T_EOF:
            state.parse_line(&state, 0, 0);
            goto parser_done;
        case T_NEWLINE:
            state.line++;
            if (nargs) {
                int kw = lookup_keyword(args[0]);
                if (kw_is(kw, SECTION)) {
                    state.parse_line(&state, 0, 0);
                    parse_new_section(&state, kw, nargs, args);
                } else {
                    state.parse_line(&state, nargs, args);
                }
                nargs = 0;
            }
            break;
        case T_TEXT:
            if (nargs < INIT_PARSER_MAXARGS) {
                args[nargs++] = state.text;
            }
            break;
        }
    }

parser_done:
    list_for_each(node, &import_list) {
         struct import *import = node_to_item(node, struct import, list);
         int ret;

         INFO("importing '%s'", import->filename);
         ret = init_parse_config_file(import->filename);
         if (ret)
             ERROR("could not import file '%s' from '%s'\n",
                   import->filename, fn);
    }
    
```

接下来在main函数中执行动作和启动进程服务：
```cpp

for(; ;) {
......
execute_one_command();
        restart_processes();
......
}

```

通常init过程需要创建一些系统文件夹并启动USB守护进程、Android Debug Bridge守护进程、Debug守护进程、ServiceManager进程、Zygote进程等。

由init.rc对ServiceManager的描述service servicemanager /system/bin/servicemanager可知servicemanager进程从platform\frameworks\base\cmd\servicemanager\Service_manager.cpp启动。在main函数中有如下代码：

```cpp

int main(int argc, char **argv)
{
    struct binder_state *bs;
    void *svcmgr = BINDER_SERVICE_MANAGER;

    bs = binder_open(128*1024);

    if (binder_become_context_manager(bs)) {
        ALOGE("cannot become context manager (%s)\n", strerror(errno));
        return -1;
    }

    svcmgr_handle = svcmgr;
    binder_loop(bs, svcmgr_handler);
    return 0;
}

```

而在android/framework/base/cmd/servicemanager/Binder.c中
```cpp

int binder_become_context_manager(struct binder_state *bs)
{
    return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
}
```

以上首先调用binder_open()打开Binder设备(/dev/binder)，调用binder_become_context_manager()把当前进程设置为ServiceManager。ServiceManager本身就是一个服务。最后binder_loop()进入循环状态，并设置svcmgr_handler回调函数等待添加、查询、获取服务等请求。

在启动servicemanager的同时，再来启动Zygote，由init.rc对zygote的描述service zygot /system/bin/app_process可知zygote进程从Android\frameworks\base\cmds\app_process\App_main.cpp启动。这个文件的main（）方法，会调用Android_Runtime.cpp的文件中的start（）方法，这个方法通过JNI机制，来调用ZygoteInit.java孵化器初始文件，这个文件的Main（）函数，将会去调用所有进程。其主要代码如下:

```cpp

......
while (i < argc) {
        const char* arg = argv[i++];
        if (!parentDir) {
            parentDir = arg;
        } else if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = "zygote";
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName = arg + 12;
        } else {
            className = arg;
            break;
        }
    }
........
if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit",
                startSystemServer ? "start-system-server" : "");
    } else if (className) {
        // Remainder of args get passed to startup class main()
        runtime.mClassName = className;
        runtime.mArgC = argc - i;
        runtime.mArgV = argv + i;
        runtime.start("com.android.internal.os.RuntimeInit",
                application ? "application" : "tool");
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        return 10;
    }
    
```

其中runtime是AppRuntime类型，而AppRuntime继承至AndroidRuntime。我们继续跟踪 runtime.start函数：因在AppRuntime中没有对start的复写，我们到AppRuntime查看start的实现，路劲：android/framework/base/core/init
代码如下：注意runtime.start所传的参数

```cpp

void AndroidRuntime::start(const char* className, const char* options)
{

 if (strcmp(options, "start-system-server") == 0) {

 /* start the virtual machine */
 JNIEnv* env;
 if (startVm(&mJavaVM, &env) != 0) {
return;
 }
 onVmCreated(env);

    ...
    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    char* slashClassName = toSlashClassName(className);
    jclass startClass = env->FindClass(slashClassName);     //com.android.internal.os.ZygoteInit
    if (startClass == NULL) {
        ...
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);
	    ...
        }
    }
    ...
}

```

即先启动了虚拟机，然后利用JNI调用了zygoteInit。路劲：android/framework/base/core/java/com/android/internal/os在ZygoteInit的main中

```cpp
 
public static void main(String argv[]) {
        try {
            ...
            if (argv[1].equals("start-system-server")) {
                startSystemServer();
            } else if (!argv[1].equals("")) {
                throw new RuntimeException(argv[0] + USAGE_STRING);
            }
            ...
        } catch (MethodAndArgsCaller caller) {
            ...
        } 
}
```

我们继续跟踪startSystemServer() ， 在startSystemServer()中 ：

```cpp

private static boolean startSystemServer() throws 
	MethodAndArgsCaller, RuntimeException {
        String args[] = {
            ...
            "--runtime-init",
            "--nice-name=system_server",
            "com.android.server.SystemServer",
        };
       
        try {
            parsedArgs = new ZygoteConnection.Arguments(args);
            ...
            pid = Zygote.forkSystemServer(
                    ...
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }
        ...
        return true;
    }
```

 Zygote包装了Linux的fork。forkSystemServer()调用forkAndSpecialize()，最终穿过虚拟机调用android\dalvik\vm\native\dalvik_system_Zygote.c中Dalvik_dalvik_system_Zygote_forkAndSpecialize()。由dalvik完成fork新的进程。
 main()最后会调用runSelectLoopMode()，进入while循环，由peers创建新的进程。

我们跳转至com.android.server.SystemServer中,目录：android/framework/base/services/java/com/android/server 

```cpp
public static void main(String[] args) {
    	...
        init1(args);
}

```

在其main()函数中调用了init1(args)这个native函数，利用JNI机制，跟踪至frameworks/base/services/jni/com_android_server_systemService.cpp，然后到
frameworks/base/cmds/system_server/library/system_init.cpp在system_init()函数中有如下代码：
```cpp
    if (strcmp(propBuf, "1") == 0) {
        // Start the SurfaceFlinger
        SurfaceFlinger::instantiate();
    }
    AndroidRuntime* runtime = AndroidRuntime::getRuntime();
    ...
    LOGI("System server: starting Android services./n");
    runtime->callStatic("com/android/server/SystemServer", "init2");
    
```

  即完成了SurfaceFlinger的实例化，然后利用运行时的callStatic()函数调用了SystemServer的init2()函数.代码如下:
 
```cpp    

 public static final void init2() {
        Slog.i(TAG, "Entered the Android system server!");
        Thread thr = new ServerThread();
        thr.setName("android.server.ServerThread");
        thr.start();
 }
```

在这个ServerThread线程中，就可以看到我们熟悉的Android服务了：
```cpp

    @Override
    public void run() {
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_SYSTEM_RUN,
            SystemClock.uptimeMillis());

        Looper.prepareMainLooper();

        ......

        
        ContentService contentService = null;
        LightsService lights = null;
        PowerManagerService power = null;
        DynamicPManagerService dpm = null;
        DisplayManagerService display = null;
        BatteryService battery = null;
        VibratorService vibrator = null;
        AlarmManagerService alarm = null;
        ......

               
        .....

       	try {
                Slog.i(TAG, "Status Bar");
                statusBar = new StatusBarManagerService(context, wm);
                ServiceManager.addService(Context.STATUS_BAR_SERVICE, statusBar);
            } catch (Throwable e) {
                reportWtf("starting StatusBarManagerService", e);
            }

            try {
                Slog.i(TAG, "Clipboard Service");
                ServiceManager.addService(Context.CLIPBOARD_SERVICE,
                        new ClipboardService(context));
            } catch (Throwable e) {
                reportWtf("starting Clipboard Service", e);
            }

            try {
                Slog.i(TAG, "NetworkManagement Service");
                networkManagement = NetworkManagementService.create(context);
                ServiceManager.addService(Context.NETWORKMANAGEMENT_SERVICE, networkManagement);
            } catch (Throwable e) {
                reportWtf("starting NetworkManagement Service", e);
            }

       

        Looper.loop();
        Slog.d(TAG, "System ServerThread is exiting!");
    }
    
    
```

最后，调用各服务的systemReady()函数通知系统就绪,至此，系统的启动过程结束.






















