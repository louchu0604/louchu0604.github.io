---
title: app启动过程
date: 2018-11-26 00:50:28
tags:
---


最开始的时候呢，是看了sunny的博客：http://blog.sunnyxx.com/2014/08/30/objc-pre-main/
和mike ash的博客：
https://www.mikeash.com/pyblog/friday-qa-2012-11-09-dyld-dynamic-linking-on-os-x.html
但是没看过源码一头雾水的，所以花了周末的时间，看了源码。
下面的是我的一些总结，也算是题纲。然后我的草稿也继续保留。
刚开始写博客，希望思路会越来越好。
引自sunny的博客
>dyld（the dynamic link editor），Apple 的动态链接器，系统 kernel 做好启动程序的初始准备后，交给 dyld 负责，援引并翻译《 Mike Ash 这篇 blog 》对 dyld 作用顺序的概括：
1. 从 kernel 留下的原始调用栈引导和启动自己
2. 将程序依赖的动态链接库递归加载进内存，当然这里有缓存机制
3. non-lazy 符号立即 link 到可执行文件，lazy 的存表里
4. Runs static initializers for the executable
5. 找到可执行文件的 main 函数，准备参数并调用
6. 程序执行中负责绑定 lazy 符号、提供 runtime dynamic loading services、提供调试器接口
7. 程序main函数 return 后执行 static terminator
8. 某些场景下 main 函数结束后调 libSystem 的 _exit 函数

### `Mach-O文件`
Mach-O文件格式是OSX和iOS系统上的可执行文件格式，【.O文件、动态库都是这个格式】
结构如下:
`mach_header`


# app 编译打包的过程
这个地方贴一下计算机原理的示意图
# app 启动的过程

dyld 
iOS中系统级别的framework都是动态链接的。原因有三，一是为了减少体积，编译的时候不需要打包进去。二是代码可以共用，因为系统下的APP用到的同一个framework都是一样的。  三是便于更新。
静态链接的的framework比如我们常用到的第三方库SVProgressHUD是在编译时链接好的。
iOS不允许我们用到系统以外的动态链接库。原因，emmm,everybody knows

源码里的汇编太难啃 ，揪出 `__dyld_start` ,调用栈如下
* `__dyld_start`
    * `dyldbootstrap::start`
        * `_main`
`__dyld_start`
```objc
__dyld_start:
	mov 	x28, sp
	and     sp, x28, #~15		// force 16-byte alignment of stack
	mov	x0, #0
	mov	x1, #0
	stp	x1, x0, [sp, #-16]!	// make aligned terminating frame
	mov	fp, sp			// set up fp to point to terminating frame
	sub	sp, sp, #16             // make room for local variables
	ldr     x0, [x28]		// get app's mh into x0
 	ldr     x1, [x28, #8]           // get argc into x1 (kernel passes 32-bit int argc as 64-bits on stack to keep alignment)
	add     x2, x28, #16		// get argv into x2
	adrp	x4,___dso_handle@page
	add 	x4,x4,___dso_handle@pageoff // get dyld's mh in to x4
	adrp	x3,__dso_static@page
	ldr 	x3,[x3,__dso_static@pageoff] // get unslid start of dyld
	sub 	x3,x4,x3		// x3 now has slide of dyld
	mov	x5,sp                   // x5 has &startGlue
	
	// call dyldbootstrap::start(app_mh, argc, argv, slide, dyld_mh, &startGlue)
	bl	__ZN13dyldbootstrap5startEPK12macho_headeriPPKclS2_Pm
	mov	x16,x0                  // save entry point address in x16
	ldr     x1, [sp]
	cmp	x1, #0
	b.ne	Lnew

	// LC_UNIXTHREAD way, clean up stack and jump to result
	add	sp, x28, #8		// restore unaligned stack pointer without app mh
	br	x16			// jump to the program's entry point

```
`dyldbootstrap::start`做的主要事情是初始化参数，并调用main，返回mian函数的地址
`main`函数：
```objc
......
return dyld::_main(appsMachHeader, appsSlide, argc, argv, envp, apple, startGlue);
......
uintptr_t
_main(const macho_header* mainExecutableMH, uintptr_t mainExecutableSlide, 
		int argc, const char* argv[], const char* envp[], const char* apple[], 
		uintptr_t* startGlue)

```
参数：
`mainExecutableMH`:当前app的Mach-O头部信息。 `macho_header`结构体的结构可以在源码中查看
【源码中：
* The 64-bit mach header appears at the very beginning of object files for 64-bit architectures.
* The 32-bit mach header appears at the very beginning of the object file for 32-bit architectures.】
`mainExecutableSlide`:地址的偏移量
`argc`:
`argv[]`:
`envp[]`:
`apple[]`:
`startGlue`:

