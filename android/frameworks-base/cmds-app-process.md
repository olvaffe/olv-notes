# `app_process`

## Overview

- `app_process` is a C++ program that runs a Java program under ART
- `app_process <art-options> <parent-dir> <native-options> [<class-name>] <java-options>`
  - `<art-options>` are forwarded to ART
    - they are parsed by `art/runtime/parsed_options.cc`
  - `<parent-dir>` is ignored
  - `<native-options>` are handled by `app_process`
    - `--zygote`
    - `--start-system-server`
    - `--application`
    - `--nice-name`
  - `<java-options>` are forwarded to Java program
- `app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote`
  - `-Xzygote` is forwarded to ART
  - `/system/bin` is ignored
  - `--zygote` is translated to `com.android.internal.os.ZygoteInit`
    - the java entrypoint is the `main` method in
      `frameworks/base/core/java/com/android/internal/os/ZygoteInit.java`
  - `--start-system-server` is translated to `start-system-server` and
    forwarded to java
  - `--socket-name=zygote` is forwarded to java
- `CLASSPATH=/system/framework/svc.jar app_process /system/bin com.android.commands.svc.Svc ...`
  - `CLASSPATH` is parsed by `art/runtime/parsed_options.cc`
  - the java entrypoint is the `main` method of
    `com.android.internal.os.RuntimeInit`
    - `nativeFinishInit` calls `app_process`'s `AppRuntime::onStarted`
      - `ar->callMain` calls the `main` method of
        `com.android.commands.svc.Svc`
