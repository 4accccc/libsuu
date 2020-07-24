# libsu

[![](https://jitpack.io/v/topjohnwu/libsu.svg)](https://jitpack.io/#topjohnwu/libsu)

[Full Javadoc](https://javadoc.jitpack.io/com/github/topjohnwu/libsu/docs/2.5.2/javadoc/overview-summary.html)

An Android library that provides APIs to a Unix (root) shell.

Some poorly coded applications requests a new shell (call `su`, or worse `su -c <commands>`) for every single command, which is very inefficient. This library makes sharing a single, globally shared shell session in Android applications super easy: developers won't have to bother about concurrency issues, and with a rich selection of both synchronous and asynchronous APIs, it is much easier to create a powerful root app.

Optionally, `libsu` comes with a full suite of I/O classes, re-creating `java.io` classes but enhanced with root access. Without even thinking about command-lines, you can use `File`, `RandomAccessFile`, `FileInputStream`, and `FileOutputStream` equivalents on files that are only accessible with root permissions. The I/O stream classes are carefully optimized and have very promising performance.

Also optionally, this library bundles with prebuilt BusyBox binaries. App developers can easily setup and create an internal BusyBox environment without relying on potentially flawed (or even no) external BusyBox.

One complex Android application using `libsu` for all root related operations is [Magisk Manager](https://github.com/topjohnwu/Magisk/tree/master/app).

## [Changelog](./CHANGELOG.md)

## [Javadoc](https://javadoc.jitpack.io/com/github/topjohnwu/libsu/docs/2.5.2/javadoc/overview-summary.html)

## Download
```groovy
android {
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
}
repositories {
    maven { url 'https://jitpack.io' }
}
dependencies {
    def libsuVersion = '2.6.0'
    implementation "com.github.topjohnwu.libsu:core:${libsuVersion}"

    /* Optional: For using com.topjohnwu.superuser.io classes */
    implementation "com.github.topjohnwu.libsu:io:${libsuVersion}"

    /* Optional: To bundle prebuilt BusyBox binaries */
    implementation "com.github.topjohnwu.libsu:busybox:${libsuVersion}"
}
```

## Quick Tutorial

### Configurations
Set configurations in your MainActivity or Application class:

```java
static {
    /* Shell.Config methods shall be called before any shell is created
     * This is the why in this example we call it in a static block
     * The followings are some examples, check Javadoc for more details */
    Shell.Config.setFlags(Shell.FLAG_REDIRECT_STDERR);
    Shell.Config.verboseLogging(BuildConfig.DEBUG);
    Shell.Config.setTimeout(10);
}
```

### Shell Operations

```java
// Run commands and get output immediately
List<String> output = Shell.su("find /dev/block -iname boot").exec().getOut();

// Aside from commands, you can also load scripts from InputStream
Shell.Result result = Shell.su(getResources().openRawResource(R.raw.script)).exec();

// You can get more stuffs from the results
int code = result.getCode();
boolean ok = result.isSuccess();
output = result.getOut();

// Run commands and output to a specific List
List<String> logs = new ArrayList<>();
Shell.su("cat /cache/magisk.log").to(logs).exec();

// Run commands in the background and don't care results
Shell.su("setenforce 0").submit();

// Run commands in the background and get results via a callback
Shell.su("sleep 5", "echo hello").submit(result -> {
    /* This callback will be called on the main (UI) thread
     * after the operation is done (5 seconds after submit) */
    result.getOut();  /* Should return a list with a single string "hello" */
})

// Create a reactive callback List, and update the UI on each line of output
List<String> callbackList = new CallbackList<String>() {
    @MainThread
    @Override
    public void onAddElement(String s) {
        /* This callback will be called on the main (UI) thread each time
         * the list adds a new element (in this case: shell outputs a new line)*/
        uiUpdate(s);  /* Some method to update the UI */
    }
};
Shell.su(
    "for i in 1 2 3 4 5;do",
    "  echo $i"
    "  sleep 1"
    "done",
    "echo 'countdown done!'").to(callbackList).submit(result -> {
        /* Some stuffs cannot be acquired from callback lists
         * e.g. return codes */
        uiUpdate(result.getCode());
    });

// Also get STDERR
List<String> stdout = new ArrayList<>();
List<String> stderr = new ArrayList<>();
Shell.su("echo hello", "echo hello >&2").to(stdout, stderr).exec();
```

### I/O
Add `com.github.topjohnwu.libsu:io` as dependency to access the I/O wrapper classes:

```java
/* Treat files that require root access just like ordinary files */
File logs = SuFile.open("/cache/magisk.log");
if (logs.exists()) {
    try (InputStream in = new SuFileInputStream(logs);
         OutputStream out = new SuFileOutputStream("/data/magisk.log.bak")) {
        /* All file data can be accessed with Java Streams */
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

### Advanced
Initialize shells with custom `Shell.Initializer`, similar to what `.bashrc` will do:

```java
class ExampleInitializer extends Shell.Initializer {
    @Override
    public boolean onInit(Context context, Shell shell) {
        try (InputStream bashrc = context.getResources().openRawResource(R.raw.bashrc)) {
            // Load a script from raw resources
            shell.newJob()
                .add(bashrc)                            /* Load a script from resources */
                .add("export ENVIRON_VAR=SOME_VALUE")   /* Run some commands */
                .exec();
        }
        return true;
    }
}

// Register the class as an initializer
Shell.Config.setInitializers(ExampleInitializer.class);
```

### BusyBox
The I/O classes relies on several commandline tools. *Most* of the tools are available in modern Android via `toybox` (Android 6+), however for compatibility and reliable/reproducible behavior (some applets included in `toybox` is not fully featured), it is a good idea to have BusyBox bundled with your app.

The BusyBox binaries are pretty large in size (1.3 - 2.1 MB for each ABI). To reduce APK size, the best option is to use either [App Bundles](https://developer.android.com/guide/app-bundle) or [Split APKs](https://developer.android.com/studio/build/configure-apk-splits). If you are not publishing to Play Store, you can also limit the supported ABIs:

```groovy
android {
  defaultConfig {
    ndk {
      // Filter your supported ABIs
      abiFilters 'armeabi-v7a'
    }
  }
}
```

To setup BusyBox, set `BusyBoxInstaller` as the first shell initializer:

```java
/* Add com.github.topjohnwu.libsu:busybox as a dependency, and
 * register BusyBoxInstaller as the first initializer. */
Shell.Config.setInitializers(BusyBoxInstaller.class, /* other initializers */);
```

## Example

This repo also comes with an example app (`:example`), check the code and play/experiment with it.
