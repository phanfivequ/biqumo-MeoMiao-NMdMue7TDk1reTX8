# React Native 错误处理完全指南

深入解析跨平台应用中的 JS 错误、原生崩溃及异常监控方案，附实战代码与最佳实践。

在 React Native 跨平台开发中，错误处理是保障应用稳定性与用户体验的核心环节。不同于纯 Web 应用或原生应用，React Native 应用的错误来源更为复杂——既包含 JavaScript 层的逻辑错误，也涉及 iOS/Android 双端的原生模块异常，甚至可能因 JS 与原生通信异常引发崩溃。本文将系统梳理 React Native 中的错误类型、核心处理工具、实战场景解决方案及监控策略，帮助开发者构建更稳健的跨平台应用。

## 一、React Native 错误类型解析

React Native 应用的错误主要分为两大类：JavaScript 层错误与原生层错误，二者在触发场景、表现形式及处理方式上存在显著差异。

### （一）JavaScript 层错误

这类错误发生在 React Native 的 JS 运行时（如 Hermes 或 JSC），多由代码逻辑缺陷导致，常见场景包括：变量未定义、函数调用方式错误、数组越界、异步操作异常等。

#### 关键特征

* 开发环境下：会触发 **RedBox（红色错误提示框）**，显示错误信息、文件路径及堆栈跟踪，直接阻断应用运行。
* 生产环境下：默认不会显示错误提示，若未处理会导致应用白屏、功能失效，严重时引发 JS 线程阻塞。
* 可通过 React 生态工具或 JS 原生 API 捕获。

#### 常见示例

```
// 1. 变量未定义错误
function greet() {
  console.log(name); // name 未声明，触发 ReferenceError
}

// 2. 异步操作错误（未捕获的 Promise 拒绝）
const fetchData = async () => {
  const response = await fetch('https://api.example.com/data');
  const data = await response.json();
  return data.user.name; // 若 data.user 为 undefined，触发 TypeError
};
fetchData(); // 未添加 catch 处理，导致未捕获 Promise 错误
```

### （二）原生层错误

这类错误发生在 iOS 或 Android 的原生代码中，常见于自定义原生模块、第三方原生库兼容性问题、原生 API 调用不当等场景，例如：iOS 中数组越界、Android 中空指针异常、原生模块向 JS 传递非法数据等。

#### 关键特征

* 开发/生产环境下：通常直接导致应用 **崩溃**，并在原生日志（Xcode 控制台、Android Logcat）中输出堆栈跟踪。
* 难以通过 JS 层直接捕获，需借助原生错误处理机制或跨层通信工具。
* 影响范围更广，可能破坏应用进程稳定性，甚至导致用户无法重启应用。

#### 常见示例

1. **iOS 原生错误（Swift）**：

```
// 自定义原生模块中数组越界
@objc func getRandomItem(_ callback: RCTResponseSenderBlock) {
  let items = ["a", "b"]
  let randomIndex = 3 // 超出数组长度（0-1）
  let item = items[randomIndex] // 触发 IndexOutOfRangeException，导致应用崩溃
  callback([nil, item])
}
```

1. **Android 原生错误（Kotlin）**：

```
// 原生模块中空指针异常
@ReactMethod
fun showToast(message: String) {
  val toast = Toast.makeText(null, message, Toast.LENGTH_SHORT) // context 为 null，触发 NullPointerException
  toast.show()
}
```

