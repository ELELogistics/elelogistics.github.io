---
title: React-Native 源码分析二-JSX如何渲染成原生页面(下)
date: 2016-09-13 11:48:38
author : 进击的小羊
tags: React-Native
---
[ React-Native 源码分析二-JSX如何渲染成原生页面(上)](http://xujinyang.github.io/2016/09/12/React-Native-jsx%20analyse1/)中这次会反推JSX如何最终变化为原生控件的过程，上面这部分算是原生的绘制已经结束，下面开始到JS代码中找，JSX布局如何传达到原生的。

<!-- more -->

经验之谈：要凭借我的半吊子js和C水平要去扒拉React-Native js部分的代码，也是够吃力的，但是我找到了一个很好的工具-webStorm，之前使用sublime text，不能查看类直接的依赖，不能全局查找引用类的地方，在面对几百个类和他们直接错综复杂的关系的时候，着实心累。有了webStom可以直接跳转到引用的类中，如果要查一个类在什么地方用到，可以使用shift+command+F查找到所有的使用到这个字符串的地方,是在陌生领域探索的利器。还有就是在文件夹中全文搜索文件名，也是常用查找方式。

在查看JS代码之前首先要找到一个突破口，因为我的脑子里面一直有个疑问，就是React和React-Native是如何搭配工作的，我们就从这个问题入手开始分析。

先看一下一个很普通的RN页面

```
import React, { Component } from 'react';
import {
  AppRegistry,
  StyleSheet,
  Text,
  View
} from 'react-native';

class TestReact extends Component {
  render() {
    return (
      <View style={styles.container}>
        <Text style={styles.welcome}>
          Welcome to React Native!
        </Text>
      </View>
    );
  }
}
...

AppRegistry.registerComponent('TestReact', () => TestReact);
```
我们发现Component 是解构赋值于react,而Text 来自 react-native 那我们就到react.js中去看一下

> node_modules/react/lib/React.js 


```
'use strict';

var _assign = require('object-assign');

var ReactChildren = require('./ReactChildren');
var ReactComponent = require('./ReactComponent');
var ReactPureComponent = require('./ReactPureComponent');
var ReactClass = require('./ReactClass');
var ReactDOMFactories = require('./ReactDOMFactories');
var ReactElement = require('./ReactElement');
var ReactPropTypes = require('./ReactPropTypes');
var ReactVersion = require('./ReactVersion');
...
var React = {

  // Modern

  Children: {
    map: ReactChildren.map,
    forEach: ReactChildren.forEach,
    count: ReactChildren.count,
    toArray: ReactChildren.toArray,
    only: onlyChild
  },

  Component: ReactComponent,
  PureComponent: ReactPureComponent,

  createElement: createElement,
  cloneElement: cloneElement,
  isValidElement: ReactElement.isValidElement,

  // Classic

  PropTypes: ReactPropTypes,
  createClass: ReactClass.createClass,
  createFactory: createFactory,

module.exports = React;

```
发现React只是引用了ReactComponent，ReactClass等类然后赋值给了自己的变量，想到老的写法中有React.createClass这样来创建组件的，那就到ReactClass中看下

```
var ReactClassInterface = {

  mixins: SpecPolicy.DEFINE_MANY,

  statics: SpecPolicy.DEFINE_MANY,

  propTypes: SpecPolicy.DEFINE_MANY,

  contextTypes: SpecPolicy.DEFINE_MANY,

  childContextTypes: SpecPolicy.DEFINE_MANY,

  getDefaultProps: SpecPolicy.DEFINE_MANY_MERGED,

  getInitialState: SpecPolicy.DEFINE_MANY_MERGED,

  getChildContext: SpecPolicy.DEFINE_MANY_MERGED,

  render: SpecPolicy.DEFINE_ONCE,

  componentWillMount: SpecPolicy.DEFINE_MANY,

  componentDidMount: SpecPolicy.DEFINE_MANY,

  componentWillReceiveProps: SpecPolicy.DEFINE_MANY,
  
  shouldComponentUpdate: SpecPolicy.DEFINE_ONCE,
  
  componentWillUpdate: SpecPolicy.DEFINE_MANY,
 
  componentDidUpdate: SpecPolicy.DEFINE_MANY,
  
  componentWillUnmount: SpecPolicy.DEFINE_MANY,

  updateComponent: SpecPolicy.OVERRIDE_BASE

};
```
很熟悉是不是，这不就是RN的生命周期嘛，找到了生命周期，那么我就看也没有地方实现了这个接口并且调用render方法的，因为Rn是通过render方法来把数据传递到native，控制native渲染UI的。
在ReactClass.js中全局搜索render，并没有发现render的实现，再到ReactComponent.js中搜索也没有发现render的实现，这个时候感觉这样来查找好像大海捞针，我们还没有找到突破口，那我们换个思路，由大到小走不通，那我们就由小到大，由具体到抽象，从一个UI控件的实现来看看也没有什么收获。

随便找个控件RefreshControl

>node_modules/react-native/Libraries/Components/RefreshControl/RefreshControl.js 

```
'use strict';

const ColorPropType = require('ColorPropType');
const NativeMethodsMixin = require('react/lib/NativeMethodsMixin');
const Platform = require('Platform');
const React = require('React');
const View = require('View');

const requireNativeComponent = require('requireNativeComponent');

if (Platform.OS === 'android') {
  var RefreshLayoutConsts = require('UIManager').AndroidSwipeRefreshLayout.Constants;
} else {
  var RefreshLayoutConsts = {SIZE: {}};
}

const RefreshControl = React.createClass({
  statics: {
    SIZE: RefreshLayoutConsts.SIZE,
  },

  mixins: [NativeMethodsMixin],

  propTypes: {
    ...View.propTypes,
    
    onRefresh: React.PropTypes.func,
   
    refreshing: React.PropTypes.bool.isRequired,
    
    tintColor: ColorPropType,
    
    titleColor: ColorPropType,
    
    title: React.PropTypes.string,
    
    enabled: React.PropTypes.bool,
    
    colors: React.PropTypes.arrayOf(ColorPropType),
    
    progressBackgroundColor: ColorPropType,
   
    size: React.PropTypes.oneOf([RefreshLayoutConsts.SIZE.DEFAULT, RefreshLayoutConsts.SIZE.LARGE]),
    
    progressViewOffset: React.PropTypes.number,
  },

  _nativeRef: (null: any),
  _lastNativeRefreshing: false,

  componentDidMount() {
    this._lastNativeRefreshing = this.props.refreshing;
  },

  componentDidUpdate(prevProps: {refreshing: boolean}) {

    if (this.props.refreshing !== prevProps.refreshing) {
      this._lastNativeRefreshing = this.props.refreshing;
    } else if (this.props.refreshing !== this._lastNativeRefreshing) {
      this._nativeRef.setNativeProps({refreshing: this.props.refreshing});
      this._lastNativeRefreshing = this.props.refreshing;
    }
  },

  render() {
    return (
      <NativeRefreshControl
        {...this.props}
        ref={ref => this._nativeRef = ref}
        onRefresh={this._onRefresh}
      />
    );
  },

  _onRefresh() {
    this._lastNativeRefreshing = true;

    this.props.onRefresh && this.props.onRefresh();

    this.forceUpdate();
  },
});

if (Platform.OS === 'ios') {
  var NativeRefreshControl = requireNativeComponent(
    'RCTRefreshControl',
    RefreshControl
  );
} else if (Platform.OS === 'android') {
  var NativeRefreshControl = requireNativeComponent(
    'AndroidSwipeRefreshLayout',
    RefreshControl
  );
}

module.exports = RefreshControl;

```
跳过前面的属性定义，直接来看render是如何渲染控件的

```
render() {
    return (
      <NativeRefreshControl
        {...this.props}
        ref={ref => this._nativeRef = ref}
        onRefresh={this._onRefresh}
      />
    );
  },

```
NativeRefreshControl是ios和Android平台通用的控件，所以有了下面区分平台的兼容代码

```
if (Platform.OS === 'ios') {
  var NativeRefreshControl = requireNativeComponent(
    'RCTRefreshControl',
    RefreshControl
  );
} else if (Platform.OS === 'android') {
  var NativeRefreshControl = requireNativeComponent(
    'AndroidSwipeRefreshLayout',
    RefreshControl
  );
}
```
我们的目光停在了requireNativeComponent这个方法上，在ios平台使用RCTRefreshControl，在Android平台使用AndroidSwipeRefreshLayout，看来就是他来兼容各平台的api。在文件夹内全局搜索
requireNativeComponent.js（这个类不在同级目录，所以不方便找，这个时候就全局搜索）

```
'use strict';

var ReactNativeStyleAttributes = require('ReactNativeStyleAttributes');
var UIManager = require('UIManager');
var UnimplementedView = require('UnimplementedView');

var createReactNativeComponentClass = require('react/lib/createReactNativeComponentClass');
...
import type { ComponentInterface } from 'verifyPropTypes';

function requireNativeComponent(
  viewName: string,
  componentInterface?: ?ComponentInterface,
  extraConfig?: ?{nativeOnly?: Object},
): Function {
  var viewConfig = UIManager[viewName];
  if (!viewConfig || !viewConfig.NativeProps) {
    warning(false, 'Native component for "%s" does not exist', viewName);
    return UnimplementedView;
  }
  var nativeProps = {
    ...UIManager.RCTView.NativeProps,
    ...viewConfig.NativeProps,
  };
  viewConfig.uiViewClassName = viewName;
  viewConfig.validAttributes = {};
  viewConfig.propTypes = componentInterface && componentInterface.propTypes;
  ...
  viewConfig.validAttributes.style = ReactNativeStyleAttributes;

  return createReactNativeComponentClass(viewConfig);
}
```
requireNativeComponent根据前面传过来的viewname,extraConfig，生成了配置变量viewConfig,最后调用createReactNativeComponentClass(viewConfig)

	var createReactNativeComponentClass = require('react/lib/createReactNativeComponentClass');
createReactNativeComponentClass来自react的lib目录下，看到了react有点欣喜，感觉这条路走对了，不废话，继续跟入
>/node_modules/react/lib/createReactNativeComponentClass.js

```
'use strict';

var ReactNativeBaseComponent = require('./ReactNativeBaseComponent');

var createReactNativeComponentClass = function (viewConfig) {
  var Constructor = function (element) {
    this._currentElement = element;
    this._topLevelWrapper = null;
    this._hostParent = null;
    this._hostContainerInfo = null;
    this._rootNodeID = 0;
    this._renderedChildren = null;
  };
  Constructor.displayName = viewConfig.uiViewClassName;
  Constructor.viewConfig = viewConfig;
  Constructor.propTypes = viewConfig.propTypes;
  Constructor.prototype = new ReactNativeBaseComponent(viewConfig);
  Constructor.prototype.constructor = Constructor;

  return Constructor;
};

module.exports = createReactNativeComponentClass;
```
createReactNativeComponentClass方法很简单，返回了一个构造函数，但是我们传入的viewConfig被new 了一个new ReactNativeBaseComponent(viewConfig)

```
'use strict';

var _assign = require('object-assign');

var NativeMethodsMixin = require('./NativeMethodsMixin');
var ReactNativeAttributePayload = require('./ReactNativeAttributePayload');
var ReactNativeComponentTree = require('./ReactNativeComponentTree');
var ReactNativeEventEmitter = require('./ReactNativeEventEmitter');
var ReactNativeTagHandles = require('./ReactNativeTagHandles');
var ReactMultiChild = require('./ReactMultiChild');
var UIManager = require('react-native/lib/UIManager');

var ReactNativeBaseComponent = function (viewConfig) {
  this.viewConfig = viewConfig;
};


ReactNativeBaseComponent.Mixin = {
  getPublicInstance: function () {
    // TODO: This should probably use a composite wrapper
    return this;
  },

  unmountComponent: function () {
    ReactNativeComponentTree.uncacheNode(this);
    deleteAllListeners(this);
    this.unmountChildren();
    this._rootNodeID = 0;
  },

  mountComponent: function (transaction, hostParent, hostContainerInfo, context) {
    var tag = ReactNativeTagHandles.allocateTag();

    this._rootNodeID = tag;
    this._hostParent = hostParent;
    this._hostContainerInfo = hostContainerInfo;

    if (process.env.NODE_ENV !== 'production') {
      for (var key in this.viewConfig.validAttributes) {
        if (this._currentElement.props.hasOwnProperty(key)) {
          deepFreezeAndThrowOnMutationInDev(this._currentElement.props[key]);
        }
      }
    }

    var updatePayload = ReactNativeAttributePayload.create(this._currentElement.props, this.viewConfig.validAttributes);

    var nativeTopRootTag = hostContainerInfo._tag;
    UIManager.createView(tag, this.viewConfig.uiViewClassName, nativeTopRootTag, updatePayload);

    ReactNativeComponentTree.precacheNode(this, tag);

    this._registerListenersUponCreation(this._currentElement.props);
    this.initializeChildren(this._currentElement.props.children, tag, transaction, context);
    return tag;
  }
};

_assign(ReactNativeBaseComponent.prototype, ReactMultiChild.Mixin, ReactNativeBaseComponent.Mixin, NativeMethodsMixin);

module.exports = ReactNativeBaseComponent;
```
进到ReactNativeBaseComponent 里面我们发现了俩个很重要的地方：

1. var UIManager = require('react-native/lib/UIManager');UIManager是JS管理原生UI的的控制类，它的出现代表着这里有人要直接控制原生UI

2. mountComponent: function (transaction, hostParent, hostContainerInfo, context) 基本上就是render的意思,仔细研究一下这个方法

```
mountComponent: function (transaction, hostParent, hostContainerInfo, context) {
    var tag = ReactNativeTagHandles.allocateTag();

    this._rootNodeID = tag;
    this._hostParent = hostParent;
    this._hostContainerInfo = hostContainerInfo;

    if (process.env.NODE_ENV !== 'production') {
      for (var key in this.viewConfig.validAttributes) {
        if (this._currentElement.props.hasOwnProperty(key)) {
          deepFreezeAndThrowOnMutationInDev(this._currentElement.props[key]);
        }
      }
    }

    var updatePayload = ReactNativeAttributePayload.create(this._currentElement.props, this.viewConfig.validAttributes);

    var nativeTopRootTag = hostContainerInfo._tag;
    UIManager.createView(tag, this.viewConfig.uiViewClassName, nativeTopRootTag, updatePayload);

    ReactNativeComponentTree.precacheNode(this, tag);

    this._registerListenersUponCreation(this._currentElement.props);
    this.initializeChildren(this._currentElement.props.children, tag, transaction, context);
    return tag;
  }
};
```

 UIManager.createView(tag, this.viewConfig.uiViewClassName, nativeTopRootTag, updatePayload) 找到了这个方法，就是找到了突破口，刚刚一路跟过来，我们在RefreshControl render方法中发现是new 了一个ReactNativeBaseComponent()，现在发现ReactNativeBaseComponent的mountComponent方法直接就调用了UIManager.createView，这和我们上一篇中讲到的com/facebook/react/uimanager/UIManagerModule.java中的createView方法难道不谋而合？我们直接点UIManager.createView进去看看，发现跳转到了不是UIManager.js 而是react-native/Libraries/ReactNative/UIManagerStatTracker.js这个不知道又是JS什么奇葩的技能导致的。不管了，不懂的东西已经那么多了，不在乎再多一个，直接看
 
 ```
 var UIManager = require('UIManager');

var installed = false;
var UIManagerStatTracker = {
  install: function() {
    if (installed) {
      return;
    }
    installed = true;
    var statLogHandle;
    var stats = {};
    function printStats() {
      console.log({UIManagerStatTracker: stats});
      statLogHandle = null;
    }
    function incStat(key: string, increment: number) {
      stats[key] = (stats[key] || 0) + increment;
      if (!statLogHandle) {
        statLogHandle = setImmediate(printStats);
      }
    }
    var createViewOrig = UIManager.createView;
    UIManager.createView = function(tag, className, rootTag, props) {
      incStat('createView', 1);
      incStat('setProp', Object.keys(props || []).length);
      createViewOrig(tag, className, rootTag, props);
    };
    var updateViewOrig = UIManager.updateView;
    UIManager.updateView = function(tag, className, props) {
      incStat('updateView', 1);
      incStat('setProp', Object.keys(props || []).length);
      updateViewOrig(tag, className, props);
    };
    var manageChildrenOrig = UIManager.manageChildren;
    UIManager.manageChildren = function(tag, moveFrom, moveTo, addTags, addIndices, remove) {
      incStat('manageChildren', 1);
      incStat('move', Object.keys(moveFrom || []).length);
      incStat('remove', Object.keys(remove || []).length);
      manageChildrenOrig(tag, moveFrom, moveTo, addTags, addIndices, remove);
    };
  },
};

module.exports = UIManagerStatTracker;
 ```
有意思的东西出现了：

- UIManager.createView
- UIManager.updateView
- UIManager.manageChildren

这三个方法在UIManagerModule中也出现过

>com/facebook/react/uimanager/UIManagerModule.java

```
public class UIManagerModule extends ReactContextBaseJavaModule implements
    OnBatchCompleteListener, LifecycleEventListener {

...

  @ReactMethod
  public void removeRootView(int rootViewTag) {
    mUIImplementation.removeRootView(rootViewTag);
  }

 
  @ReactMethod
  public void createView(int tag, String className, int rootViewTag, ReadableMap props) {
    if (DEBUG) {
      FLog.d(
          ReactConstants.TAG,
          "(UIManager.createView) tag: " + tag + ", class: " + className + ", props: " + props);
    }
    mUIImplementation.createView(tag, className, rootViewTag, props);
  }

  @ReactMethod
  public void updateView(int tag, String className, ReadableMap props) {
    if (DEBUG) {
      FLog.d(
          ReactConstants.TAG,
          "(UIManager.updateView) tag: " + tag + ", class: " + className + ", props: " + props);
    }
    mUIImplementation.updateView(tag, className, props);
  }

  @ReactMethod
  public void manageChildren(
      int viewTag,
      @Nullable ReadableArray moveFrom,
      @Nullable ReadableArray moveTo,
      @Nullable ReadableArray addChildTags,
      @Nullable ReadableArray addAtIndices,
      @Nullable ReadableArray removeFrom) {
    if (DEBUG) {
      FLog.d(
          ReactConstants.TAG,
          "(UIManager.manageChildren) tag: " + viewTag +
          ", moveFrom: " + moveFrom +
          ", moveTo: " + moveTo +
          ", addTags: " + addChildTags +
          ", atIndices: " + addAtIndices +
          ", removeFrom: " + removeFrom);
    }
    mUIImplementation.manageChildren(
        viewTag,
        moveFrom,
        moveTo,
        addChildTags,
        addAtIndices,
        removeFrom);
  }
	...
}

```

这时候我们可以认为这个地方就是在调用原生的方法在createView或者是创建了createView的配置信息。

分析到这里我们已经有点眉目了，原来Rn和原生一样，也是先渲染内部子控件，然后再渲染外部控件。所以Component来自React的，但是UI控件是React-Native的，在render生命周期执行的时候会执行子控件的render方法，子控件会调用UIManager来把信息传递到原始的UIManagerModule，UIManagerModule根据传过来的Tag找到对应的UIManager，最后生成一个Operation添加到UI处理队列中，当mDispatchUIRunnables执行runable的时候调用Operation.execute抽象方法，其实就是调用UIManager.createViewInstance来真正生成View，然后调用viewManager.updateProperties 设置View的属性。这样一个控件就创建出来了。

最后附上The Life-Cycle of a Composite Component

>react/lib/ReactCompositeComponent.js

    /**
     * ------------------ The Life-Cycle of a Composite Component ------------------
     *
     * - constructor: Initialization of state. The instance is now retained.
     *   - componentWillMount
     *   - render
     *   - [children's constructors]
     *     - [children's componentWillMount and render]
     *     - [children's componentDidMount]
     *     - componentDidMount
     *
     *       Update Phases:
     *       - componentWillReceiveProps (only called if parent updated)
     *       - shouldComponentUpdate
     *         - componentWillUpdate
     *           - render
     *           - [children's constructors or receive props phases]
     *         - componentDidUpdate
     *
     *     - componentWillUnmount
     *     - [children's componentWillUnmount]
     *   - [children destroyed]
     * - (destroyed): The instance is now blank, released by React and ready for GC.
     *
     * -----------------------------------------------------------------------------
     */

[原文地址](http://xujinyang.github.io/2016/09/13/React-Native-jsx%20analyse2/)







