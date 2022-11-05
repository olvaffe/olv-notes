Android Runtime
===============

## Using NDK for JNI

- The Java class should have, for example,

    package com.example.hellojni;

    ...;

    public class HelloJni extends Activity {
        ....;

        public native String  stringFromJNI();

        static {
            System.loadLibrary("hello-jni");
        }
    }
- `libhello-jni.so` defines `Java_com_example_hellojni_HelloJni_stringFromJNI`

## EGL Java binding

- `com.google.android.gles_jni.EGLImpl` has
   - `native private static void _nativeClassInit();`
   - `static { _nativeClassInit(); }`
- `com_google_android_gles_jni_EGLImpl.cpp` has
  - `register_com_google_android_gles_jni_EGLImpl` that calls
    `android::AndroidRuntime::registerNativeMethods` to register native methods
  - `nativeClassInit` C function is registered as `_nativeClassInit`
- In `AndroidRuntime::start`, `startReg` is called to register all native
  methods to the VM.  This is when
  `register_com_google_android_gles_jni_EGLImpl` is called

## Dalvik JNI

- `dvmLoadNativeCode` dlopen()s a library
  - it is called when the Java code does `System.loadLibrary`.  The class is
    defined by JDK and it calls into the VM's `nativeLoad`
- `dvmResolveNativeMethod` resolves a native method
  - it looks up the internal native methods first
  - if no hit, it looks up in the shared libraries using
    - `createJniNameString` mangles the method to
      `Java/<package>/<class>/<method>`
    - `mangleString` mangles the above string to
      `Java_<package>_<class>_<method>`
- Internal native methods have type `DalvikBridgeFunc`.  It is set to
  `method->nativeFunc` and can be called directly
- External native methods must be called through a JNI bridge.
  `method->nativeFunc` is set to the bridge while the actual function is set to
  `method->insns`.  The bridge will call `dvmPlatformInvoke` to execute the
  actual function.
