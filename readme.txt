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
	 ├── cmdLine.isSubProcess()           --> 输出 dEQP 启动信息
	 ├── deSetRoundingMode()              --> 设置浮点数舍入模式
	 ├── qpWatchDog_create(...)           --> 初始化 WatchDog（超时保护）
	 ├── qpCrashHandler_create(...)       --> 初始化崩溃处理（Crash Handler）
	 ├── new TestContext(...)             --> 创建测试上下文
	 ├── new TestPackageRoot(...)         --> 是 dEQP 框架中测试树（test tree）构建的入口
	 └── runMode 分支:
		 ├── new TestSessionExecutor(...)    --> EXECUTE 正常跑测试，创建m_testExecutor
		 ├── writeCaselistsToStdout(...)
		 ├── ...
  └── LOOP
      {
	└──app->iterate()
                 ├── m_testExecutor->iterate()  		 -->✅核心，创建 App 对象
				├── STATE_TRAVERSE_HIERARCHY	 --> 遍历测试用例树（Test Tree），包括 package、group、case 等节点遍历测试用例结构树（Test Tree）
				|
				├──STATE_EXECUTE_TEST_CASE	 --> 实际执行一个测试用例，即调用 TestCase::iterate()
					├── DE_ASSERT(m_iterator.getState() == TestHierarchyIterator::STATE_LEAVE_NODE &&
          				       isTestNodeTypeExecutable(m_iterator.getNode()->getNodeType()));
						要求当前节点是 LEAVE_NODE 状态（说明我们刚刚进入并准备退出这个节点）；
						且节点类型是可执行的 test case（不是 group 或 package）；
						⚠️ 这点很重要：说明遍历在进入 node → 执行 → 离开 node 的生命周期中，执行发生在“LEAVE_NODE”节点阶段。
				        ├── iterateTestCase(testCase);
							├── m_caseExecutor->iterate(testCase)
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

2.app->iterate()
\GL_CTS_TEST\GL_CTS_TEST\GL_CTS_TEST\VK-GL-CTS-main\VK-GL-CTS-main\framework\common\tcuApp.cpp

🔹 m_testExecutor->iterate()
关键函数，每一次调用它，就执行一个用例（或者一次 init/iterate 步骤）。
如果你想“打断点逐步调试具体用例”，就从这里打断点：
(gdb) break TestSessionExecutor::iterate
(gdb) run

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

🌲 STATE_TRAVERSE_HIERARCHY —— 遍历测试用例结构树
🔍 目的：
遍历整个测试结构（测试用例组织成一个层级树），决定当前要不要进入 package/group/case，并在进入 case 后，切换到执行状态。
💡 结构树长啥样？
测试用例被组织成一棵树，节点类型有：
TestPackage（例如 gles2, gles3, egl）
TestGroup（例如 shaders, fbo, textures）
TestCase（最终的测试，比如 shaders.discard.basic.1）

📦 节点遍历逻辑：

const TestHierarchyIterator::State hierIterState = m_iterator.getState();
状态有三种：
STATE_ENTER_NODE: 即将进入这个节点
STATE_LEAVE_NODE: 已经遍历完这个节点的子节点
STATE_FINISHED: 整棵树遍历完了

🔁 核心逻辑（分进来和出去处理）：

if (hierIterState == ENTER_NODE || LEAVE_NODE)
{
    switch (nodeType)
    {
        case PACKAGE: enter/leave test package
        case GROUP:   enter/leave test group
        case CASE:    如果是 ENTER_NODE，则尝试进入 test case；
                     如果成功，就切到 STATE_EXECUTE_TEST_CASE
                     否则下一轮 LEAVE_NODE 自动 leaveTestCase()
    }
    m_iterator.next();  // 移动到下一个节点（自动进入/退出）
}
else if (hierIterState == FINISHED)
{
    m_status.isComplete = true;
    return false;  // 整个 test tree 遍历完了，测试结束
}
📌 注意：
enterTestCase() 成功后，状态切换到执行状态（STATE_EXECUTE_TEST_CASE）
m_iterator.next() 是核心，它决定我们走向下一个节点（可能是子节点，也可能是返回父节点）

🧪 STATE_EXECUTE_TEST_CASE —— 执行测试用例
🧠 目的：
对刚刚进入的 TestCase 节点，执行其实际逻辑

主要调用 TestCase::iterate()，这是实际的测试代码跑的地方

TestCase* testCase = static_cast<TestCase*>(m_iterator.getNode());
TestCase::IterateResult result = iterateTestCase(testCase);
✅ iterateTestCase() 返回值：

enum IterateResult
{
    CONTINUE, // 还有工作要做（下一帧继续调用）
    STOP      // 测试用例已完成，可以返回 TRAVERSE
};
🔁 执行结果影响状态：

if (iterResult == STOP)
    m_state = STATE_TRAVERSE_HIERARCHY;
即，如果这个用例执行完了，就切回去继续遍历结构树，进入下一个 case。



3.结果处理app->getResult().isComplete || app->getResult().numFailed)


E:\1代码仓库\GL_CTS_TEST\GL_CTS_TEST\GL_CTS_TEST\VK-GL-CTS-main\VK-GL-CTS-main\framework\common\tcuApp.cpp
App类构建


