# JDK_Version_init()

> **在《7. INitializeJVM 初始化JavaMV.md》第3节，调用了`JDK_Version_init()`函数。**

> **src/hotspot/share/runtime/java.cpp**
>
> > **其实是初始化 JDK_Version::_current 静态变量，最终\_current对应的实例对象字段信息如下：**
> >
> > **_major：12**
> >
> > **_thread_park_blocker：true**
> >
> > **_post_vm_init_hook_enabled：true**
> >
> > **其余的数值字段信息均为0**

```c++
void JDK_Version_init() {
  JDK_Version::initialize();
}

void JDK_Version::initialize() {
  jdk_version_info info;
  
  // _current为JDK_Version中定义的静态变量 static JDK_Version _current; 
  // 用来标识当前JDK_Version实例对象.
  // 由于_current是默认构造函数创建的, 此时它应该处于无效状态, 需要在初始化之后才会变为有效状态.
  assert(!_current.is_valid(), "Don't initialize twice");

  void *lib_handle = os::native_java_library();
  
  //查找 "JDK_GetVersionInfo0"函数指针
  jdk_version_info_fn_t func = CAST_TO_FN_PTR(jdk_version_info_fn_t,
     os::dll_lookup(lib_handle, "JDK_GetVersionInfo0"));

  //调用 "JDK_GetVersionInfo0()" 初始化info中的字段值
  (*func)(&info, sizeof(info));

  int major = JDK_VERSION_MAJOR(info.jdk_version);
  int minor = JDK_VERSION_MINOR(info.jdk_version);
  int security = JDK_VERSION_SECURITY(info.jdk_version);
  int build = JDK_VERSION_BUILD(info.jdk_version);

  // 本字段在JDK_GetVersionInfo0中设置为1
  if (info.pending_list_uses_discovered_field == 0) {
    vm_exit_during_initialization("Incompatible JDK is not using Reference.discovered field for pending list");
  }
  _current = JDK_Version(major, minor, security, info.patch_version, build,
                         info.thread_park_blocker == 1, //true
                         info.post_vm_init_hook_enabled == 1 /*true*/);
}
```



## 1. jdk_version_info 结构体

> **源文件：src/hotspot/share/include/jvm.h**

```c
typedef struct {
    unsigned int jdk_version; /* Encoded $VNUM as specified by JEP-223 */
    unsigned int patch_version : 8; /* JEP-223 patch version */
    unsigned int reserved3 : 8;
    unsigned int reserved1 : 16;
    unsigned int reserved2;

    /* The following bits represents new JDK supports that VM has dependency on.
     * VM implementation can use these bits to determine which JDK version
     * and support it has to maintain runtime compatibility.
     *
     * When a new bit is added in a minor or update release, make sure
     * the new bit is also added in the main/baseline.
     */
    unsigned int thread_park_blocker : 1;
    unsigned int post_vm_init_hook_enabled : 1;
    unsigned int pending_list_uses_discovered_field : 1;
    unsigned int : 29;
    unsigned int : 32;
    unsigned int : 32;
} jdk_version_info;
```



## 2. JDK_GetVersionInfo0()函数

> **源文件：src/java.base/share/native/libjava/jdk_util.c**
>
> > 对于本机openjdk12源码，相关的属性值如下：
> >
> > VERSION_FEATURE：12
> >
> > VERSION_INTERIM：0
> >
> > VERSION_UPDATE：0
> >
> > VERSION_PATCH：0
> >
> > VERSION_BUILD：0

```c
JNIEXPORT void JDK_GetVersionInfo0(jdk_version_info* info, size_t info_size) {
    /* These VERSION_* macros are given by the build system */
    const unsigned int version_major = VERSION_FEATURE;
    const unsigned int version_minor = VERSION_INTERIM;
    const unsigned int version_security = VERSION_UPDATE;
    const unsigned int version_patch = VERSION_PATCH;
    const unsigned int version_build = VERSION_BUILD;

    memset(info, 0, info_size);
    info->jdk_version = ((version_major & 0xFF) << 24) |
                        ((version_minor & 0xFF) << 16) |
                        ((version_security & 0xFF) << 8)  |
                        (version_build & 0xFF);
    info->patch_version = version_patch;
    info->thread_park_blocker = 1;
    // Advertise presence of PostVMInitHook:
    // future optimization: detect if this is enabled.
    info->post_vm_init_hook_enabled = 1;
    info->pending_list_uses_discovered_field = 1;
}
```



### 2.1 属性值来源

> **在`build/macosx-x86_64-server-fastdebug/spec.gmk`文件中包含如下几行内容：**
>
> ```properties
> ## Building blocks of the version string
> # First three version numbers, with well-specified meanings (numerical)
> VERSION_FEATURE := 12
> VERSION_INTERIM := 0
> VERSION_UPDATE := 0
> VERSION_PATCH := 0
> VERSION_EXTRA1 := 0
> VERSION_EXTRA2 := 0
> VERSION_EXTRA3 := 0
> # The pre-release identifier (string)
> VERSION_PRE := internal
> # The build number (numerical)
> VERSION_BUILD := 0
> # Optional build information (string)
> VERSION_OPT := adhoc.gwwwwt.jdk12
> ```

