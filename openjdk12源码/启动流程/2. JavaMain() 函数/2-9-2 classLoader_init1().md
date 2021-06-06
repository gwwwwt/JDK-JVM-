

# classLoader_init1() 函数

> **类加载器初始化，`其实就是初始化了ClassLoader中的几个静态成员变量，使它们指向相应的函数指针。`个人前面以为setup_bootstrap_search_path()方法调用中可能有一些重要的逻辑，但根据本机调试过程，基本也没有啥用，无语...**
>
> **源码: classLoader.cpp**

## 1. 相关说明

### 1.1 Arguments::\_sun_boot_library_path 和 Arguments::\_system_boot_class_path 成员变量什么时候初始化的

> **关于 Arguments:: 的初始化，参考《2. JavaMain函数/2-0 Threads::create_vm() - 创建VM.md》第2.1节**
>
> **在本机上，_sun_boot_library_path 路径为`/Users/gwwwwt/jvm/jdk12/build/macosx-x86_64-server-fastdebug/jdk/lib`**
>
> **而_system_boot_class_path 路径为`/Users/gwwwwt/jvm/jdk12/build/macosx-x86_64-server-fastdebug/jdk/modules/java.base`**



### 1.2 os::dll_locate_lib() 和 os::dll_load() 方法

> **classLoader初始化时还需要加载其它的库，dll_locate_lib()用来定位库，dll_load()则用来加载库**
>
> **源码: os.cpp**

#### 1.2.1 os::dll_locate_lib() 定位动态库位置

```c++
/*
参数 pname: 要在哪个目录下搜索动态库
参数 fname: 动态库的中间名, dll_locate_lib中会拼接成 lib$(fname).dylib 格式的动态库名进行定位
参数 buffer: 在定位到动态库后, 保存该动态库路径
*/
bool os::dll_locate_lib(char *buffer, size_t buflen, const char* pname, const char* fname) {
		bool retval = false;

    // <1> 拼接要查找的库的名字; 对于MacOS系统, JNI_LIB_PREFIX = "lib"; JNI_LIB_SUFFIX = ".dylib"
    // 另外需要参数 fname 来标识共享库的中间的字符串, 从而拼接成 lib$(fname).dylib 形式的字符串, 即库名
  	size_t fullfnamelen = strlen(JNI_LIB_PREFIX) + strlen(fname) + strlen(JNI_LIB_SUFFIX);
  	char* fullfname = (char*)NEW_C_HEAP_ARRAY(char, fullfnamelen + 1, mtInternal);
  
  	if (dll_build_name(fullfname, fullfnamelen + 1, fname)) {
      
    		const size_t pnamelen = pname ? strlen(pname) : 0; // 检查目录路径参数是否为空

    		if (pnamelen == 0) {
            //<2>: pname参数为NULL的情况
      			const char* p = get_current_directory(buffer, buflen); //如果pname为空, 则使用当前工作目录来查找库
      			if (p != NULL) {
        				const size_t plen = strlen(buffer);
        				const char lastchar = buffer[plen - 1];
                // pname为空时, buffer中结果为: ${当前工作目录}/lib${fname}.dylib; 
                // 当然还需要检查这个库文件是否存在.
        				retval = conc_path_file_and_check(buffer, &buffer[plen], buflen - plen,
                                          "", lastchar, fullfname);
      			}
    		} else if (strchr(pname, *os::path_separator()) != NULL) {
      			//<3>: pname非NULL且pname字符串中包含文件路径分隔符(如'/')
      			int n;
      			char** pelements = split_path(pname, &n); //分割pname字符串
      			if (pelements != NULL) {
        				for (int i = 0; i < n; i++) {
          					char* path = pelements[i];
          					size_t plen = (path == NULL) ? 0 : strlen(path);
          					if (plen == 0) {
            						continue; // Skip the empty path values.
          					}
          					const char lastchar = path[plen - 1];
          					retval = conc_path_file_and_check(buffer, buffer, buflen, path, lastchar, fullfname);
          					if (retval) break;
        				}	
        				
        				for (int i = 0; i < n; i++) {  // Release the storage allocated by split_path.
          					if (pelements[i] != NULL) {
            						FREE_C_HEAP_ARRAY(char, pelements[i]);
          					}
        				}
        
            		FREE_C_HEAP_ARRAY(char*, pelements);
      			}
    	} else {
      		// <4>: pname非NULL且不包含'/'路径分隔符
      		const char lastchar = pname[pnamelen-1];
      		retval = conc_path_file_and_check(buffer, buffer, buflen, pname, lastchar, fullfname);
    	}
  	}

  	FREE_C_HEAP_ARRAY(char*, fullfname);
  	return retval;
}

```