后面的都是草稿，，别看了。。


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
`mainExecutableMH(macho_header)` 头部信息和 `mainExecutableSlide(uintptr_t)`偏移量
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
```
uintptr_t
_main(const macho_header* mainExecutableMH, uintptr_t mainExecutableSlide, 
		int argc, const char* argv[], const char* envp[], const char* apple[], 
		uintptr_t* startGlue)
{
	uintptr_t result = 0;
	//头部信息
	sMainExecutableMachHeader = mainExecutableMH;
#if __MAC_OS_X_VERSION_MIN_REQUIRED
	// if this is host dyld, check to see if iOS simulator is being run
	//准备工作
	const char* rootPath = _simple_getenv(envp, "DYLD_ROOT_PATH");
	if ( rootPath != NULL ) {
		// look to see if simulator has its own dyld
		char simDyldPath[PATH_MAX]; 
		strlcpy(simDyldPath, rootPath, PATH_MAX);
		strlcat(simDyldPath, "/usr/lib/dyld_sim", PATH_MAX);
		int fd = my_open(simDyldPath, O_RDONLY, 0);
		if ( fd != -1 ) {
			result = useSimulatorDyld(fd, mainExecutableMH, simDyldPath, argc, argv, envp, apple, startGlue);
			if ( !result && (*startGlue == 0) )
				halt("problem loading iOS simulator dyld");
			return result;
		}
	}
#endif

	CRSetCrashLogMessage("dyld: launch started");

#if LOG_BINDINGS
	char bindingsLogPath[256];
	
	const char* shortProgName = "unknown";
	if ( argc > 0 ) {
		shortProgName = strrchr(argv[0], '/');
		if ( shortProgName == NULL )
			shortProgName = argv[0];
		else 
			++shortProgName;
	}
	//拼接路径
	mysprintf(bindingsLogPath, "/tmp/bindings/%d-%s", getpid(), shortProgName);
	// 读取
	sBindingsLogfile = open(bindingsLogPath, O_WRONLY | O_CREAT, 0666);
	if ( sBindingsLogfile == -1 ) {
		::mkdir("/tmp/bindings", 0777);
		sBindingsLogfile = open(bindingsLogPath, O_WRONLY | O_CREAT, 0666);
	}
	//dyld::log("open(%s) => %d, errno = %d\n", bindingsLogPath, sBindingsLogfile, errno);
#endif	
//设置上下文
	setContext(mainExecutableMH, argc, argv, envp, apple);

	// Pickup the pointer to the exec path.
	//指针指向执行路径
	sExecPath = _simple_getenv(apple, "executable_path");

	// <rdar://problem/13868260> Remove interim apple[0] transition code from dyld
	if (!sExecPath) sExecPath = apple[0];
	
	bool ignoreEnvironmentVariables = false;
	if ( sExecPath[0] != '/' ) {
		// have relative path, use cwd to make absolute
		char cwdbuff[MAXPATHLEN];
	    if ( getcwd(cwdbuff, MAXPATHLEN) != NULL ) {
			// maybe use static buffer to avoid calling malloc so early...
			char* s = new char[strlen(cwdbuff) + strlen(sExecPath) + 2];
			strcpy(s, cwdbuff);
			strcat(s, "/");
			strcat(s, sExecPath);
			sExecPath = s;
		}
	}
	// Remember short name of process for later logging
	//为之后的日志输出记录
	sExecShortName = ::strrchr(sExecPath, '/');
	if ( sExecShortName != NULL )
		++sExecShortName;
	else
		sExecShortName = sExecPath;
    sProcessIsRestricted = processRestricted(mainExecutableMH, &ignoreEnvironmentVariables, &sProcessRequiresLibraryValidation);
    if ( sProcessIsRestricted ) {
#if SUPPORT_LC_DYLD_ENVIRONMENT
		checkLoadCommandEnvironmentVariables();
#endif 	
		pruneEnvironmentVariables(envp, &apple);
		// set again because envp and apple may have changed or moved
		setContext(mainExecutableMH, argc, argv, envp, apple);
	}
	else {
		if ( !ignoreEnvironmentVariables )
			checkEnvironmentVariables(envp);
		defaultUninitializedFallbackPaths(envp);
	}
	if ( sEnv.DYLD_PRINT_OPTS )
		printOptions(argv);
	if ( sEnv.DYLD_PRINT_ENV ) 
		printEnvironmentVariables(envp);
	getHostInfo(mainExecutableMH, mainExecutableSlide);
	// install gdb notifier
	stateToHandlers(dyld_image_state_dependents_mapped, sBatchHandlers)->push_back(notifyGDB);
	stateToHandlers(dyld_image_state_mapped, sSingleHandlers)->push_back(updateAllImages);
	// make initial allocations large enough that it is unlikely to need to be re-alloced
	sAllImages.reserve(INITIAL_IMAGE_COUNT);
	sImageRoots.reserve(16);
	sAddImageCallbacks.reserve(4);
	sRemoveImageCallbacks.reserve(4);
	sImageFilesNeedingTermination.reserve(16);
	sImageFilesNeedingDOFUnregistration.reserve(8);
	
#ifdef WAIT_FOR_SYSTEM_ORDER_HANDSHAKE
	// <rdar://problem/6849505> Add gating mechanism to dyld support system order file generation process
	WAIT_FOR_SYSTEM_ORDER_HANDSHAKE(dyld::gProcessInfo->systemOrderFlag);
#endif
	

	try {
		// add dyld itself to UUID list
		// 将dyldimage加入uuidlist
		addDyldImageToUUIDList();
		CRSetCrashLogMessage(sLoadingCrashMessage);
		// instantiate ImageLoader for main executable
		//初始化instantiateFromLoadedImage 
		sMainExecutable = instantiateFromLoadedImage(mainExecutableMH, mainExecutableSlide, sExecPath);
		// 配置上下文信息
		gLinkContext.mainExecutable = sMainExecutable;
		gLinkContext.processIsRestricted = sProcessIsRestricted;
		gLinkContext.processRequiresLibraryValidation = sProcessRequiresLibraryValidation;
		gLinkContext.mainExecutableCodeSigned = hasCodeSignatureLoadCommand(mainExecutableMH);

#if TARGET_IPHONE_SIMULATOR
	#if TARGET_OS_WATCH || TARGET_OS_TV
		// disable error during bring up of these simulators
	#else
		// check main executable is not too new for this OS
		{
			if ( ! isSimulatorBinary((uint8_t*)mainExecutableMH, sExecPath) ) {
				throwf("program was built for Mac OS X and cannot be run in simulator"); 
			}
			uint32_t mainMinOS = sMainExecutable->minOSVersion();
			// dyld is always built for the current OS, so we can get the current OS version
			// from the load command in dyld itself.
			uint32_t dyldMinOS = ImageLoaderMachO::minOSVersion((const mach_header*)&__dso_handle);
			if ( mainMinOS > dyldMinOS ) {
				throwf("app was built for iOS %d.%d which is newer than this simulator %d.%d", 
						mainMinOS >> 16, ((mainMinOS >> 8) & 0xFF),
						dyldMinOS >> 16, ((dyldMinOS >> 8) & 0xFF));
			}
		}
	#endif
#endif

		// load shared cache
		checkSharedRegionDisable();
	#if DYLD_SHARED_CACHE_SUPPORT
		if ( gLinkContext.sharedRegionMode != ImageLoader::kDontUseSharedRegion )
			mapSharedCache();
	#endif

		// Now that shared cache is loaded, setup an versioned dylib overrides
	#if SUPPORT_VERSIONED_PATHS
		checkVersionedPaths();
	#endif

		// load any inserted libraries
		if	( sEnv.DYLD_INSERT_LIBRARIES != NULL ) {
			for (const char* const* lib = sEnv.DYLD_INSERT_LIBRARIES; *lib != NULL; ++lib) 
				loadInsertedDylib(*lib);
		}
		// record count of inserted libraries so that a flat search will look at 
		// inserted libraries, then main, then others.
		sInsertedDylibCount = sAllImages.size()-1;

		// link main executable
		gLinkContext.linkingMainExecutable = true;
		link(sMainExecutable, sEnv.DYLD_BIND_AT_LAUNCH, true, ImageLoader::RPathChain(NULL, NULL));
		sMainExecutable->setNeverUnloadRecursive();
		if ( sMainExecutable->forceFlat() ) {
			gLinkContext.bindFlat = true;
			gLinkContext.prebindUsage = ImageLoader::kUseNoPrebinding;
		}

		// link any inserted libraries
		// do this after linking main executable so that any dylibs pulled in by inserted 
		// dylibs (e.g. libSystem) will not be in front of dylibs the program uses
		
		if ( sInsertedDylibCount > 0 ) {
			//链接  细节见下方
			for(unsigned int i=0; i < sInsertedDylibCount; ++i) {
				ImageLoader* image = sAllImages[i+1];
				link(image, sEnv.DYLD_BIND_AT_LAUNCH, true, ImageLoader::RPathChain(NULL, NULL));
				image->setNeverUnloadRecursive();
			}
			// only INSERTED libraries can interpose
			// register interposing info after all inserted libraries are bound so chaining works
			// interposing info 这个是什么意思
			for(unsigned int i=0; i < sInsertedDylibCount; ++i) {
				ImageLoader* image = sAllImages[i+1];
				image->registerInterposing();
			}
		}

		// <rdar://problem/19315404> dyld should support interposition even without DYLD_INSERT_LIBRARIES
		for (int i=sInsertedDylibCount+1; i < sAllImages.size(); ++i) {
			ImageLoader* image = sAllImages[i];
			if ( image->inSharedCache() )
				continue;
			image->registerInterposing();
		}

		// apply interposing to initial set of images
		for(int i=0; i < sImageRoots.size(); ++i) {
			sImageRoots[i]->applyInterposing(gLinkContext);
		}
		gLinkContext.linkingMainExecutable = false;
		
		// <rdar://problem/12186933> do weak binding only after all inserted images linked
		sMainExecutable->weakBind(gLinkContext);
		
		CRSetCrashLogMessage("dyld: launch, running initializers");

		//调用所有的image的initializer方法
	#if SUPPORT_OLD_CRT_INITIALIZATION
		// Old way is to run initializers via a callback from crt1.o
		if ( ! gRunInitializersOldWay ) 
			initializeMainExecutable(); 
	#else
		// run all initializers
		initializeMainExecutable(); 
	#endif
		// find entry point for main executable
		result = (uintptr_t)sMainExecutable->getThreadPC();
		if ( result != 0 ) {
			// main executable uses LC_MAIN, needs to return to glue in libdyld.dylib
			if ( (gLibSystemHelpers != NULL) && (gLibSystemHelpers->version >= 9) )
				*startGlue = (uintptr_t)gLibSystemHelpers->startGlueToCallExit;
			else
				halt("libdyld.dylib support not present for LC_MAIN");
		}
		else {
			// main executable uses LC_UNIXTHREAD, dyld needs to let "start" in program set up for main()
			result = (uintptr_t)sMainExecutable->getMain();
			*startGlue = 0;
		}
	}
	catch(const char* message) {
		syncAllImages();
		halt(message);
	}
	catch(...) {
		dyld::log("dyld: launch failed\n");
	}

	CRSetCrashLogMessage(NULL);
	
	return result;
}
```

