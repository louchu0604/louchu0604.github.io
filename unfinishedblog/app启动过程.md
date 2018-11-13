main()调用之前的加载过程
* 先将可执行文件加载，
* 加载dyld

#### dyld
* dyld加载dylib（）



调用栈
* `__dyld_start`
    * `dyldbootstrap::start`
        * `_main`

#### 先看start()

```objc
uintptr_t start(const struct macho_header* appsMachHeader, int argc, const char* argv[], 
				intptr_t slide, const struct macho_header* dyldsMachHeader,
				uintptr_t* startGlue)
{
	// if kernel had to slide dyld, we need to fix up load sensitive locations
	// we have to do this before using any global variables
    //如果不是在规定的地址处载入dyld 则需要重新定位dyld
	if ( slide != 0 ) {
		rebaseDyld(dyldsMachHeader, slide);
	}

	// allow dyld to use mach messaging
    //初始化
	mach_init();

	// kernel sets up env pointer to be just past end of agv array
	const char** envp = &argv[argc+1];
	
	// kernel sets up apple pointer to be just past end of envp array
	const char** apple = envp;
	while(*apple != NULL) { ++apple; }
	++apple;

	// set up random value for stack canary
	__guard_setup(apple);

#if DYLD_INITIALIZER_SUPPORT
	// run all C++ initializers inside dyld
	runDyldInitializers(dyldsMachHeader, slide, argc, argv, envp, apple);
#endif

	// now that we are done bootstrapping dyld, call dyld's main
	uintptr_t appsSlide = slideOfMainExecutable(appsMachHeader);
	return dyld::_main(appsMachHeader, appsSlide, argc, argv, envp, apple, startGlue);
}
```

接下来是`_main`
这个函数有点长，要耐心看 
前两个入参是
`mainExecutableMH(macho_header)` 头部信息和 `mainExecutableSlide(uintptr_t)`偏移量


动态链接库包括： 
iOS 中用到的所有系统 framework 
加载OC runtime方法的libobjc， 
系统级别的libSystem，例如libdispatch(GCD)和libsystem_blocks (Block)

动态链接库加载的具体流程
动态链接库的加载步骤具体分为5步：
1.load dylibs image 读取库镜像文件 
2.Rebase image 
3.Bind image 
4.Objc setup 
5.initializers