#### 1.2.2 os::dll_load() 加载动态库

> **源码: os_bsd.cpp**

```c++
// 注: 这是在MacOS上的实现, 在Linux上是用的不同实现
void * os::dll_load(const char *filename, char *ebuf, int ebuflen) {
  void * result= ::dlopen(filename, RTLD_LAZY); // 核心就是使用dlopen系统调用加载动态库
  if (result != NULL) {
    return result;
  }

  ::strncpy(ebuf, ::dlerror(), ebuflen-1);
  ebuf[ebuflen-1]='\0';

  return NULL;
}
```



### 1.3 ClassPathEntry及其子类

```c++
// class path下的库的表示类;
class ClassPathEntry : public CHeapObj<mtClass> {
		private:
  			ClassPathEntry* volatile _next; //唯一的字段, 用于构建单向链表
		public:
        // 定义了纯虚函数, 所以不能直接实例化ClassPathEntry对象
  			virtual bool is_modules_image() const = 0;
  			virtual bool is_jar_file() const = 0;
  			virtual const char* name() const = 0;
  			virtual JImageFile* jimage() const = 0;
  			virtual void close_jimage() = 0;
};

// 子类1, 表示目录
class ClassPathDirEntry: public ClassPathEntry {
  const char* _dir;   //目录名 
};

// 子类2, 表示jar包类型库
class ClassPathZipEntry: public ClassPathEntry {
 		jzfile* _zip;              // 标识jar包, 即jar包文件指针
  	const char*   _zip_name;   // jar包名
};

// 子类3, java image files
class ClassPathImageEntry: public ClassPathEntry {
  	JImageFile* _jimage; //image file文件指针
  	const char* _name;   //image file名
};
```



## 2. classLoader_init1() 函数

```c++
void classLoader_init1() { //实际调用了 ClassLoader::initialize() 方法
  	ClassLoader::initialize();
}

void ClassLoader::initialize() {
  
  load_zip_library(); //<1> 初始化ClassLoader类中的与zip/jar包处理的相关函数指针成员变量, 参考2.1节
  
  load_jimage_library(); //<2> 初始化与image相关的函数指针, 参考第2.2节

  setup_bootstrap_search_path(); //<3> 初始化bootstrap path, 参考第2.3节
}
```



### 2.1 ClassLoader::load_zip_library() 

> **本方法主要加载了与处理zip/jar文件相关的函数指针，并赋值给了ClassLoader中的相关静态成员变量。当然由于这些函数可能定义在不同的运行库中，所以又需要调用1.2中的相关函数先加载运行库，再查找对应的函数指针**
>
> > **初始化相关成员变量: **
> >
> > ```c++
> > //classLoader.cpp
> > static ZipOpen_t         ZipOpen            = NULL;
> > static ZipClose_t        ZipClose           = NULL;
> > static FindEntry_t       FindEntry          = NULL;
> > static ReadEntry_t       ReadEntry          = NULL;
> > static GetNextEntry_t    GetNextEntry       = NULL;
> > static canonicalize_fn_t CanonicalizeEntry  = NULL;
> > static ZipInflateFully_t ZipInflateFully    = NULL;
> > static Crc32_t           Crc32              = NULL;
> > ```

