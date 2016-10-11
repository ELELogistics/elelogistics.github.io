---
title: React-Native 源码分析二-JSX如何渲染成原生页面(上)
date: 2016-09-12 18:04:17
author : 进击的小羊
tags: React-Native
---
本文跳过了React-Native 的通讯过程，详细请参考大头鬼写的[Java和JS的通讯原理](https://zhuanlan.zhihu.com/p/20464825)，虽然0.33版本加入了懒加载，原来配置表生成的时机和方式发生了改变,但是原理还是没有改变：通过约定的JSON，解析出moduleName,function name，然后通过本地找到对应的模块中的方法，然后通过反射执行这些方法，实现调用。

<!-- more -->

这篇将从Android原生反推JSX如何最终变化为原生控件的过程。

博主使用的环境是（版本很重要，RN发展飞快，不同的版本之间可能有差别）

>  "react": "15.3.1",

>  "react-native": "^0.33.0",

在[React-Native 源码分析一-如何启动JS页面](http://xujinyang.github.io/2016/09/09/React-Native-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)的最后一步，我们看到XReactInstanceManagerImpl.java的attachMeasuredRootViewToInstance方法中有设置View的逻辑

```
  private void attachMeasuredRootViewToInstance(
	...
    UIManagerModule uiManagerModule = catalystInstance.getNativeModule(UIManagerModule.class);
    int rootTag = uiManagerModule.addMeasuredRootView(rootView);
    rootView.setRootViewTag(rootTag);
    ...
  }

```
可以看到uiManagerModule.addMeasuredRootView(rootView)这个方法好像很厉害的样子，进去看看
```
public int addMeasuredRootView(final SizeMonitoringFrameLayout rootView) {
	//去掉了宽高赋值的代码
	
    mUIImplementation.registerRootView(rootView, tag, width, height, themedRootContext);
	//忽略了setOnSizeChangedListener
 	
    return tag;
  }
```

去掉无关代码之后，可以看到 mUIImplementation.registerRootView(rootView, tag, width, height, themedRootContext)方法传递了view和相关宽，高，theme信息进去，进去看代码发现利用这些数据构造了一个ReactShadowNode，然后add到了mOperationsQueue中，一看到Queue立马想到肯定有个UI相关的轮循在处理UI绘制事务。
```
public void registerRootView(
      SizeMonitoringFrameLayout rootView,
      int tag,
      int width,
      int height,
      ThemedReactContext context) {
    final ReactShadowNode rootCSSNode = createRootShadowNode();
    rootCSSNode.setReactTag(tag);
    rootCSSNode.setThemedContext(context);
    rootCSSNode.setStyleWidth(width);
    rootCSSNode.setStyleHeight(height);

    mShadowNodeRegistry.addRootNode(rootCSSNode);

    // register it within NativeViewHierarchyManager
    mOperationsQueue.addRootView(tag, rootView, context);
  }

```

所以我们先放下这里，回到上一个方法addMeasuredRootView的注释
```
/**
   * Registers a new root view. JS can use the returned tag with manageChildren to add/remove
   * children to this view.
   *
   * Note that this must be called after getWidth()/getHeight() actually return something. See
   * CatalystApplicationFragment as an example.
   *
   * TODO(6242243): Make addMeasuredRootView thread safe
   * NB: this method is horribly not-thread-safe.
   */
```
js能根据tag，使用manageChildren 来添加，删除 rootview中的子view

那么可以猜想manageChildren 可能是js直接控制原生代码增删布局的入口，来看下
```
@ReactMethod
  public void manageChildren(
      int viewTag,
      @Nullable ReadableArray moveFrom,
      @Nullable ReadableArray moveTo,
      @Nullable ReadableArray addChildTags,
      @Nullable ReadableArray addAtIndices,
      @Nullable ReadableArray removeFrom) {

    mUIImplementation.manageChildren(
        viewTag,
        moveFrom,
        moveTo,
        addChildTags,
        addAtIndices,
        removeFrom);
  }
```
果然这是个用ReactMethod注解过的方法，代表这他要被JS直接调用,从注释:Interface for adding/removing/moving views within a parent view from JS也能知道js通过这个方法增删改view，同样有@ReactMethod注解的类还有：createView，removeRootView，updateView，setChildren，replaceExistingNonRootView，removeSubviewsFromContainerWithID，measure，measureInWindow。。。等等方法，随便找了一个方法看一下，比如createView

```
   @ReactMethod
  public void createView(int tag, String className, int rootViewTag, ReadableMap props) {
    if (DEBUG) {
      FLog.d(
          ReactConstants.TAG,
          "(UIManager.createView) tag: " + tag + ", class: " + className + ", props: " + props);
    }
    mUIImplementation.createView(tag, className, rootViewTag, props);
  }
```
继续进mUIImplementation.createView，

```
 public void createView(int tag, String className, int rootViewTag, ReadableMap props) {
    ReactShadowNode cssNode = createShadowNode(className);
    ReactShadowNode rootNode = mShadowNodeRegistry.getNode(rootViewTag);
    cssNode.setReactTag(tag);
    cssNode.setViewClassName(className);
    cssNode.setRootNode(rootNode);
    cssNode.setThemedContext(rootNode.getThemedContext());

    mShadowNodeRegistry.addNode(cssNode);

    ReactStylesDiffMap styles = null;
    if (props != null) {
      styles = new ReactStylesDiffMap(props);
      cssNode.updateProperties(styles);
    }

    handleCreateView(cssNode, rootViewTag, styles);
  }
```
构造一个ReactShadowNode，其中createShadowNode 是通过className 找到之前注册的ViewManager比如ReactTextInputManager,再设置他的rootNode，最后handleCreateView

```
  protected void handleCreateView(
      ReactShadowNode cssNode,
      int rootViewTag,
      @Nullable ReactStylesDiffMap styles) {
    if (!cssNode.isVirtual()) {
      mNativeViewHierarchyOptimizer.handleCreateView(cssNode, cssNode.getThemedContext(), styles);
    }
  }

  public void handleCreateView(
      ReactShadowNode node,
      ThemedReactContext themedContext,
      @Nullable ReactStylesDiffMap initialProps) {
    if (!ENABLED) {
      int tag = node.getReactTag();
      mUIViewOperationQueue.enqueueCreateView(
          themedContext,
          tag,
          node.getViewClass(),
          initialProps);
      return;
    }
  }

public void enqueueCreateView(
      ThemedReactContext themedContext,
      int viewReactTag,
      String viewClassName,
      @Nullable ReactStylesDiffMap initialProps) {
    synchronized (mNonBatchedOperationsLock) {
      mNonBatchedOperations.addLast(
        new CreateViewOperation(
          themedContext,
          viewReactTag,
          viewClassName,
          initialProps));
    }
  }
```
这样一路跟下去，我们会发现，如果要createView的一个View，最后只是在ArrayDeque<UIOperation> mNonBatchedOperations中add了一个CreateViewOperation()，很敏感的会发现UIOperation 是抽象的接口
```
public interface UIOperation {
    void execute();
  }
```
果然只有一个接口execute，那自然的还有很多实现了UIOperation的类比如：RemoveRootViewOperation，ChangeJSResponderOperation，ShowPopupMenuOperation等等，
之前我们好像隐约的感觉到有个UI轮询在不停的执行这些UIOperation，也就是业务方只需要往池子里面添加就行，这样的队列在Android很多系统中都有遇到，比如Handle还有EventBus，有兴趣的读者可以看一下我之前的[一个总结](http://xujinyang.github.io/2016/03/26/%E9%98%9F%E5%88%97%E5%9C%A8Android%E4%B8%AD%E7%9A%84%E4%BD%BF%E7%94%A8/)
这个类的名字com/facebook/react/uimanager/UIViewOperationQueue.java 所以大胆的在里面找轮训的代码，很快我们发现了dispatchViewUpdates方法
```
 void dispatchViewUpdates(final int batchId) {
				...
                 if (nonBatchedOperations != null) {
                   for (UIOperation op : nonBatchedOperations) {
                     op.execute();
                   }
                 }

                 ...
           });
    }
```
在一个线程数组中添加了一个线程，专门for循环调用各自的execute()方法，这里举个例子CreateViewOperation

```
private final class CreateViewOperation extends ViewOperation {

    private final ThemedReactContext mThemedContext;
    private final String mClassName;
    private final @Nullable ReactStylesDiffMap mInitialProps;

    public CreateViewOperation(
        ThemedReactContext themedContext,
        int tag,
        String className,
        @Nullable ReactStylesDiffMap initialProps) {
      super(tag);
      mThemedContext = themedContext;
      mClassName = className;
      mInitialProps = initialProps;
      Systrace.startAsyncFlow(Systrace.TRACE_TAG_REACT_VIEW, "createView", mTag);
    }

    @Override
    public void execute() {
      Systrace.endAsyncFlow(Systrace.TRACE_TAG_REACT_VIEW, "createView", mTag);
      mNativeViewHierarchyManager.createView(
          mThemedContext,
          mTag,
          mClassName,
          mInitialProps);
    }
  }
```
执行execute方法也就是 执行mNativeViewHierarchyManager.createView
```
public void createView(
      ThemedReactContext themedContext,
      int tag,
      String className,
      @Nullable ReactStylesDiffMap initialProps) {
 		...
    try {
      ViewManager viewManager = mViewManagers.get(className);

      View view = viewManager.createView(themedContext, mJSResponderHandler);
      mTagsToViews.put(tag, view);
      mTagsToViewManagers.put(tag, viewManager);

      view.setId(tag);
      if (initialProps != null) {
        viewManager.updateProperties(view, initialProps);
      }
    } finally {
      Systrace.endSection(Systrace.TRACE_TAG_REACT_VIEW);
    }
  }

```
这里的mViewManagers.get(className) 是根据className找到之前MainReactPackage里面添加的各种ViewManagers，然后调用ViewManager的createView方法，因为ViewManager是父类，他的createView里调用抽象方法createViewInstance,看下面代码

```
 public final T createView(
      ThemedReactContext reactContext,
      JSResponderHandler jsResponderHandler) {
    T view = createViewInstance(reactContext);
    addEventEmitters(reactContext, view);
    if (view instanceof ReactInterceptingViewGroup) {
      ((ReactInterceptingViewGroup) view).setOnInterceptTouchEventListener(jsResponderHandler);
    }
    return view;
  }

 protected abstract T createViewInstance(ThemedReactContext reactContext);

```

createViewInstance 抽象方法是每个子类必须要实现的方法，也是正在构造View的方法，还是举个例子：ReactTextInputManager

```
public class ReactTextInputManager extends BaseViewManager<ReactEditText, LayoutShadowNode> {

  /* package */ static final String REACT_CLASS = "AndroidTextInput";


  @Override
  public String getName() {
    return REACT_CLASS;
  }

  @Override
  public ReactEditText createViewInstance(ThemedReactContext context) {
    ReactEditText editText = new ReactEditText(context);
    int inputType = editText.getInputType();
    editText.setInputType(inputType & (~InputType.TYPE_TEXT_FLAG_MULTI_LINE));
    editText.setImeOptions(EditorInfo.IME_ACTION_DONE);
    editText.setTextSize(
        TypedValue.COMPLEX_UNIT_PX,
        (int) Math.ceil(PixelUtil.toPixelFromSP(ViewDefaults.FONT_SIZE_SP)));
    return editText;
  }
}
```
他的createViewInstance方法就是new ReactEditText(context)，到这里一个View已经创建完成，那么他的属性在哪里设置？放心JS已经将生成一个View要的数据都带了回来，initialProps中就是jsx中的style，viewManager.updateProperties(view, initialProps);再下面就是解析，设置属性，然后在在rootView中测量大小，确定位置，原生的UI渲染就完成了，期间细节太繁琐，不容易都写出来，只是描述一个流程，如果真正了解绘制细节的，还有好几个重要的类需要慢慢解析，请需求的同学自行解读。

前文中这次会反推JSX如何最终变化为原生控件的过程，上面这部分算是原生的绘制已经结束，下面开始到JS代码中找，JSX布局如何传达到原生的。


[原文地址](http://xujinyang.github.io/2016/09/12/React-Native-jsx%20analyse1/)



