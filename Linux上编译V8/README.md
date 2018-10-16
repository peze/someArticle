# Linux上编译V8


好吧~~看到这篇文章你也许会问： 编译一个有详细文档的V8有什么难的，你还写个东西？当翻译么？ 嗯。。。如果你真的这样认为，那我只能告诉你：朋友你太天真了。v8的文档是个梗好吧。。我都没想到google这么大的公司开源个项目，文档写得如此粗糙，你要是想通过V8的文档来做v8的编译或者embed开发，我只能说 兄弟，你会被坑得错过大年三十的。为什么呢？因为开发者光顾着升级代码，不更新文档了，文档里大量老旧错误信息，网上这方面的信息上也有很多问题。 好了，多的不说了。我直接贴出这个编译的过程，中间遇到的坑以及编译samples中hello-world.cc的方法：

## 步骤一

google 在v8这个的编译上面用了一个极其蛋疼的工具（推广需要）。以前可以直接通过git拉下来，然后通过 `make dependencies`这个方式来安装依赖，这个方式 我只能说这个老方式确实很不错。but！！这个方式已经不行了，你要是拉代码下来然后用这个命令，他会提醒你使用 gclient sync。好吧~所以这个命令是啥呢，你要是直接使用，会提醒你 压根儿没这个命令。嗯。。因为他要下google的一个包。命令如下：

	git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
	
然后为了让全局能够使用这个命令，我们必须还要clone下来的这个路径加入到PATH中

	vim ~/.zshrc
	#或者.bash_profile
	export PATH=~/git/depot_tools:"$PATH"
	
好了 这个时候就可以使用gclient sync了，我看到有网上的博客，在同步下来的v8的代码中使用这个命令，然后在里面又同步了一个v8，然后他们还专门把下面路径的v8里面的third_party文件夹拿出来。。。好吧。。这个看着确实有点搞笑。。这个gclient sync他是把代码整个拉下来并且把v8编译所需要的依赖包都拉下来的一个工具。首先你要配置v8拉取的地址

	gclient config https://chromium.googlesource.com/v8/v8.git
	
这个时候文件夹下生成一个文件 .gclient，看下里面的内容
	
	solutions = [
  		{   "name": "v8",
    		"url": "https://chromium.googlesource.com/v8/v8.git",
    		"deps_file": "DEPS",
    		"managed": True,
    		"custom_deps": {},
  		},
	]
	cache_dir = None


好吧，这里面其实就注意一个叫deps_file的文件默认是DEPS，里面会记录要依赖的文件。然后你就可以使用gclient sync了。。这个东西就可以把整套代码拉下来了。所以完全不用自己再去clone代码了 。。像网上blog里那位哥们儿在clone的代码里用这个命令 然后又拉取了一个完整v8，还说依赖包安装在了v8工程下v8目录里，然后把它copy出来也是蛮搞笑的。这一步必须要用挂vpn才能完成，用shadowsocks全局模式也是没办法。这个我有亲身体会，因为我用了全剧模式依然能copy代码 但是没法安装依赖，我曾经一度以为是我自己电脑出了问题，还有安装了几次，都以失败告终。。后来仔细看报错信息 才发现是个 网络错误

	Failed to connect to chromium.googlesource.com port 443: Operation timed out
	
为了验证是网络问题 我直接使用了公司的香港服务器（平时只作科学上网工具）。果然这次用了迅雷不及掩耳之势就搞定了。

### 步骤二

下载下来了代码以及依赖包以后，就是编译了。在linux下面就可以直接通过下面两个命令中的一个来编译

	make ia32.release //32位用
	make x64.release //64位用
	
成功所有的.o .a文件都在都在 `pwd`/out/x64.release/obj.target里面。现在来说编译就已经成功了。


### 步骤三

我们编译目的就是 使用编译出来的文件来进行我们自己的编译。首先我们来试试编译samples文件夹中的 hello_world.cc文件的编译。我用了v8官方的的命令

	g++ -I. hello-world.cc -o hello_world -Wl,--start-group out/x64.release/obj.target/{tools/gyp/libv8_{base,libbase,external_snapshot,libplatform},third_party/icu/libicu{uc,i18n,data}}.a -Wl,--end-group -lrt -ldl -pthread -std=c++0x
	
好吧。。这个会报几个错误 tools/gyp里面所有文件都不存在。这个时候呢，我就专门去`out/x64.release/obj.target`这个目录下找了下，我发现`{base,libbase,external_snapshot,libplatform}.a`这几个文件都在 `out/x64.release/obj.target/src`下 `third_party/icu/libicu{uc,i18n}}.a`两个文件存在，而`libicudata.a`文件并不存在，于是我直接取消了这个文件 然后使用

	g++ -I. samples/hello-world.cc -o hello_world -Wl,--start-group out/x64.release/obj.target/{src/libv8_{base,libbase,external_snapshot,libplatform},third_party/icu/libicu{uc,i18n,data}}.a -Wl,--end-group -lrt -ldl -pthread -std=c++0x
	