```c++
// Arguments::get_dll_dir() 方法返回的就是 Arguments::_sun_boot_library_path
void ClassLoader::load_zip_library() {
    //<1>: 使用1.2节的搜索/加载的函数, 保证加载了 libjava.dylib 运行库
  	os::native_java_library(); 
  
  	char path[JVM_MAXPATHLEN];
  	char ebuf[1024];
  	void* handle = NULL;
    //<2>: 加载 libzip.dylib 运行库
  	if (os::dll_locate_lib(path, sizeof(path), Arguments::get_dll_dir(), "zip")) {
    		handle = os::dll_load(path, ebuf, sizeof ebuf);
  	}

	  //<3>: 从libzip.dylib库中查找相关的函数指针
 	 	ZipOpen      = CAST_TO_FN_PTR(ZipOpen_t, os::dll_lookup(handle, "ZIP_Open"));
  	ZipClose     = CAST_TO_FN_PTR(ZipClose_t, os::dll_lookup(handle, "ZIP_Close"));
  	FindEntry    = CAST_TO_FN_PTR(FindEntry_t, os::dll_lookup(handle, "ZIP_FindEntry"));
  	ReadEntry    = CAST_TO_FN_PTR(ReadEntry_t, os::dll_lookup(handle, "ZIP_ReadEntry"));
  	GetNextEntry = CAST_TO_FN_PTR(GetNextEntry_t, os::dll_lookup(handle, "ZIP_GetNextEntry"));
  	ZipInflateFully = CAST_TO_FN_PTR(ZipInflateFully_t, os::dll_lookup(handle, "ZIP_InflateFully"));
  	Crc32        = CAST_TO_FN_PTR(Crc32_t, os::dll_lookup(handle, "ZIP_CRC32"));

  	//<4>: 从libjava.dylib库中查找 Canonicalize() 函数指针
  	void *javalib_handle = os::native_java_library();
  	CanonicalizeEntry = CAST_TO_FN_PTR(canonicalize_fn_t, os::dll_lookup(javalib_handle, "Canonicalize"));
}
```



### 2.2 ClassLoader::load_jimage_library()

> **与2.1节类似，也是初始化ClassLoader中的一些静态成员变量，但这里是image相关的函数指针**
>
> **`这里image好像跟module相关，所以后面可能会略过它了`**
>
> > **初始化的静态成员变量:**
> >
> > ```c++
> > static JImageOpen_t                    JImageOpen             = NULL;
> > static JImageClose_t                   JImageClose            = NULL;
> > static JImagePackageToModule_t         JImagePackageToModule  = NULL;
> > static JImageFindResource_t            JImageFindResource     = NULL;
> > static JImageGetResource_t             JImageGetResource      = NULL;
> > static JImageResourceIterator_t        JImageResourceIterator = NULL;
> > static JImage_ResourcePath_t           JImageResourcePath     = NULL;
> > ```

```c++
void ClassLoader::load_jimage_library() {
  	os::native_java_library(); //<1>: 确保加载了libjava.dylib
  	
  	char path[JVM_MAXPATHLEN];
  	char ebuf[1024];
  	void* handle = NULL;
    // <2>: 加载libjimage.dylib库
  	if (os::dll_locate_lib(path, sizeof(path), Arguments::get_dll_dir(), "jimage")) {
    		handle = os::dll_load(path, ebuf, sizeof ebuf);
  	}
  	
  	// <3>: 查找相关函数指针并赋值给ClassLoader的对应成员变量
  	JImageOpen = CAST_TO_FN_PTR(JImageOpen_t, os::dll_lookup(handle, "JIMAGE_Open"));
  
	  JImageClose = CAST_TO_FN_PTR(JImageClose_t, os::dll_lookup(handle, "JIMAGE_Close"));
  
	  JImagePackageToModule = 
      CAST_TO_FN_PTR(JImagePackageToModule_t, os::dll_lookup(handle, "JIMAGE_PackageToModule"));

  	JImageFindResource = CAST_TO_FN_PTR(JImageFindResource_t, os::dll_lookup(handle, "JIMAGE_FindResource"));
 	 	
  	JImageGetResource = CAST_TO_FN_PTR(JImageGetResource_t, os::dll_lookup(handle, "JIMAGE_GetResource"));
  	
  	JImageResourceIterator =
      CAST_TO_FN_PTR(JImageResourceIterator_t, os::dll_lookup(handle, "JIMAGE_ResourceIterator"));
  	
  	JImageResourcePath = CAST_TO_FN_PTR(JImage_ResourcePath_t, os::dll_lookup(handle, "JIMAGE_ResourcePath"));
}
```



### 2.3 ClassLoader::setup_bootstrap_search_path(); 注: 没有啥用

> **以本机的运行环境来说，Arguments::get_sysclasspath() 的结果只有一个字符串，而这个字符串会创建一个临时的ClassPathDirEntry后就不用了，`所以整个方法可以认为没啥用`**

