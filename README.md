# 鸿蒙应用分享-懂车帝

分享人：陈庆伟

---

## 业务&技术实践

懂车帝应用内路由演变

---

### 鸿蒙路由方案能力对比

| 业务场景                       | Navigation                            | Router                                 |
| ------------------------------ | ------------------------------------- | -------------------------------------- |
| 一多能力                       | 支持，Auto模式自适应单栏跟双栏显示    | 不支持                                 |
| 跳转指定页面                   | pushPath & pushDestination            | pushUrl & pushNameRoute                |
| 获取指定页面参数               | 支持                                  | 不支持                                 |
| 传参类型                       | 传参为对象形式                        | 传参为对象形式，对象中暂不支持方法变量 |
| 清理指定路由                   | removeByIndexes & removeByName        | 不支持                                 |
| 自定义转场动画                 | 支持                                  | 支持，动画类型受限                     |
| geometryTransition共享元素动画 | 支持（NavDestination之间共享）        | 不支持                                 |
| 页面生命周期监听               | UIObserver.on('navDestinationUpdate') | UIObserver.on('routerPageUpdate')      |
| 获取页面栈对象                 | 支持                                  | 不支持                                 |
| 路由拦截                       | 支持通过setInercption做路由拦截       | 不支持                                 |
| 路由栈信息查询                 | 支持                                  | getState() & getLength()               |
| 路由栈move操作                 | moveToTop & moveIndexToTop            | 不支持                                 |
| 沉浸式页面                     | 支持                                  | 不支持，需通过window配置               |

Note:

---

### 最初方案

手写 import 语句

```typescript [2-4]
// 差异配置
import(
  "../../../../../../business/main_frame/carpick/src/main/ets/featureconfig/CarDiffEntry"
);
// 条件选车
import(
  "../../../../../../business/main_frame/carpick/src/main/ets/carfilter/CarFilterEntry"
);
...
```

--

调用 push 方法

```typescript
router.pushNamedRoute({ name: option.name, params: option.param });
```

Note:

---

### 生成 import 语句

通过 hvigor task 在编译时生成代码

```typescript
// 生成route for navigation
pluginContext.registerTask({
  name: "generateNavigationRoutes",
  run: (taskContext) => generateNavigationRoutes(project.getNodeDir()),
  dependencies: [],
  postDependencies: ["default@PreBuild"],
});
```

--

```typescript [4]
// AUTO-GENERATED FILE. DO NOT MODIFY!
function importAll() {
  // routeName: about_dcd
  import('../../../../../../business/content/profile/src/main/ets/settings/AboutDCDPage')
  // routeName: account_edit
  import('../../../../../../business/content/profile/src/main/ets/settings/useredit/AccountEditPage')
  ...
}
```

Note:

---

### 同一页面内组件之间传递 Show/Hide 事件

```typescript [2]
export class PageEventSource {
  get onPageVisibilityChange(): Listenable<boolean>;

  dispose(): void;
}
```

缺陷：Consume不支持可空的/本地初始化，导致预加载时没有 Provide 发生运行时异常。

