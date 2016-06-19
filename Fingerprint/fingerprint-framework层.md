# Fingerprint -- framework层#


Android Fingerprint模块framework层包含三个部分：FingerprintManager、FingerprintService和Fingerprintd。他们之间的关系如下图：
![](http://i.imgur.com/3jvB31m.png)


### 一、各模块简要说明 ###
----------


- FingerprintManager可以说是应用层的代理模块。每一个需要用到fingerprint的app都是通过调用FingerprintManager的接口实现相关功能的。它向上提供了preEnroll、enroll、postEnroll、authenticate、getAuthenticatorId等接口，向下注册了回调函数。底层将注册结果，认证结果回调通知到FingerprintManager。

- FingerprintService管理整个注册，识别、删除指纹、检查使用权限流程的逻辑。

- Fingerprintd 初始化Hal层的fingerprint模块，为FingerprintService提供操作指纹的接口。向Hal层注册消息回调函数。向KeystoreService添加获取到的auth_token。


### 二、Fingerprint framework层初始化流程浅析 ###
----------

在应用注册指纹，使用指纹识别之前。Fingerprint要先完成相应的初始化工作。主要是对Hal层的初始化。我们看一下具体流程。

1、在android6.0中FingerprintService会先添加把自己添加到ServiceManager。在 SystemServiceRegistry.java中会从ServiceManager中取到FingerprintService的binder对象，然后构造出FingerprintManager，并添加到ServiceManager中等待app获取。

       registerService(Context.FINGERPRINT_SERVICE, FingerprintManager.class,
            new CachedServiceFetcher<FingerprintManager>() {
        @Override
        public FingerprintManager createService(ContextImpl ctx) {
            IBinder binder = ServiceManager.getService(Context.FINGERPRINT_SERVICE);
            IFingerprintService service = IFingerprintService.Stub.asInterface(binder);
            return new FingerprintManager(ctx.getOuterContext(), service);
        }});

2、看一下FingerprintManager的构造函数。很简单就是保存好FingerprintService的binder对象待后续使用，然后构造了消息回调用的handler。
		

		public FingerprintManager(Context context, IFingerprintService service) {
	        mContext = context;
	        mService = service;
	        if (mService == null) {
	            Slog.v(TAG, "FingerprintManagerService was null");
	        }
	        mHandler = new MyHandler(context);
	    }

3、重点在FingerprintService.java，在其onStart方法中先new FingerprintServiceWrapper，（FingerprintServiceWrapper按照IFingerprintService.aidl实现了authenticate、cancelAuthentication、enroll等相应的服务端接口），再将其添加到servicemanager管理起来。然后调getFingerprintDaemon()和listenForUserSwitches()。

    @Override
    public void onStart() {
        publishBinderService(Context.FINGERPRINT_SERVICE, 
			new FingerprintServiceWrapper());
        IFingerprintDaemon daemon = getFingerprintDaemon();
        if (DEBUG) Slog.v(TAG, "Fingerprint HAL id: " + mHalDeviceId);
        listenForUserSwitches();
    }

细看getFingerprintDaemon()，主要是获取fingerprintd，并调用openhal初始化指纹的HAL层。这个地方简单的一句，在hal层却有一大堆初始化的工作要做。

    public IFingerprintDaemon getFingerprintDaemon() {
        if (mDaemon == null) {
			//获取fingerprintd
            mDaemon = IFingerprintDaemon.Stub.asInterface(
					ServiceManager.getService(FINGERPRINTD));
            if (mDaemon != null) {
                try {
                    mDaemon.asBinder().linkToDeath(this, 0);
					//向fingerprintd注册回调函数mDaemonCallback
                    mDaemon.init(mDaemonCallback);

					//调用获取fingerprintd的openhal函数
                    mHalDeviceId = mDaemon.openHal();
                    if (mHalDeviceId != 0) {

						/*建立fingerprint文件系统节点，设置节点访问权限，
						调用fingerprintd的setActiveGroup，
						将路径传下去。此路径一半用来存储指纹模板的图片等*/
                        updateActiveGroup(ActivityManager.getCurrentUser());
                    } else {
                        Slog.w(TAG, "Failed to open Fingerprint HAL!");
                        mDaemon = null;
                    }
                } catch (RemoteException e) {
                    Slog.e(TAG, "Failed to open fingeprintd HAL", e);
                    mDaemon = null; // try again later!
                }
            } else {
                Slog.w(TAG, "fingerprint service not available");
            }
        }
        return mDaemon;
    }

4、fingerprintd在init.rc有相应的开机启动脚本，所以一开机就会跑它的main函数。mian函数很简单就是将fingerprintd添加到servicemanager中管理。然后开启线程，等待binder调用。我们重点看一下它的openHal（）函数。

    int64_t FingerprintDaemonProxy::openHal() {
    ALOG(LOG_VERBOSE, LOG_TAG, "nativeOpenHal()\n");
    int err;
    const hw_module_t *hw_module = NULL;

	//根据名称获取指纹hal层模块。hw_module这个一般由指纹芯片厂商根据 fingerprint.h实现
    if (0 != (err = hw_get_module(FINGERPRINT_HARDWARE_MODULE_ID, &hw_module))) {
        ALOGE("Can't open fingerprint HW Module, error: %d", err);
        return 0;
    }
    if (NULL == hw_module) {
        ALOGE("No valid fingerprint module");
        return 0;
    }

    mModule = reinterpret_cast<const fingerprint_module_t*>(hw_module);
	
    if (mModule->common.methods->open == NULL) {
        ALOGE("No valid open method");
        return 0;
    }

    hw_device_t *device = NULL;
	//大头在这里，调用fingerprint_module_t的open函数
    if (0 != (err = mModule->common.methods->open(hw_module, NULL, &device))) {
        ALOGE("Can't open fingerprint methods, error: %d", err);
        return 0;
    }

    if (kVersion != device->version) {
        ALOGE("Wrong fp version. Expected %d, got %d", kVersion, device->version);
        // return 0; // FIXME
    }

    mDevice = reinterpret_cast<fingerprint_device_t*>(device);

	//向hal层注册消息回调函数，主要回调 注册指纹进度，识别结果，错误信息等等
    err = mDevice->set_notify(mDevice, hal_notify_callback);
    if (err < 0) {
        ALOGE("Failed in call to set_notify(), err=%d", err);
        return 0;
    }

    // Sanity check - remove
    if (mDevice->notify != hal_notify_callback) {
        ALOGE("NOTIFY not set properly: %p != %p", mDevice->notify, hal_notify_callback);
    }

    ALOG(LOG_VERBOSE, LOG_TAG, "fingerprint HAL successfully initialized");
    return reinterpret_cast<int64_t>(mDevice); // This is just a handle
	}

### 三、总结 ###
----------
Fingerprint framework层并没有太多实质性的东西，它只是一个壳，封装好了指纹hal层模块，让上次用户更简单的使用指纹接口。hal层才是指纹模块的核心。注册，识别流程和hal层的东西我会在之后的文章中分享。