这次报的是没有v8和libplatform头文件。好吧 我就使用了 

	g++ -I./include samples/hello-world.cc -o hello_world -Wl,--start-group out/x64.release/obj.target/{src/libv8_{base,libbase,external_snapshot,libplatform},third_party/icu/libicu{uc,i18n}}.a -Wl,--end-group -lrt -ldl -pthread -std=c++0x
这个时候还是报错了报出了里面头文件的问题主要在于platform里面的文件`libplatform.h`

	#include "libplatform/libplatform-export.h"
	#include "libplatform/v8-tracing.h"
	#include "v8-platform.h"  // NOLINT(build/include)
看了文件夹里面文件的排布以后我发现`-I./include`这个肯定是没问题的。但是问题在于 hello-world.cc里面 我把头文件改成了：

	#include "libplatform/libplatform.h"
	#include "v8.h"
这样以后，编译依然失败了，不过值得庆幸的是。没有报头文件找不到的错误。但是报了这个错

		
	/home/peijie/v8/v8/out/x64.release/obj.target/v8_base/src/isolate.o: In function `v8::internal::Isolate::Deinit()':
	../src/isolate.cc:(.text._ZN2v88internal7Isolate6DeinitEv+0x107): undefined reference to `v8::sampler::Sampler::Stop()'
	/home/peijie/v8/v8/out/x64.release/obj.target/v8_base/src/log.o: In function `v8::internal::Profiler::Engage()':
	../src/log.cc:(.text._ZN2v88internal8Profiler6EngageEv+0x123): undefined reference to `v8::sampler::Sampler::IncreaseProfilingDepth()'
	../src/log.cc:(.text._ZN2v88internal8Profiler6EngageEv+0x133): undefined reference to `v8::sampler::Sampler::Start()'
	/home/peijie/v8/v8/out/x64.release/obj.target/v8_base/src/log.o: In function `v8::internal::Profiler::Disengage()':
	../src/log.cc:(.text._ZN2v88internal8Profiler9DisengageEv+0x3d): undefined reference to `v8::sampler::Sampler::Stop()'
	../src/log.cc:(.text._ZN2v88internal8Profiler9DisengageEv+0x45): undefined reference to `v8::sampler::Sampler::DecreaseProfilingDepth()'
	/home/peijie/v8/v8/out/x64.release/obj.target/v8_base/src/log.o: In function `v8::internal::Logger::SetUp(v8::internal::Isolate*)':
	../src/log.cc:(.text._ZN2v88internal6Logger5SetUpEPNS0_7IsolateE+0x531): undefined reference to `v8::sampler::Sampler::Sampler(v8::Isolate*)'
	/home/peijie/v8/v8/out/x64.release/obj.target/v8_base/src/log.o: In function `v8::internal::Ticker::~Ticker()':
	../src/log.cc:(.text._ZN2v88internal6TickerD2Ev[_ZN2v88internal6TickerD2Ev]+0x16): undefined reference to `v8::sampler::Sampler::Stop()'
	../src/log.cc:(.text._ZN2v88internal6TickerD2Ev[_ZN2v88internal6TickerD2Ev]+0x2e): undefined reference to `v8::sampler::Sampler::~Sampler()'
	/home/peijie/v8/v8/out/x64.release/obj.target/v8_base/src/log.o: In function `v8::internal::Ticker::~Ticker()':
	../src/log.cc:(.text._ZN2v88internal6TickerD0Ev[_ZN2v88internal6TickerD0Ev]+0x16): undefined reference to `v8::sampler::Sampler::Stop()'
	../src/log.cc:(.text._ZN2v88internal6TickerD0Ev[_ZN2v88internal6TickerD0Ev]+0x2d): undefined reference to `v8::sampler::Sampler::~Sampler()'
	/home/peijie/v8/v8/out/x64.release/obj.target/v8_base/src/log.o: In function `v8::internal::SamplingThread::Run()':
	../src/log.cc:(.text._ZN2v88internal14SamplingThread3RunEv[_ZN2v88internal14SamplingThread3RunEv]+0x11): undefined reference to `v8::sampler::Sampler::DoSample()'
	/home/peijie/v8/v8/out/x64.release/obj.target/v8_base/src/profiler/cpu-profiler.o: In function `v8::internal::ProfilerEventsProcessor::ProfilerEventsProcessor(v8::internal::Isolate*, v8::internal::ProfileGenerator*, v8::base::TimeDelta)':
	../src/profiler/cpu-profiler.cc:(.text._ZN2v88internal23ProfilerEventsProcessorC2EPNS0_7IsolateEPNS0_16ProfileGeneratorENS_4base9TimeDeltaE+0x51): undefined reference to `v8::sampler::Sampler::Sampler(v8::Isolate*)'
	../src/profiler/cpu-profiler.cc:(.text._ZN2v88internal23ProfilerEventsProcessorC2EPNS0_7IsolateEPNS0_16ProfileGeneratorENS_4base9TimeDeltaE+0x1ec): undefined reference to `v8::sampler::Sampler::IncreaseProfilingDepth()'
	/home/peijie/v8/v8/out/x64.release/obj.target/v8_base/src/profiler/cpu-profiler.o: In function `v8::internal::ProfilerEventsProcessor::~ProfilerEventsProcessor()':
	../src/profiler/cpu-profiler.cc:(.text._ZN2v88internal23ProfilerEventsProcessorD2Ev+0x14): undefined reference to `v8::sampler::Sampler::DecreaseProfilingDepth()'
	/home/peijie/v8/v8/out/x64.release/obj.target/v8_base/src/profiler/cpu-profiler.o: In function `v8::internal::ProfilerEventsProcessor::Run()':
	../src/profiler/cpu-profiler.cc:(.text._ZN2v88internal23ProfilerEventsProcessor3RunEv+0x11): undefined reference to `v8::sampler::Sampler::DoSample()'
	/home/peijie/v8/v8/out/x64.release/obj.target/v8_base/src/profiler/cpu-profiler.o: In function `v8::internal::CpuSampler::~CpuSampler()':
	../src/profiler/cpu-profiler.cc:(.text._ZN2v88internal10CpuSamplerD0Ev[_ZN2v88internal10CpuSamplerD0Ev]+0x5): undefined reference to `v8::sampler::Sampler::~Sampler()'
	/home/peijie/v8/v8/out/x64.release/obj.target/v8_base/src/profiler/cpu-profiler.o:(.rodata._ZTVN2v88internal10CpuSamplerE[_ZTVN2v88internal10CpuSamplerE]+0x10): undefined reference to `v8::sampler::Sampler::~Sampler()'
	/home/peijie/v8/v8/out/x64.release/obj.target/v8_base/src/v8.o: In function `v8::internal::V8::TearDown()':
	../src/v8.cc:(.text._ZN2v88internal2V88TearDownEv+0x1b): undefined reference to `v8::sampler::Sampler::TearDown()'
	/home/peijie/v8/v8/out/x64.release/obj.target/v8_base/src/v8.o: In function `v8::internal::V8::InitializeOncePerProcessImpl()':
	../src/v8.cc:(.text._ZN2v88internal2V828InitializeOncePerProcessImplEv+0x98): undefined reference to `v8::sampler::Sampler::SetUp()'
	collect2: error: ld returned 1 exit status
