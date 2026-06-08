# CLAUDE.md - component-jetpack-mvvm

## 项目描述

基于 WanAndroid API 的 Android 客户端，采用组件化 + Jetpack + Kotlin + MVVM 架构。项目展示了 Android 现代开发的最佳实践，包括组件化拆分、Jetpack 全家桶集成、协程网络请求封装等。

- 包名：`com.fuusy.jetpackkt`
- API 地址：`https://www.wanandroid.com`
- 作者：fuusy

## 技术栈

| 类别 | 技术 | 版本 |
|------|------|------|
| 语言 | Kotlin | 1.4.10 |
| 构建 | Gradle (AGP) | 4.0.0 |
| 最低 SDK | minSdk 16, targetSdk 30 | - |
| 网络 | OkHttp + Retrofit + 协程 | 4.9.0 / 2.9.0 |
| 数据库 | Room + Paging3 | 2.3.0-alpha01 / 3.0.0-beta01 |
| 导航 | Navigation | 2.3.5 |
| 依赖注入 | Koin | 2.2.0-rc-3 |
| 路由 | ARouter | 1.5.0 |
| 图片加载 | Coil | 1.2.1 |
| 状态管理 | LoadSir | 1.3.8 |
| 数据绑定 | DataBinding | - |
| 序列化 | Gson | 2.8.6 |
| 存储 | MMKV | 1.2.1 |

## 项目结构

```
component-jetpack-mvvm/
├── app/                  # 主模块，Application、MainActivity、SplashActivity
├── common/               # 公共基础模块（Base类、网络封装、工具类、自定义View）
├── service/              # 服务层模块（目前为 library，存放公共 Service 接口）
├── home/                 # 首页模块（文章列表、每日一问、广场、Banner）
├── project/              # 项目模块（项目分类、项目列表）
├── personal/             # 个人中心模块
├── login/                # 登录注册模块
├── webview/              # WebView 模块
├── build.gradle          # 根构建脚本
├── dependencies.gradle   # 统一依赖版本管理
├── settings.gradle       # 模块注册
└── gradle.properties     # 全局配置（含 singleModule 标志位）
```

## 模块职责

### common 模块
基础层，所有业务模块依赖此模块。包含：
- `base/` - BaseActivity、BaseFragment、BaseViewModel、BaseRepository、BaseVmActivity、BasePagingAdapter
- `network/` - RetrofitManager（Retrofit 单例）、BaseResp、ResState、StateLiveData、IStateObserver、LogInterceptor、LocalCookieJar
- `loadsir/` - LoadSir 状态回调（LoadingCallback、ErrorCallback、EmptyCallback）
- `support/` - Constants、SingleLiveData、StatusBar、NavigationExtensions、BnvMediator、NetStateHelper
- `widget/` - 自定义 View（FSmartRefreshLayout、LoadingDialog、ItemSettingsView 等）
- `utils/` - AppHelper、AppUtil、SpUtils、ToastUtil

### 业务模块（home / project / personal / login / webview）
每个业务模块遵循统一结构：
- `api/` - Retrofit Service 接口定义
- `bean/` - 数据实体类
- `repo/` - Repository 层，负责数据获取（网络 + 本地）
- `viewmodel/` - ViewModel 层
- `ui/` - Fragment / Activity
- `adapter/` - RecyclerView Adapter（含 PagingAdapter）
- `di/` - Koin 依赖注入模块定义

## 构建说明

### 正常构建（全部模块作为 library 编入主 App）
```bash
./gradlew assembleDebug
```
默认 `gradle.properties` 中 `singleModule=false`，所有子模块以 library 形式编译。

### 子模块独立运行
1. 修改 `gradle.properties` 中 `singleModule=true`
2. 同步 Gradle 后直接运行目标模块

原理：子模块 `build.gradle` 通过 `singleModule` 标志位切换 `com.android.application` 和 `com.android.library` 插件，同时通过 `sourceSets` 切换不同的 AndroidManifest.xml（library 模式使用 `src/main/manifest/AndroidManifest.xml` 空壳清单，避免多图标问题）。

### 统一依赖管理
`dependencies.gradle` 定义了所有模块共享的通用依赖，各模块通过 `apply from:'../dependencies.gradle'` 引入。

## 组件化架构说明

### 模块划分策略
- **app** - 壳工程，负责组装所有业务模块，管理 Application 初始化
- **common** - 基础库，不包含业务逻辑
- **service** - 接口服务层，用于跨模块暴露服务
- **业务模块** - 相互独立，通过 ARouter 进行页面跳转和通信

### 跨模块通信
使用阿里 ARouter 路由框架，通过 URL 路由实现模块间页面跳转，解耦模块间依赖。每个模块的 `build.gradle` 中配置 `AROUTER_MODULE_NAME` 参数。

### 依赖注入
使用 Koin 框架，每个业务模块在 `di/` 目录下定义自己的 Koin Module（如 `moduleHome`、`moduleLogin`），在 MainApp 中统一加载。

## MVVM 架构说明

### 分层结构
```
UI (Fragment/Activity)
  ↓ 观察 LiveData
ViewModel
  ↓ 调用 Repository
Repository
  ↓ 通过 Retrofit Service 请求数据
Network / Database
```

### BaseViewModel
提供 `loadingLiveData` 和 `errorLiveData` 用于 UI 层统一监听加载和错误状态。内置 `launch` 方法封装协程启动。

### BaseRepository
提供两种网络请求封装方式：
1. `executeResp()` - 直接协程调用，通过 StateLiveData 分发状态
2. `executeReqWithFlow()` - 基于 Flow 的请求封装，支持 onStart/onEmpty/onCompletion 等状态

### 网络状态管理
- `DataState` 枚举：STATE_LOADING / STATE_SUCCESS / STATE_EMPTY / STATE_FAILED / STATE_ERROR
- `StateLiveData` - 带状态的 LiveData，UI 层通过 `IStateObserver` 统一处理各状态
- 结合 LoadSir 进行界面状态切换（加载中/成功/失败/空数据）

### 数据绑定
使用 Android DataBinding，BaseVmActivity 通过泛型 `T : ViewDataBinding` 自动绑定布局。

### Paging3 分页
- 纯网络分页：自定义 PagingSource（如 DailyQuestionPagingSource）
- Room 缓存分页：RemoteMediator + Room + PagingSource（如 ArticleRemoteMediator）

## 关键注意事项

1. DataBinding 已启用，布局文件需使用 `<layout>` 标签包裹
2. ARouter 注解处理器（kapt）在每个需要路由的模块中必须配置
3. Room 数据库定义在 home 模块的 `dao/db/AppDatabase` 中，版本号为 1
4. Navigation 使用多 navigation 方案（navi_home / navi_project / navi_personal），配合自定义 NavigationExtensions 实现 BottomNavigationView 的 Fragment 栈管理
5. Retrofit 的 baseUrl 为 `https://www.wanandroid.com`，在 RetrofitManager 中硬编码
6. 子模块独立运行时，需确保该模块有自己的 Launcher Activity 声明在主 AndroidManifest.xml 中
