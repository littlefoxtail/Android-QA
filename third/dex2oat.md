# dex2oat

## JIT编译器（Just in time）

动态编译的一种形式，用于提高程序的运行效率。在Android中，通常把JIT编译形式工作的编译器称为JIT编译器。
JIT编译器，安装诞生之初，其运行程序的核心组件依赖于Dalvik的运行环境，又称为Dalvik虚拟机，它的租用是用于运行程序的核心组件

## AndroidN的混合编译

在Dalvik虚拟机中，总是在运行时通过JIT（Just-In-Time）把字节码文件编译成机器码文件再执行，这样跑起来程序就很慢，所在ART，改为AOT（Ahead-Of-Time）提前编译，即在安装应用或OTA系统升级时提前把字节码编译成机器码，这样就可以直接执行，提高了运行效率。但是AOT有个缺点就是每次执行的时间都太长了，并且占用的ROM空间又很大

### Android N安装应用为什么这么快

谷歌在Android N中，JIT即时编译器再次回归，使之成为了一种JIT/AOT混合编译模式。
在Android N中，应用在安装时不再做编译，而是解释字节码。省去了冗长的编译时间，安装速度自然大大提升。

这个时候你可能就有新的疑问了，在应用安装时不做编译，那应用执行起来不是会变得很慢么？然而事实上是，Android N中的应用执行速度相比Android M并没有太大的差异，甚至使用一段时间后，Android N的应用执行速度比Android M还要快上不少。
这就要归功于新加入的这个JIT/AOT混合编译技术了。新增的JIT编译器用于对ART进行代码分析，使之可以在应用运行时，持续优化Android应用的性能。使得安装时不做编译，也能达到与安装时完整编译一样的效果。这种编译模式我们依旧同称为AOT，只不过它的含义不再是预编译，而是全时编译技术（All Of the Time compilation）。（应用安装和首次运行不做AOT编译，把运行中JIT解释执行的那部分代码收集起来，在手机空闲的时候通过dex2aot编译生成一份名为app image的base.art文件，然后在下次启动的时候一次性把app image加载进来到缓存，预先加载代替用时查找以提升应用的性能。）
此外，JIT的分析结果会被保存起来。当Android设备空闲或充电时，ART就会根据JIT的分析结果，将代码中的常用方法进行编译，而不常用的方法则待到需要时再编译，因而省下了部分存储空间。直观的体现就是我们安装完应用后，存储空间的占用变少了。

## dex加载

```java
// DexFile
private static native Object openDexFileNative(..);
```

```cpp
// DexFile_openDexFileNative
static jobject DexFile_openDexFileNative(JNIEnv* env,
                                         jclass,
                                         jstring javaSourceName,
                                         jstring javaOutputName ATTRIBUTE_UNUSED,
                                         jint flags ATTRIBUTE_UNUSED,
                                         jobject class_loader,
                                         jobjectArray dex_elements) {
  ScopedUtfChars sourceName(env, javaSourceName);
  if (sourceName.c_str() == nullptr) {
    return nullptr;
  }

  std::vector<std::string> error_msgs;
  const OatFile* oat_file = nullptr;
  std::vector<std::unique_ptr<const DexFile>> dex_files =
        //关键调用
      Runtime::Current()->GetOatFileManager().OpenDexFilesFromOat(sourceName.c_str(),
                                                                  class_loader,
                                                                  dex_elements,
                                                                  /*out*/ &oat_file,
                                                                  /*out*/ &error_msgs);
  return CreateCookieFromOatFileManagerResult(env, dex_files, oat_file, error_msgs);
}
```

## oat_file_manager.cc#OpenDexFilesFromOat

1. 检测是否已经有一个打开的oat文件
2. 如果没有已经打开的oat文件，则从磁盘上检测是否有一个已经生成的oat文件
3. 如果磁盘上有一个生成的oat文件，则检测该oat文件是否过期了以及是否包含了所有的dex文件
4. 如果以上都不满足，则会重新生成

### app image文件的加载

在app启动的时需要加载应用的oat文件以及可能存在的app image文件，大致流程：

1. 通过`OpenDexFilesFromOat`加载oat时，若app image存在，则通过调用`OpenImageSpace`函数加载；

    ```cpp
    // OpenDexFilesFromOat
    if (source_oat_file != nullptr) {
        bool added_image_space = false;
        if (source_oat_file->isExecutable()) {
            if (ShouldLoadAppImage(...)) {
                // apk启动时需要加载应用的oat文件以及可能存在的app image文件
                image_space = oat_file_assistant.openImageSpace(source_oat_file);
            }
        }
    }
    ```

2. 在加载app image文件时，通过`Update`函数，将art文件中的dexcache中dex的所有class插入到ClassTable，同时method更新到decache

    ```cpp
    // class_linker.cc#AddImageSpace
    if (app_image) {
        // 若app image 存在，则通过调用
        AppImageLoadingHelper::Update(this, space, class_loader, dex_caches, &temp_set);
    }
    ```

3. 类加载时，使用时ClassLinker::LookupClass会先从ClassTable中去查找，找不到时才会走到DefineClass中。

## dex2opt与dex2oat的区别

![dex2optanddex2oat](/img/dex2optanddex2oat.webp)

前者针对Dalvik虚拟机，后者针对Art虚拟机。ART之所以会比Dalvik快，是因为ART执行的是本地机器指令，而Dalvik执行的是Dex字节码，通过解释器执行。尽管Dalvik也会对频繁执行的代码进行JIT生成本地机器指令来执行，但毕竟在应用程序运行的过程中将Dex字节码翻译成本地机器机器指令也会影响到应用程序本身的执行，因此即使Dalvik使用了JIT，也在一定程度上也比不上直接就可以执行本地机器指令的运行时。

dex2opt是针对dex文件进行verification和optimization的操作，其对dex文件的优化结果变成了odex文件，这个文件和dex文件很像，只是使用了一些优化操作码（譬如优化调用虚拟指令等）
dex2oat是对dex文件的AOT提前编译，其需要一个dex文件，然后对其进行编译，结果是一个本地可执行的ELF文件，可以直接本地处理器执行。
除此之外还可以看到Dalvik虚拟机中有使用JIT编译器，也就是说其也能将程序运行的热点java字节码编译成本地code执行。所以其与Art虚拟机还是有区别的，Art虚拟机的dex2oat是提前编译所有dex字节码