`addDyldImageToUUIDList`细节


```objc

static void addDyldImageToUUIDList()
{
	const struct macho_header* mh = (macho_header*)&__dso_handle;
	const uint32_t cmd_count = mh->ncmds;
	const struct load_command* const cmds = (struct load_command*)((char*)mh + sizeof(macho_header));
	const struct load_command* cmd = cmds;
	for (uint32_t i = 0; i < cmd_count; ++i) {
		switch (cmd->cmd) {
			case LC_UUID: {
				uuid_command* uc = (uuid_command*)cmd;
				dyld_uuid_info info;
				info.imageLoadAddress = (mach_header*)mh;
				memcpy(info.imageUUID, uc->uuid, 16);
				addNonSharedCacheImageUUID(info);
				return;
			}
		}
		cmd = (const struct load_command*)(((char*)cmd)+cmd->cmdsize);
	}
}

```
`ImageLoader::link`细节  也就是题纲说的`link`的细节
```objc
void ImageLoader::link(const LinkContext& context, bool forceLazysBound, bool preflightOnly, bool neverUnload, const RPathChain& loaderRPaths)
{
	//dyld::log("ImageLoader::link(%s) refCount=%d, neverUnload=%d\n", this->getPath(), fDlopenReferenceCount, fNeverUnload);
	
	// clear error strings
	(*context.setErrorStrings)(dyld_error_kind_none, NULL, NULL, NULL);

	uint64_t t0 = mach_absolute_time();
	//递归加载所有的library
	this->recursiveLoadLibraries(context, preflightOnly, loaderRPaths);
	context.notifyBatch(dyld_image_state_dependents_mapped);
	
	// we only do the loading step for preflights
	if ( preflightOnly )
		return;
		
	uint64_t t1 = mach_absolute_time();
	context.clearAllDepths();
	this->recursiveUpdateDepth(context.imageCount());

	uint64_t t2 = mach_absolute_time();
	//递归修复地址
 	this->recursiveRebase(context);
	context.notifyBatch(dyld_image_state_rebased);
	//递归绑定
	uint64_t t3 = mach_absolute_time();
 	this->recursiveBind(context, forceLazysBound, neverUnload);

	uint64_t t4 = mach_absolute_time();
	if ( !context.linkingMainExecutable )
		this->weakBind(context);
	uint64_t t5 = mach_absolute_time();	

	context.notifyBatch(dyld_image_state_bound);
	uint64_t t6 = mach_absolute_time();	

	std::vector<DOFInfo> dofs;
	this->recursiveGetDOFSections(context, dofs);
	context.registerDOFs(dofs);
	uint64_t t7 = mach_absolute_time();	

	// interpose any dynamically loaded images
	if ( !context.linkingMainExecutable && (fgInterposingTuples.size() != 0) ) {
		this->recursiveApplyInterposing(context);
	}
	
	// clear error strings
	(*context.setErrorStrings)(dyld_error_kind_none, NULL, NULL, NULL);

	fgTotalLoadLibrariesTime += t1 - t0;
	fgTotalRebaseTime += t3 - t2;
	fgTotalBindTime += t4 - t3;
	fgTotalWeakBindTime += t5 - t4;
	fgTotalDOF += t7 - t6;
	
	// done with initial dylib loads
	fgNextPIEDylibAddress = 0;
}

```
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