```c++
void ClassLoader::setup_bootstrap_search_path() {
    // Arguments::get_sysclasspath() 方法返回的就是 _system_boot_class_path的值
  	const char* sys_class_path = Arguments::get_sysclasspath();

  	setup_boot_search_path(sys_class_path);
}

// Set up the _jrt_entry if present and boot append path
void ClassLoader::setup_boot_search_path(const char *class_path) {
  	int len = (int)strlen(class_path);
  	int end = 0;
  	bool set_base_piece = true;

    //<1>: 以path_separator为分隔符遍历class_path中的子字符串
  	for (int start = 0; start < len; start = end) { 
    		while (class_path[end] && class_path[end] != os::path_separator()[0]) {
      			end++;
    		}
    		// <1.1>: 创建局部变量path保存每次获取到的子字符串
    		char* path = NEW_RESOURCE_ARRAY(char, end - start + 1);
    		strncpy(path, &class_path[start], end - start);
    		path[end - start] = '\0';

        //<1.2>: set_base_piece为true时表示是第一次执行遍历操作
    		if (set_base_piece) { 
      			struct stat st;
            // 调用stat系统调用, 如果stat返回0, 表示path对应的文件存在
      			if (os::stat(path, &st) == 0) {
                // <1.3>: 创建ClassPathEntry
                // 关于 create_class_path_entry() 方法参考 2.3.1节
                // ----- 注: 但是这里创建了ClassPathEntry之后并没有使用, 所以........
        				ClassPathEntry* new_entry = create_class_path_entry(path, &st, false, false, CHECK);

        				if (Arguments::has_jimage()) { //<1.4>: 对于Java module的处理, 略过
          					_jrt_entry = new_entry;
        				}
      			} 
          
            //<1.4> 第一次执行遍历之后, 就将set_base_piece设置为false
      			set_base_piece = false;
    		} else {
            //<1.5>: 非第一次遍历, 所以set_base_peice为false, 会进入本分支
      			update_class_path_entry_list(path, false, true);
    		}

        //<1.6>: 跳过其中的空路径
    		while (class_path[end] == os::path_separator()[0]) {
      			end++;
    		}
  }
}
```



#### 2.3.1 ClassLoader::create_class_path_entry() 方法

> **简单点理解（省略java module相关），`create_class_path_entry() 方法基本就是在path为文件的情况下，创建ClassPathZipEntry对象；path为目录的情况下，创建ClassPathDirEntry对象`**
>
> > **`注: 这里Entry中只持有open之后的文件句柄，并没有对文件进行实际的读取以及进一步的处理操作`**

```c++
/*
path: jar包/文件/目录路径
st: path对应的文件属性
throw_exception: 处理过程中遇到错误, 是直接抛出错误还是让方法返回NULL后继续处理下一个path
is_boot_append: 是新建列表还是将解析到的ClassPathEntry附加到已有的列表中.
		如果是处理的第一个path, 则is_boot_append值为false; 否则is_boot_append值为true
TRAPS: 没啥意义		
*/
ClassPathEntry* ClassLoader::create_class_path_entry(const char *path, 
                                                     const struct stat* st,
                                                     bool throw_exception,
                                                     bool is_boot_append,
                                                     TRAPS) {
		JavaThread* thread = JavaThread::current();
  	ClassPathEntry* new_entry = NULL; //用于返回结果ClassPathEntry对象
  
  	if ((st->st_mode & S_IFMT) == S_IFREG) { //<1>: 如果path非目录
    		char* canonical_path = NEW_RESOURCE_ARRAY_IN_THREAD(thread, char, JVM_MAXPATHLEN);
        //<1.1>: 获取完整路径, 一般情况下与path相同
    		if (!get_canonical_path(path, canonical_path, JVM_MAXPATHLEN)) {
      			if (throw_exception) {
        				THROW_MSG_(vmSymbols::java_io_IOException(), "Bad pathname", NULL);
      			} else {
        				return NULL;
      			}
    		}
      
    		jint error;
				char* error_msg = NULL;
        //<1.2>: 基本就是调用open, zip持有path文件句柄
      	jzfile* zip = (*ZipOpen)(canonical_path, &error_msg);
        
      	if (zip != NULL && error_msg == NULL) {
            //<1.3>: 文件存在, 直接创建ClassPathZipEntry对象
        		new_entry = new ClassPathZipEntry(zip, path, is_boot_append);
      	} else {
        		//<1.4>: 错误日志
    		}

		} else { 
        // <2>: path文件为目录类型, 所以new_entry为 ClassPathDirEntry
    		new_entry = new ClassPathDirEntry(path); 
  	}
  	return new_entry; 
}
```



