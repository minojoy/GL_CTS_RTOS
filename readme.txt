# GL_CTS_RTOS
GL_CTS run on rtos

一、单测试结构梳理
  main()                                 //函数入口：GL_CTS_TEST\GL_CTS_TEST\GL_CTS_TEST\VK-GL-CTS-main\VK-GL-CTS-main\framework\platform\tcuMain.cpp
  └── CommandLine cmdLine(argc, argv)    //命令行参数解析 
  └── platform(createPlatform())         //创建平台对象（窗口系统 + 渲染上下文）
  └── archive(cmdLine.getArchiveDir())   //测试数据文件系统，如 .vert, .frag 文件的来源。
  └── log()                              //生成 XML 或文本格式的测试输出。
  └── app()                              //✅核心，创建 App 对象
	App::App(...)
	 ├── cmdLine.getRunMode()
	 ├── qpWatchDog_create(...)           --> watchdog线程
	 ├── qpCrashHandler_create(...)       --> 安装信号钩子
	 ├── new TestContext(...)             --> 测试上下文初始化
	 ├── new TestPackageRoot(...)         --> 加载注册测试包
	 └── runMode 分支:
		 ├── new TestSessionExecutor(...)    --> 开始测试执行
		 ├── writeCaselistsToStdout(...)
		 ├── ...
  └── LOOP
      {
	└──app->iterate()
        └──app->getResult()
      }

1.App::App(...)
	✅ 1. 输出 dEQP 启动信息
	if (!cmdLine.isSubProcess())
	{
		print("dEQP Core %s ...\n", qpGetReleaseName(), ...);
	}


	✅ 2. 设置浮点数舍入模式（平台相关）
	deSetRoundingMode(DE_ROUNDINGMODE_TO_NEAREST_EVEN);
	这个是平台兼容性设置，可能会影响浮点测试的精度。


	✅ 3. 初始化 WatchDog（超时保护）
	if (cmdLine.isWatchDogEnabled())
		m_watchDog = qpWatchDog_create(...);
	这个是用于防止死循环或测试超时，内部会创建定时器线程。你可以继续查看 qpWatchDog_create 的实现。


	✅ 4. 初始化崩溃处理（Crash Handler）
	if (cmdLine.isCrashHandlingEnabled())
		m_crashHandler = qpCrashHandler_create(onCrash, this);
	安装崩溃钩子，比如 SEGFAULT 或 ABORT，适合做回溯、重试或日志打点。


	✅ 5. 创建测试上下文 TestContext
	m_testCtx = new TestContext(...);
	这是整个测试流程的核心上下文，包括日志、平台接口、参数等信息的整合体。你要拆 demo，这里是必须深入的对象。



	✅ 6. 创建测试根 TestPackageRoot
	m_testRoot = new TestPackageRoot(*m_testCtx, TestPackageRegistry::getSingleton());
	这一句加载了注册的测试包，是 dEQP 框架中测试树（test tree）构建的入口。例如：
	GLES2、GLES3、Vulkan 等子系统，都会注册各自的测试包。

	TestPackageRegistry::getSingleton() 返回静态注册的包（常见于 *_init.cpp 文件）。

	✅ 7. 根据命令行参数执行测试或导出用例列表
	switch (cmdLine.getRunMode())
	{
	  case RUNMODE_EXECUTE:
		m_testExecutor = new TestSessionExecutor(...);
		break;
	  case RUNMODE_DUMP_STDOUT_CASELIST:
		writeCaselistsToStdout(...);
		break;
	  case RUNMODE_DUMP_XML_CASELIST:
		writeXmlCaselistsToFiles(...);
		break;
	  ...
	}
	这是 根据参数执行不同动作：

	模式	动作
	EXECUTE	正常跑测试
	DUMP_STDOUT_CASELIST	打印所有测试用例名
	DUMP_XML_CASELIST	导出 XML
	VERIFY_AMBER_COHERENCY	检查 Amber 能力一致性






!app->iterate()
		
		E:\1代码仓库\GL_CTS_TEST\GL_CTS_TEST\GL_CTS_TEST\VK-GL-CTS-main\VK-GL-CTS-main\framework\common\tcuApp.cpp

		🔹 m_testExecutor->iterate()
		关键函数，每一次调用它，就执行一个用例（或者一次 init/iterate 步骤）。

		如果你想“打断点逐步调试具体用例”，就从这里打断点：

		bash
		复制
		编辑
		(gdb) break TestSessionExecutor::iterate
		(gdb) run
		可以再深入下去分析每个用例是怎么跑的，比如 ShaderDiscardTests。


		TestSessionExecutor::iterate() 函数的完整实现
		控制整个测试用例树的遍历与单个用例的执行
		📌 举个形象例子：
		想象你在逛超市，每个状态代表你当前的“行为”：
		STATE_TRAVERSE_HIERARCHY：你在超市里浏览货架（走结构树）
		STATE_EXECUTE_TEST_CASE：你挑中了某样商品，打开来看质量如何（执行测试）
		你不会在一边走路一边拆包装 —— 你是先找到目标，再集中执行检查。

		STATE_TRAVERSE_HIERARCHY	遍历测试用例结构树（Test Tree）
			可能遇到的是：
			Package 节点（比如 GLES3 功能包）
			Group 节点（一个测试小类，比如 shader.discard）
			Case 节点（真正的测试用例）
		
		STATE_EXECUTE_TEST_CASE	    真正执行测试用例
			🧪 STATE_EXECUTE_TEST_CASE：
			如果当前节点是一个可执行的 TestCase，就切换到这个状态
			然后调用：TestCase::iterate()，执行实际测试逻辑（比如 setup、绘制、验证）




结果获取
!app->getResult().isComplete || app->getResult().numFailed)


E:\1代码仓库\GL_CTS_TEST\GL_CTS_TEST\GL_CTS_TEST\VK-GL-CTS-main\VK-GL-CTS-main\framework\common\tcuApp.cpp
App类构建