issue：[go/detail/240829163137107](https://issuereporter.developer.huawei.com/detail/240829163137107/comment)

Note: 为新需求设计的组件

---

### 支持任意组件拦截返回事件(Part I)

```typescript
type OnBackPressedCallback = () => boolean;

export class OnBackPressedDispatcher {
  addCallback(callback: OnBackPressedCallback);

  removeCallback(callback: OnBackPressedCallback);

  takePriority(callback: OnBackPressedCallback);

  dispatch(): boolean;

  dispose(): void;
}
```

Note:

---

### 支持任意组件拦截返回事件(Part II)

```typescript [2]
export class PageEventSource {
  get onBackPressedDispatcher(): OnBackPressedDispatcher;

  get onPageVisibilityChange(): Listenable<boolean>;

  dispose(): void;
}
```

Note:

---

### 支持任意组件拦截返回事件(Part III)

这是到目前为止我们 Entry 组件的样子：

```typescript [4-5|7-9|11-13|15-17|19-20]
@Entry({ routeName: 'ugc_detail' })
@Component
export struct UgcVideoPage {
  @Provide
  eventSource: PageEventSource = new PageEventSource()

  onPageShow(): void {
    this.eventSource.onPageVisibilityChange.notifyListeners(true)
  }

  onPageHide(): void {
    this.eventSource.onPageVisibilityChange.notifyListeners(false)
  }

  onBackPress(): boolean | void {
    return this.eventSource.onBackPressedDispatcher.dispatch()
  }

  aboutToDisappear(): void {
    this.eventSource.dispose()
  }

}
```

必须要添加 eventSource，手动重写 onPageShow、onPageHide、onBackPress，以及
aboutToDisappear，否则会导致逻辑错误。

Note:

---

### 迁移 Navigation

#### 背景

Router路由后续将不做演进，系统路由推荐从Router切换到Navigation。

同时由于Router的不支持 Dialog 路由，无法通过 schema 弹起
dialog，因此我们也希望能通过Navigation实现Dialog路由功能。

--

#### 方案：包装现有 Entry 组件

切换到 Navigation 会导致原本的 onPageShow、onPageHide、onBackPress，这三个

API 不再生效，但由于我们从一开始就是用来 PageEventSource

在组件间直接传递页面事件，所以这里只需要在下面的 Wrapper 中利用 Navigation 新的

API将事件转发到PageEventSource，其余组件均无需改动。

--

#### Demo

```typescript
// 以下代码可以通过编译期生成

@Builder
export function SeriesDetailEntryNavDestinationBuilder(params: object) {
  NavDestination() {
    SeriesDetailEntry({
      args: params as Record<string, string>,
    })
  }.hideTitleBar(true)
    .onShown(() => {
      // TODO：分发事件
    })
    .onHidden(() => {
      // TODO：分发事件
    })
    .onBackPressed(() => {
      // TODO：分发事件
      return false
    })
}

const builder: WrappedBuilder<[object]> = wrapBuilder(SeriesDetailEntryNavDestinationBuilder);
AutoNavigation.addBuilder("concern", builder)
```

Note:

---

### 依旧使用功能代码生成方案

```typescript [20-24]
// AUTO-GENERATED FILE. DO NOT MODIFY!

@Component
export struct AboutDCDPageProvide {
  @Prop params?: object = undefined;
  @Provide({ allowOverride: "eventSource" }) eventSource: PageEventSource = new PageEventSource("about_dcd", this.params)

  build() {
    PageWrapper({
      destinationMode: NavDestinationMode.STANDARD,
      params: this.params,
      routerName: "about_dcd",
    }) {
      AboutDCDPage()
    }
  }
}

@Builder
function AboutDCDPageNavDestinationBuilder(context: NavigationContext) {
  AboutDCDPageProvide({params: context.getParams()})
}

AutoNavigation.addBuilder("about_dcd", wrapBuilder(AboutDCDPageNavDestinationBuilder))
```

理论上我们只需要生成每个页面的 Builder 以及 addBuilder 调用。

为什么这里还需要 PageProvide 组件？ 这同样是 JS 的一个限制：

[尾随闭包情况下provide未定义错误](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-provide-and-consume-0000001820879589#ZH-CN_TOPIC_0000001857876921__builderparam%E5%B0%BE%E9%9A%8F%E9%97%AD%E5%8C%85%E6%83%85%E5%86%B5%E4%B8%8Bprovide%E6%9C%AA%E5%AE%9A%E4%B9%89%E9%94%99%E8%AF%AF)

Note:

---

### 在PageWrapper 组件内对每个页面公共逻辑进行统一处理

```typescript [10-13|15-21|23-26|29-54|55-77]
@Component
export struct PageWrapper {
  @BuilderParam pageBuilder: () => void;
  @Prop destinationMode: NavDestinationMode = NavDestinationMode.STANDARD;
  @Consume private eventSource: PageEventSource
  @Prop params?: object = undefined;
  @Prop routerName?: string = undefined;
  @State longTakeSession: LongTakeSession = new LongTakeSession()

  aboutToAppear(): void {
    GlobalPageEventRegistry.onEvent(this.eventSource.pageUniqueId, NavigationEvent.ABOUT_APPEAR)
    AutoContinue.createData(this.eventSource.pageUniqueId)
  }

  onBackPress(): boolean {
    const result = this.eventSource.onBackPressedDispatcher.dispatch();
    if (!result) {
      AutoNavigation.setResult(undefined)
    }
    return result;
  }

  aboutToDisappear(): void {
    this.eventSource.dispose()
    AutoContinue.destroyData(this.eventSource.pageUniqueId)
    GlobalPageEventRegistry.onEvent(this.eventSource.pageUniqueId, NavigationEvent.ABOUT_DISAPPEAR)
  }

  build() {
    NavDestination() {
      LongTakeTransitionDelegate({
        longTakeSession: this.longTakeSession
      }) {
        this.pageBuilder()
      }
    }
    .hideTitleBar(true)
    .mode(this.destinationMode)
    .onWillAppear(() => {
      GlobalPageEventRegistry.onEvent(this.eventSource.pageUniqueId, NavigationEvent.WILL_APPEAR)
    })
    .onAppear(()=>{
      GlobalPageEventRegistry.onEvent(this.eventSource.pageUniqueId, NavigationEvent.APPEAR)
    })
    .onWillShow(() => {
      this.triggerPrePageEventFromDialog(NavigationEvent.WILL_HIDE);
      GlobalPageEventRegistry.onEvent(this.eventSource.pageUniqueId, NavigationEvent.WILL_SHOW)
    })
    .onShown(() => {
      this.eventSource.onPageVisibilityChange.notifyListeners(true)
      AutoContinue.activeData(this.eventSource.pageUniqueId)
      this.triggerPrePageEventFromDialog(NavigationEvent.PAGE_HIDE);
      GlobalPageEventRegistry.onEvent(this.eventSource.pageUniqueId, NavigationEvent.PAGE_SHOW)
    })
    .onWillHide(() => {
      GlobalPageEventRegistry.onEvent(this.eventSource.pageUniqueId, NavigationEvent.WILL_HIDE)
      this.triggerPrePageEventFromDialog(NavigationEvent.WILL_SHOW);
    })
    .onHidden(() => {
      this.eventSource.onPageVisibilityChange.notifyListeners(false)
      AutoContinue.inactiveData(this.eventSource.pageUniqueId)
      GlobalPageEventRegistry.onEvent(this.eventSource.pageUniqueId, NavigationEvent.PAGE_HIDE)
      this.triggerPrePageEventFromDialog(NavigationEvent.PAGE_SHOW);
    })
    .onWillDisappear(() => {
      GlobalPageEventRegistry.onEvent(this.eventSource.pageUniqueId, NavigationEvent.WILL_DISAPPEAR)
    })
    .onDisAppear(() => {
      GlobalPageEventRegistry.onEvent(this.eventSource.pageUniqueId, NavigationEvent.DISAPPEAR)
    })
    .onBackPressed(() => {
      return this.onBackPress()
    })
    .onReady((context: NavDestinationContext) => {
      this.longTakeSession.tryCustomNavigation(this.getUIContext(), context.navDestinationId, this.params)
    })
  }

  private triggerPrePageEventFromDialog(event: NavigationEvent) {
    if (this.destinationMode === NavDestinationMode.DIALOG) {
      GlobalPageEventRegistry.onEvent(GlobalPageEventRegistry.getPrePageUniqueId(this.eventSource.pageUniqueId), event)
    }
  }
}
```

由于 ArkTs 构建 UI 的方式主要是通过组合来实现，所以并不能像 Android 中的 Activity 一样，通过继承 BaseActivity 来实现页面公共逻辑；

所以这里我们通过代码生成的方式，让每一个页面都包裹在 PageWrapper 这个组件中，做到了这件事。

Note:

---

### 支持 Dialog

只需要添加一个@Dialog 装饰器

```typescript [1]
@Dialog
@Entry({ routeName: 'my_dialog' })
@Component
export struct MyDialog {

}
```

--

生成的代码只有一行区别

```typescript [5]
@Component
export struct ReportDialogProvide {
  build() {
    PageWrapper({
      destinationMode: NavDestinationMode.DIALOG,
      params: this.params,
      routerName: "report_dialog",
    }) {
      ReportDialog()
    }
  }
}
```

Note:

---

### 更易用的 API

```typescript
export function showDialog(builderParams: DcdDialogBuilderParams) {
  SchemaUtils.push(DCD_DIALOG_CONTAINER_PAGE_URL, builderParams);
}
```

Note:

---

### 如何实现动态路由

将生成的 import 包裹在闭包中，在路由跳转前执行。

```typescript [4-8]
export function registerRoutes(
  routes: Map<string, () => Promise<object | undefined>>,
) {
  routes.set("about_dcd", (): Promise<object> => {
    return import(
      "../../../../../../../business/content/profile/src/main/ets/gen/routes/AboutDCDPageRoute"
    );
  });
  routes.set("account_edit", (): Promise<object> => {
    return import(
      "../../../../../../../business/content/profile/src/main/ets/gen/routes/AccountEditPageRoute"
    );
  });
}
```

--

在路由拦截器中拦截处理。

```typescript [3]
export async function dynamicImportHandler<T>(context: NavigationContext<T>) {
  if (!isHttpUrl(context.option.schema)) {
    await AutoNavigationRegister.dynamicImport(context.option.url.host);
  }
  context.next();
}
```

Note:

---

### 更好的事件传递方案

```typescript [3-6]
export type NavigationEventCallback = (event: NavigationEvent,  pageUniqueId: string) => void;

GlobalPageEventRegistry.registerCallback( /*lifecycleOwner*/ this, (event) => {
  this.onPageEventReceived(event);
}, AutoNavigation.getCurrentContext().option.pageUniqueId);

export enum NavigationEvent {
  ABOUT_APPEAR = "AboutAppear",
  WILL_APPEAR = "WillAppear",
  APPEAR = "Appear",
  WILL_SHOW = "WillShow",
  PAGE_SHOW = "PageShow",
  WILL_HIDE = "WillHide",
  PAGE_HIDE = "PageHide",
  WILL_DISAPPEAR = "WillDisappear",
  DISAPPEAR = "Disappear",
  ABOUT_DISAPPEAR = "AboutDisappear",
}
```

--

和鸿蒙官方新出的方案如出一辙：[observer.on('navDestinationUpdate')](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-arkui-observer-V5#observeronnavdestinationupdate)

```typescript
uiObserver.on("navDestinationUpdate", (info) => {
  console.info("NavDestination state update", JSON.stringify(info));
});
```

解决了预加载时没有 Provide 导致 Crash 的问题。

支持更全面的事件。

与组件解耦，可以代码中任意位置感知页面生命周期。

更好的与组件的生命周期进行绑定，在组件销毁的时候组到自动取消监听。

Note:

---

## 后续规划

路由参数自动解析

类型安全的路由参数传递