![image](https://img2024.cnblogs.com/blog/139239/202510/139239-20251030095457136-1603898235.png)

层级结构：应用层 → JavaScript 层（RedBox/白屏）、原生层（iOS/Android 崩溃）→ 底层运行时（Hermes/JSC、原生系统 API）

## 二、核心错误处理工具与 API

针对不同类型的错误，React Native 提供了多层次的处理工具——从 React 内置的错误边界，到 JS 运行时 API，再到原生层的崩溃捕获机制。

### （一）JavaScript 层核心处理工具

#### 1. Error Boundaries（错误边界）

Error Boundaries 是 React 16+ 引入的官方错误捕获机制，专门用于捕获子组件树中的 JS 错误（包括渲染错误、生命周期方法错误），并展示降级 UI，避免整个组件树崩溃。**注意**：它无法捕获异步操作（如 setTimeout、Promise）、事件处理器中的错误及服务器端渲染错误。

#### 实现方式

需创建一个类组件，实现 `getDerivedStateFromError`（更新错误状态）和 `componentDidCatch`（日志上报）两个生命周期方法：

```
import React, { Component } from 'react';

class ErrorBoundary extends Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  // 静态方法：捕获错误并更新组件状态，用于渲染降级 UI
  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  // 实例方法：捕获错误信息，可用于日志上报
  componentDidCatch(error, errorInfo) {
    // 上报错误到监控平台（如 Sentry）
    console.error('Error Boundary 捕获错误：', error, errorInfo.componentStack);
  }

  render() {
    if (this.state.hasError) {
      // 降级 UI：向用户展示友好提示
      return (
        <div style={{ padding: 20, textAlign: 'center' }}>
          <h2>页面加载出错了h2>
          <p>{this.state.error?.message}p>
          <button onClick={() => this.setState({ hasError: false })}>
            刷新页面
          button>
        div>
      );
    }
    // 无错误时，渲染子组件树
    return this.props.children;
  }
}

// 使用方式：包裹可能出错的组件
export default function App() {
  return (
    <ErrorBoundary>
      <MainComponent /> {/* 可能触发 JS 错误的核心组件 */}
    ErrorBoundary>
  );
}
```

#### 2. React Native Error Utils

`ErrorUtils` 是 React Native 内置的 JS 错误捕获工具，可全局监听未被错误边界捕获的 JS 错误（包括异步操作错误），相当于 JS 层的“最后一道防线”。

#### 使用方式

```
import { ErrorUtils } from 'react-native';

// 保存原始错误处理函数（可选，便于后续恢复默认行为）
const originalErrorHandler = ErrorUtils.getGlobalHandler();

// 自定义全局错误处理函数
const customErrorHandler = (error, isFatal) => {
  // isFatal：布尔值，标识错误是否致命（可能导致应用崩溃）
  console.error(`全局捕获 JS 错误（${isFatal ? '致命' : '非致命'}）：`, error);
  
  // 上报错误信息（如错误消息、堆栈跟踪、设备信息）
  reportErrorToMonitor({
    message: error.message,
    stack: error.stack,
    isFatal,
    platform: Platform.OS,
  });

  // 若需要保留默认行为（如开发环境显示 RedBox），可调用原始处理函数
  originalErrorHandler(error, isFatal);
};

// 注册全局错误处理函数
ErrorUtils.setGlobalHandler(customErrorHandler);
```

#### 3. Promise 错误捕获

React Native 中未捕获的 Promise 拒绝（如未添加 `catch` 的异步请求）会触发警告（开发环境）或静默失败（生产环境），需通过以下方式统一处理：

```
// 监听未捕获的 Promise 拒绝
if (YellowBox) {
  // 开发环境：屏蔽特定警告（可选）
  YellowBox.ignoreWarnings(['Possible Unhandled Promise Rejection']);
}

// 全局捕获未处理的 Promise 错误
process.on('unhandledRejection', (reason, promise) => {
  console.error('未处理的 Promise 错误：', reason, promise);
  // 上报错误信息
  reportErrorToMonitor({
    type: 'UnhandledPromiseRejection',
    message: reason?.message || String(reason),
    stack: reason?.stack,
  });
});
```

### （二）原生层错误处理工具

原生层错误（崩溃）无法通过 JS 工具直接捕获，需分别在 iOS 和 Android 端实现原生错误处理逻辑，或使用第三方监控库简化流程。

#### 1. iOS 原生错误捕获（Swift/Objective-C）

iOS 中可通过 `NSSetUncaughtExceptionHandler` 捕获未处理的异常，通过 `signal` 监听信号量错误（如内存访问错误）：

```
// AppDelegate.swift
import UIKit

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {
  var window: UIWindow?

  func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    // 注册异常捕获处理器
    NSSetUncaughtExceptionHandler { exception in
      let name = exception.name.rawValue
      let reason = exception.reason ?? "未知原因"
      let stackTrace = exception.callStackSymbols.joined(separator: "\n")
      
      // 保存错误日志到本地或上报
      let errorLog = "iOS 崩溃：\n名称：\(name)\n原因：\(reason)\n堆栈：\(stackTrace)"
      print(errorLog)
      // 调用自定义上报方法
      ErrorReporter.shared.report(errorLog: errorLog)
    }

    // 监听信号量错误（如 SIGSEGV、SIGABRT）
    let signals = [SIGABRT, SIGILL, SIGSEGV, SIGFPE, SIGBUS, SIGPIPE]
    for signal in signals {
      signal(signal) { sig in
        let errorLog = "iOS 信号量错误：信号 \(sig)"
        print(errorLog)
        ErrorReporter.shared.report(errorLog: errorLog)
        // 退出应用（避免僵尸进程）
        exit(sig)
      }
    }

    return true
  }
}
```

#### 2. Android 原生错误捕获（Kotlin/Java）

Android 中可通过实现 `Thread.UncaughtExceptionHandler` 捕获线程未处理的异常：

```
// CrashHandler.kt
import android.content.Context
import java.io.PrintWriter
import java.io.StringWriter

class CrashHandler(private val context: Context) : Thread.UncaughtExceptionHandler {
  // 保存默认异常处理器
  private val defaultHandler = Thread.getDefaultUncaughtExceptionHandler()

  override fun uncaughtException(t: Thread, e: Throwable) {
    // 收集错误信息
    val errorLog = StringBuilder()
    errorLog.append("Android 崩溃：\n线程：${t.name}\n")
    // 获取堆栈跟踪
    val sw = StringWriter()
    val pw = PrintWriter(sw)
    e.printStackTrace(pw)
    errorLog.append("堆栈：${sw.toString()}")

    // 保存日志或上报
    print(errorLog.toString())
    ErrorReporter.report(context, errorLog.toString())

    // 调用默认处理器（触发系统崩溃提示）
    defaultHandler?.uncaughtException(t, e)
  }

  companion object {
    // 在 Application 中初始化
    fun init(context: Context) {
      Thread.setDefaultUncaughtExceptionHandler(CrashHandler(context))
    }
  }
}

// 初始化（在自定义 Application 类中）
class MyApp : Application() {
  override fun onCreate() {
    super.onCreate()
    CrashHandler.init(this)
  }
}
```

## 三、实战场景：关键业务错误处理方案

结合 React Native 开发中的高频场景，以下是针对性的错误处理实践方案，涵盖网络请求、异步操作、原生模块调用等核心环节。

### （一）网络请求错误处理

网络请求是错误高发场景，需处理请求失败、响应异常、数据解析错误等问题，建议封装统一的请求工具：

```
import axios from 'axios';
import { Alert } from 'react-native';

// 创建 axios 实例
const api = axios.create({
  baseURL: 'https://api.example.com',
  timeout: 10000,
});

// 请求拦截器：添加请求头（如 Token）
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// 响应拦截器：统一处理错误
api.interceptors.response.use(
  (response) => response.data, // 成功时直接返回数据
  (error) => {
    let errorMessage = '网络请求失败，请稍后重试';
    
    // 分类处理错误
    if (error.response) {
      // 服务器返回错误（4xx/5xx）
      const status = error.response.status;
      const data = error.response.data;
      errorMessage = data?.message || `请求错误（${status}）`;
      
      // 特殊状态码处理（如 401 未授权）
      if (status === 401) {
        // 触发登出逻辑
        logout();
        errorMessage = '登录已过期，请重新登录';
      }
    } else if (error.request) {
      // 无响应（网络错误、超时）
      errorMessage = error.code === 'ECONNABORTED' ? '请求超时' : '网络异常，请检查网络连接';
    } else {
      // 请求配置错误（如参数错误）
      errorMessage = `请求配置错误：${error.message}`;
    }

    // 上报错误信息
    reportErrorToMonitor({
      type: 'NetworkError',
      message: errorMessage,
      stack: error.stack,
      requestConfig: error.config,
    });

    // 向用户展示错误提示
    Alert.alert('提示', errorMessage);
    
    return Promise.reject(error);
  }
);

// 使用示例：获取用户数据
const fetchUser = async (userId) => {
  try {
    const data = await api.get(`/users/${userId}`);
    return data;
  } catch (error) {
    // 业务层可额外处理（如重试、降级）
    console.error('获取用户数据失败：', error);
    throw error; // 向上传递错误，供组件处理
  }
};
```

### （二）原生模块调用错误处理

React Native 调用自定义原生模块时，需处理参数校验、原生逻辑异常等问题，建议通过 Promise 封装原生方法，便于捕获错误：

#### 1. 原生模块封装（以 Android 为例）

```
// CustomModule.kt
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.bridge.ReactContextBaseJavaModule
import com.facebook.react.bridge.ReactMethod
import com.facebook.react.bridge.Promise

class CustomModule(reactContext: ReactApplicationContext) : ReactContextBaseJavaModule(reactContext) {
  override fun getName() = "CustomModule"

  // 用 Promise 封装原生方法，便于 JS 捕获错误
  @ReactMethod
  fun processData(input: String, promise: Promise) {
    try {
      // 校验参数
      if (input.isEmpty()) {
        throw IllegalArgumentException("输入参数不能为空")
      }
      // 业务逻辑
      val result = "处理后的结果：$input"
      promise.resolve(result) // 成功回调
    } catch (e: Exception) {
      // 错误回调：传递错误信息到 JS 层
      promise.reject("PROCESS_ERROR", e.message, e)
    }
  }
}
```

#### 2. JS 层调用与错误处理

```
import { NativeModules } from 'react-native';
const { CustomModule } = NativeModules;

// 调用原生模块方法
const processNativeData = async (input) => {
  try {
    const result = await CustomModule.processData(input);
    return result;
  } catch (error) {
    // 捕获原生模块抛出的错误
    console.error('原生模块调用失败：', error.code, error.message);
    // 上报错误
    reportErrorToMonitor({
      type: 'NativeModuleError',
      module: 'CustomModule',
      method: 'processData',
      code: error.code,
      message: error.message,
    });
    // 向用户提示
    Alert.alert('错误', `处理失败：${error.message}`);
    throw error;
  }
};

// 使用示例
processNativeData('测试输入')
  .then((result) => console.log(result))
  .catch((error) => console.error(error));
```

### （三）异步操作错误处理（如文件读写、存储）

React Native 中的异步操作（如 `AsyncStorage`、文件系统操作）需通过 `try/catch` 捕获错误，并提供降级方案：

```
import AsyncStorage from '@react-native-async-storage/async-storage';
import { Alert } from 'react-native';

// 封装 AsyncStorage 操作，统一处理错误
const StorageService = {
  async setItem(key, value) {
    try {
      const jsonValue = JSON.stringify(value);
      await AsyncStorage.setItem(key, jsonValue);
    } catch (error) {
      console.error(`存储 ${key} 失败：`, error);
      // 上报错误
      reportErrorToMonitor({
        type: 'StorageError',
        operation: 'setItem',
        key,
        message: error.message,
      });
      // 提示用户
      Alert.alert('存储错误', '数据保存失败，请检查存储空间');
      throw error;
    }
  },

  async getItem(key) {
    try {
      const jsonValue = await AsyncStorage.getItem(key);
      return jsonValue != null ? JSON.parse(jsonValue) : null;
    } catch (error) {
      console.error(`获取 ${key} 失败：`, error);
      reportErrorToMonitor({
        type: 'StorageError',
        operation: 'getItem',
        key,
        message: error.message,
      });
      // 降级处理：返回默认值
      return null;
    }
  },
};
```

[图例插入标识：React Native 异步操作错误处理流程示意图] 流程节点：发起异步操作（setItem/getItem）→ try 块执行操作 → 成功：返回结果 / 失败：catch 捕获 → 错误上报 → 用户提示/降级处理

## 四、错误监控与日志上报

仅在应用内处理错误不够，还需建立完善的监控体系，实时收集错误信息，以便定位问题并优化。常用方案分为“自建监控”和“第三方监控”两类。

### （一）第三方监控工具（推荐）

第三方工具已封装好 JS 层与原生层的错误捕获逻辑，支持崩溃分析、用户行为追踪、设备信息收集等功能，主流工具包括：

#### 1. Sentry

* 支持 React Native 全平台错误捕获（JS 错误、原生崩溃）。
* 提供详细的堆栈跟踪、错误上下文（用户信息、设备信息、应用版本）。
* 支持错误分组、告警通知（邮件、Slack）。

#### 集成示例

```
// 安装依赖：npm install @sentry/react-native && npx pod-install ios
import * as Sentry from '@sentry/react-native';

// 初始化 Sentry（在 App 入口处）
Sentry.init({
  dsn: '你的 Sentry DSN',
  environment: __DEV__ ? 'development' : 'production',
  tracesSampleRate: 1.0, // 性能监控采样率
});

// 手动上报错误（可选）
try {
  // 可能出错的逻辑
} catch (error) {
  Sentry.captureException(error, {
    extra: { customInfo: '额外上下文信息' },
    tags: { module: 'user', action: 'login' },
  });
}
```

#### 2. Bugsnag

* 专注于移动应用崩溃监控，支持 React Native 双端原生崩溃捕获。
* 提供错误优先级分级、用户会话跟踪、版本趋势分析。

#### 3. Firebase Crashlytics

* 与 Firebase 生态集成，适合使用 Firebase 的项目。
* 免费版功能足够满足中小项目需求，支持崩溃统计与过滤。

### （二）自建监控系统

若需定制化监控逻辑，可通过以下方式实现：

1. **日志收集**：在 JS 层和原生层捕获错误后，将错误日志（含错误信息、堆栈、设备信息、用户 ID）保存到本地。
2. **日志上报**：在应用下次启动时，检查本地日志，将未上报的错误通过网络请求发送到自建服务器。
3. **后台管理**：搭建后台系统，对错误日志进行分类、统计、搜索，设置告警规则（如某类错误发生率超过 5% 时触发告警）。

## 五、错误处理最佳实践

### （一）开发阶段最佳实践

1. **启用严格模式**：在 `App.js` 中启用 React 严格模式，提前发现潜在问题：

```
import { StrictMode } from 'react';
import { AppRegistry } from 'react-native';
import App from './App';
import { name as appName } from './app.json';

AppRegistry.registerComponent(appName, () => () => <StrictMode><App />StrictMode>);
```

1. **禁用生产环境的 RedBox/YellowBox**：避免向用户暴露错误细节，保护敏感信息。
2. **编写错误处理测试**：使用 Jest 测试错误边界、异常捕获逻辑，确保其能正常工作。

### （二）生产阶段最佳实践

1. **避免静默失败**：所有异步操作、原生模块调用必须添加错误处理，禁止忽略 `catch` 块。
2. **提供友好的用户提示**：避免向用户展示技术术语（如“NullPointerException”），用通俗语言说明问题（如“数据加载失败，请检查网络”）。
3. **实现错误降级**：核心功能出错时，提供替代方案（如网络请求失败时展示缓存数据）。
4. **定期分析错误日志**：优先修复高频错误、严重崩溃（如启动时崩溃），持续优化应用稳定性。
5. **版本控制错误处理逻辑**：记录错误处理代码的变更，便于回溯问题。

### （三）跨平台兼容性注意事项

1. **原生模块错误适配**：针对 iOS 和 Android 原生模块的差异，分别处理平台专属错误（如 iOS 的权限错误、Android 的存储权限错误）。
2. **Hermes 引擎兼容**：使用 Hermes 引擎时，部分 JS 错误的堆栈跟踪格式会变化，需确保监控工具支持 Hermes 日志解析。
3. **第三方库版本控制**：避免因第三方原生库版本更新导致的兼容性崩溃，建议锁定关键依赖版本。

## 六、结语

React Native 错误处理的核心在于“分层捕获、全面监控、友好降级”——JS 层通过错误边界和全局处理器覆盖逻辑错误，原生层通过平台专属机制捕获崩溃，再结合监控工具实现问题的实时感知与快速定位。

开发者需根据应用场景选择合适的处理方案：小型应用可使用 React 内置工具+简易日志上报；中大型应用建议集成 Sentry 等第三方监控工具，同时定制化错误处理逻辑。通过系统化的错误处理策略，能显著提升应用稳定性，改善用户体验，降低运维成本。

本博客参考[wgetcloud](https://yiang.org)。转载请注明出处！