好吧 在这里我的思维走进过误区，就是觉得是少了`libicudata.a`这个文件，而且为此到处找了这个文件。但是后来我仔细看了报错信息，里面都是跟Sampler对象有关，而src的文件夹中刚好有一个`libv8_libsampler.a`文件，于是我又在编译语句中加入了这个文件

	g++ -I./include samples/hello-world.cc -o hello_world -Wl,--start-group out/x64.release/obj.target/{src/libv8_{base,libbase,external_snapshot,libplatform,libsampler},third_party/icu/libicu{uc,i18n}}.a -Wl,--end-group -lrt -ldl -pthread -std=c++0x
	
这次成功编译，我迫不及待的运行。结果。。。

		Failed to open startup resource './snapshot_blob.bin'.


	#
	# Fatal error in ../src/snapshot/natives-external.cc, line 122
	# Check failed: holder_.
	#

	==== C stack trace ===============================

	    ./hello_world() [0xbe7dce]
	    ./hello_world() [0xbe6883]
	    ./hello_world() [0xbe9ecc]
	    ./hello_world() [0x62b31d]
	    ./hello_world() [0x6328af]
	    ./hello_world() [0x6cbb15]
	    ./hello_world() [0x42dece]
	    ./hello_world() [0x406545]
	    /lib64/libc.so.6(__libc_start_main+0xf5) [0x7f258e0e4e65]
	    ./hello_world() [0x4063f9]
	Illegal instruction
	
好吧到这一步的时候，已经没有什么问题了，官方文档上给了解决方案：
	
	V8 requires its 'startup snapshot' to run. Copy the snapshot files to where your binary is stored: cp out/x64.release/*.bin .

我直接执行了这步操作就已经成功的输出了结果。

### 总结

这个编译的过程真是一波三折。我觉得其中也有很多我其实并没有理解的地方，比如换头文件引用路径这个。在直接使用`make x64.release`的时候 是能够成功编译hello-world的（当然我改了路径使用make也能够成功）。然后我看了v8的makefile里面没有使用g++的的命令来直接编译，而是通过gyp来编译的。所以他为何能够成功对我也没有指导意义，于是我只有用我能够成功的方法。但总的来说v8文档的问题是必然存在的。虽然我这个方法不是最好的方法，但是却是行之有效的方法，鄙人对C++的接触还不是很深。先做个记录以后等深入了或许能发现原因再回来解释。