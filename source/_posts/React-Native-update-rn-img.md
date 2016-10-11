---
title: React-Native 图片热更新初探
date: 2016-10-09 20:48:23
author : 进击的小羊
tags: React-Native
---

在做React-Native热更新的时候，必不可少的要面对RN中图片的热更新命题，今天我们先来探究一下React-Native中图片是如何加载出来的，搞懂了这个问题，图片的热更新问题也就迎刃而解。

<!-- more -->

博主使用的环境是
>  "react": "15.3.1",

>  "react-native": "^0.33.0",

>   "webstorm" -方便在JS中跳转

#### 分析过程

我们先来看一下在RN中图片是如何设置的

```
import {
	StyleSheet,
	Image,
	View
} from 'react-native';
...
render(){
	return (
  		<View style={{ alignItems: 'center' }}>
			<Image
				style={{ width: 90, height: 90, marginBottom: 20 }}
				source={require('../img/drawer_avatar.png') }
				/>
		  </View>
    )；
}
...
```
可以看出Image控件有一个属性source用于设置图片地址，下面的代码跟踪都是围绕这个属性，因为所谓的图片热更新也就是更换source的地址，当我第一次带着热更新的问题来看这段代码的时候我就产生了一个疑问---"../img/drawer_avatar.png"明明是一个确定的地址，应该如何实现替换尼？这个疑问会在本文的最后解答。

下面我们就到源码中去探究Image控件的实现原理。

> react-native/Libraries/Image/Image.android.js

```
render: function() {
    const source = resolveAssetSource(this.props.source);
    const loadingIndicatorSource = resolveAssetSource(this.props.loadingIndicatorSource);

...

    if (source && (source.uri || Array.isArray(source))) {
      let style;
      let sources;
      //如果存在uri 就设置sources = [{uri: source.uri}];
      if (source.uri) {
        const {width, height} = source;
        style = flattenStyle([{width, height}, styles.base, this.props.style]);
        sources = [{uri: source.uri}];
      } else {
        style = flattenStyle([styles.base, this.props.style]);
        sources = source;
      }
...
      const {onLoadStart, onLoad, onLoadEnd} = this.props;
    	//合并属性，这个很简单
      const nativeProps = merge(this.props, {
        style,
        shouldNotifyLoadEvents: !!(onLoadStart || onLoad || onLoadEnd),
        src: sources,
        loadingIndicatorSrc: loadingIndicatorSource ? loadingIndicatorSource.uri : null,
      });

        return (
         ...
       	
          return <RKImage {...nativeProps}/>;
    }
    return null;
  }
});

var RKImage = requireNativeComponent('RCTImageView', Image, cfg);

```
简化后的render，可以看出，前面一大串对source的处理加工生成一个nativeProps对象，然后将nativeProps插在了RKImage控件中，而RKImage对应着原生代码中一个叫RCTImageView的控件，原生控件相关的内容请看[RN控件渲染分析的博客中有讲解](http://xujinyang.github.io/2016/09/12/React-Native-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%BA%8C-JSX%E5%A6%82%E4%BD%95%E6%B8%B2%E6%9F%93%E6%88%90%E5%8E%9F%E7%94%9F%E9%A1%B5%E9%9D%A2(%E4%B8%8A)/)

顺其自然，uri属性在nativeProps中，被传递到了原生控件RCTImageView中，那我们就到RCTImageView中去看一下，他怎么处置nativeProps的

> node_modules/react-native/ReactAndroid/src/main/java/com/facebook/react/views/image/ReactImageManager.java

几番源码分析之后，我已经对查找源码位置手到擒来。(其实就是find all)

```

public class ReactImageManager extends SimpleViewManager<ReactImageView> {

  public static final String REACT_CLASS = "RCTImageView";

  @Override
  public String getName() {
    return REACT_CLASS;
  }

  private @Nullable AbstractDraweeControllerBuilder mDraweeControllerBuilder;
  private final @Nullable Object mCallerContext;

  public ReactImageManager(
      AbstractDraweeControllerBuilder draweeControllerBuilder,
      Object callerContext) {
    mDraweeControllerBuilder = draweeControllerBuilder;
    mCallerContext = callerContext;
  }

  public ReactImageManager() {
    // Lazily initialize as FrescoModule have not been initialized yet
    mDraweeControllerBuilder = null;
    mCallerContext = null;
  }

  public AbstractDraweeControllerBuilder getDraweeControllerBuilder() {
    if (mDraweeControllerBuilder == null) {
      mDraweeControllerBuilder = Fresco.newDraweeControllerBuilder();
    }
    return mDraweeControllerBuilder;
  }

  @Override
  public ReactImageView createViewInstance(ThemedReactContext context) {
    return new ReactImageView(
        context,
        getDraweeControllerBuilder(),
        getCallerContext());
  }

  // In JS this is Image.props.source
  @ReactProp(name = "src")
  public void setSource(ReactImageView view, @Nullable ReadableArray sources) {
    view.setSource(sources);
  }
}
```

这是一个很典型的UI控件实现，继承之SimpleViewManager，实现了getName，createViewInstance 等方法，然后提供可以自定义的属性给JS，这里我们只看src，忽略其他属性。

这里还有一个**非常非常非常重要的点**，就是ReactImageView的内部实现其实就是Fresco，当初我看这里的时候，以为是很简单的判断网络url类似的处理，可是啃了半天也没有啃下来，原来这就是fresco类似MCV架构中自定义V的实现过程，关于[Fresco这里分析一个分析专栏](http://www.cnblogs.com/pandapan/p/4644195.html)，用过Fresco的都知道，它支持

- 显示网络图片，
- 本地绝对路径图片，
- 还有resource中的图片id或者图片名称，


 如果ReactImageView中的内容都是显示相关的，那么对我们来说就可以当做一个黑盒，暂时忽略，换个思路，我们在debug过程中看一下刚才的代码跑到
```
@ReactProp(name = "src")
  public void setSource(ReactImageView view, @Nullable ReadableArray sources) {
    view.setSource(sources);
  }
```
的时候 sources中的内容是什么

source.uri=img_drawer_avatar

这就很奇怪了，一个本地路径的图片到了要显示的时候，只传过来了图片名称，没有图片路径，在这个问题上我们不过多的纠结，看过fresco就会知道，它是通过

```
  private int getDrawableResourceByName(String name) {
    return getResources().getIdentifier(
        name,
        "drawable",
        getContext().getPackageName());
  }
```
getIdentifier方法来通过图片名称查找drawable图片的，那么已经指定了drawable，那么就说明，图片的一定存在于res-》drawable-mdpi 这样的文件中，而且是在编译前，不然是找不到的，

由上面的分析，我们得出两个结论

1. ../img/drawer_avatar.png 的图片路径传过来的时候，只剩下了img_drawer_avatar这个文件名
2. 图片一定存在于res目录中
3. 如果要想热更新图片，那一定不是这种图片名称的方式，而是本地绝对路径。

得出了第三个结论，我也感觉有点不可思议，可能是走通了整个过程之后的原因，我已经回想不起原来是思路了，自然的就推理出了这个结论。


看了那么多累了吧，国际惯例，放松一下

![珠珠女神](http://mxycsku.qiniucdn.com/group6/M00/85/31/wKgBjFT4JMGAUaJJAAT3uAxbsng58.jpeg)

歇歇好了再上路，这个时候回过头，到js中去找什么地方把../img/drawer_avatar.png变成img_drawer_avatar的，可是我们刚才已经看过了在Image.android.js中并没有什么特别的，除了下面这个方法

	const source = resolveAssetSource(this.props.source);

这行以下的代码，就是合并属性，返回view，那么我们从这里入手，进去看看

找到resolveAssetSource的源码
> react-native/Libraries/Image/resolveAssetSource.js


```
function resolveAssetSource(source: any): ?ResolvedAssetSource {
  if (typeof source === 'object') {
    return source;
  }

  var asset = AssetRegistry.getAssetByID(source);
  if (!asset) {
    return null;
  }

  const resolver = new AssetSourceResolver(getDevServerURL(), getBundleSourcePath(), asset);
  if (_customSourceTransformer) {
    return _customSourceTransformer(resolver);
  }
  return resolver.defaultAsset();
}

module.exports = resolveAssetSource;
```
这行代码承上启下，说明腹中有乾坤，这次不着急进去，先看下传入的三个参数分别是什么

- getDevServerURL 判断是bundle是来自服务器还是本地文件，如果是本地文件就返回null。
- getBundleSourcePath 这个我们需要看一下源码

```
function getBundleSourcePath(): ?string {
  if (_bundleSourcePath === undefined) {
    const scriptURL = SourceCode.scriptURL;
    if (!scriptURL) {
      // scriptURL is falsy, we have nothing to go on here
      _bundleSourcePath = null;
      return _bundleSourcePath;
    }
    if (scriptURL.startsWith('assets://')) {
      // running from within assets, no offline path to use
      _bundleSourcePath = null;
      return _bundleSourcePath;
    }
    if (scriptURL.startsWith('file://')) {
      // cut off the protocol
      _bundleSourcePath = scriptURL.substring(7, scriptURL.lastIndexOf('/') + 1);
    } else {
      _bundleSourcePath = scriptURL.substring(0, scriptURL.lastIndexOf('/') + 1);
    }
  }
  return _bundleSourcePath;
}

```

我们都知道，RN早就把设置bundle路径的权利交给了用户，在Application中我们可以返回实现了的ReactNativeHost，其中有个方法getJSBundleFile 就是设置bundle路径，如果你没有复写这个方法，那就会使用默认路径：assets://index.android.bundle,如果你设置了路径，比如我们设置应用内路径：data/data/me.ele.crowdsource/files/assets

而上面代码中的SourceCode.scriptURL就是我们设置的bundle路径，如果想求证，可以自行去跟踪getJSBundleFile之后的代码。

再回到上面的代码，如果使用了默认路径，那么返回 _bundleSourcePath = null，如果使用了自定义路径/data/data/me.ele.crowdsource/files/assets/ 经过切割后变成/data/data/me.ele.crowdsource/files/assets/

- asset = AssetRegistry.getAssetByID(source)存放这图片的基本信息
```
    {
                 __packager_asset: true,
                 httpServerLocation: '/assets/img',
                 width: 138,
                 height: 138,
                 scales: [ 1 ],
                 hash: 'dcb77e36443aee806e846f4e81b36bc6',
                 name: 'drawer_avatar',
                 type: 'png'
    }
```
搞清楚了传递的参数，我们再到AssetSourceResolver 中看一下defaultAsset方法

```
constructor(serverUrl: ?string, bundlePath: ?string, asset: PackagerAsset) {
    this.serverUrl = serverUrl;
    this.bundlePath = bundlePath;
    this.asset = asset;
  }

 isLoadedFromServer(): boolean {
    return !!this.serverUrl;
  }

  isLoadedFromFileSystem(): boolean {
    return !!this.bundlePath;
  }
defaultAsset(): ResolvedAssetSource {
	//如果是本地服务器也就是debug模式，返回devserver 类似这样：http://localhost:8081/index.android.bundle?platform=android&dev=true
    if (this.isLoadedFromServer()) {
      return this.assetServerURL();
    }

    if (Platform.OS === 'android') {
  	 //如果是不是默认的assets路径下面
      return this.isLoadedFromFileSystem() ?
        this.drawableFolderInBundle() :
        this.resourceIdentifierWithoutScale();
    } else {
      return this.scaledAssetPathInBundle();
    }
  }

/**
   * If the jsbundle is running from a sideload location, this resolves assets
   * relative to its location
   * E.g. 'file:///sdcard/AwesomeModule/drawable-mdpi/icon.png'
   */
  drawableFolderInBundle(): ResolvedAssetSource {
    const path = this.bundlePath || '';
    return this.fromSource(
      'file://' + path + getAssetPathInDrawableFolder(this.asset)
    );
  }

 /**
   * The default location of assets bundled with the app, located by
   * resource identifier
   * The Android resource system picks the correct scale.
   * E.g. 'assets_awesomemodule_icon'
   */
  resourceIdentifierWithoutScale(): ResolvedAssetSource {
    invariant(Platform.OS === 'android', 'resource identifiers work on Android');
    return this.fromSource(assetPathUtils.getAndroidResourceIdentifier(this.asset));
  }
```

我们重点看一下：

		return this.isLoadedFromFileSystem() ?this.drawableFolderInBundle() :
        this.resourceIdentifierWithoutScale();

如果使用默认的assets路径返回resourceIdentifierWithoutScale，如果是自定义路径返回drawableFolderInBundle

这俩个方法也贴了出来，由注释可以看出resourceIdentifierWithoutScale方法最终会把图片处理成assets_awesomemodule_icon，格式为，图片文件夹+"_"+图片名称，比如../img/drawer_avatar这个图片，会返回img_drawer_avatar,不要问我怎么那么确定的，跟进去看源码
```
	
const assetPathUtils = require('../../local-cli/bundle/assetPathUtils');

function getAndroidResourceIdentifier(asset) {
  var folderPath = getBasePath(asset);
  console.log(folderPath);
  return (folderPath + '/' + asset.name)
    .toLowerCase()
    .replace(/\//g, '_')           // Encode folder structure in file name
    .replace(/([^a-z0-9_])/g, '')  // Remove illegal chars
    .replace(/^assets_/, '');      // Remove "assets_" prefix
}
//去掉/号
function getBasePath(asset) {
  var basePath = asset.httpServerLocation;
  if (basePath[0] === '/') {
    basePath = basePath.substr(1);
  }
  return basePath;
}
```
将所有/变成_，去掉非法字符，去掉assets，这样assets/img/drawer_avatar 就变成了img_drawer_avatar，这时候有同学会有疑问了，我的图片名字明明是drawer_avatar，这里怎么加了img_在前面，这样为什么能访问到图片尼？哈哈，这是因为assetPathUtils的这个方法打包脚本也在用，图片打包的时候，名字也按照相同的逻辑处理成了img_开头的文件，不信自己去看bundle命令之后生成的图片名称

到这里，我们就明白了，在默认asstes路径的情况下，图片必须要打包到res路径下，不然的话，这里只返回一个图片名称，在RCTImageView中在不知道绝对路径的情况下是找不到不在R文件中的图片的。

走通了一种情况，再看自定义路径的情况

```
 /**
   * If the jsbundle is running from a sideload location, this resolves assets
   * relative to its location
   * E.g. 'file:///sdcard/AwesomeModule/drawable-mdpi/icon.png'
   */
  drawableFolderInBundle(): ResolvedAssetSource {
    const path = this.bundlePath || '';
    return this.fromSource(
      'file://' + path + getAssetPathInDrawableFolder(this.asset)
    );
  }
```
同样注释给出了一个示例，因为bundlePath 就是我们设置的bundle路径，比如上面说到的file:///data/data/me.ele.crowdsource/files/assets/+getAssetPathInDrawableFolder(this.asset)


```
/**
 * Returns a path like 'drawable-mdpi/icon.png'
 */
function getAssetPathInDrawableFolder(asset): string {
  var scale = AssetSourceResolver.pickScale(asset.scales, PixelRatio.get());
  var drawbleFolder = assetPathUtils.getAndroidDrawableFolderName(asset, scale);
  var fileName =  assetPathUtils.getAndroidResourceIdentifier(asset);
  return drawbleFolder + '/' + fileName + '.' + asset.type;
}
```
这边就不多分析，如注释所说，返回drawable-mdpi/img_drawer_avatar.png

那么完整的路径就是：file:///data/data/me.ele.crowdsource/files/assets/drawable-mdpi/img_drawer_avatar.png

这样的绝对路径传到RCTImageView中fresco是能获取到正确的图片的。

整个流程走完，我们再来回答上面的问题：

#### 问题

../img/drawer_avatar.png"明明是一个确定的地址，应该如何实现替换尼？

#### 解答
Image 控件resolveAssetSource 方法可以把一个确定路径如../img/drawer_avatar.png的地址，切分成 file://+自定义bundle路径+图片名称的路径，那么如果我们想替换这个图片，只需要替换bundle路径就可以，替换了bundle路径那么bundle也就更着要变，这也就能做到js代码的热更新，再在bundle路径下面放上drawable-mdpi/img_drawer_avatar.png 图片就可以实现图片的热更新了。

#### 总结

回答完上面的问题，我们还要明确一点，使用默认的assets路径是不能实现图片替换的，原因上文已经说过一次，因为bundle路径为assets的时候，resolveAssetSource只会返回一个图片名称，fresco通过系统提供的getIdentifier方法只能查找到R文件中存在的图片，网络更新下来的图片肯定存在于R文件中，所以这样就实现不了热更新了。

说到这里图片热更新的方法也就有了

首先将bundle命令的图片路径输出到assets中

    react-native bundle --platform android  --dev false --entry-file index.android.js \
      --bundle-output ../app/src/main/assets/index.android.bundle \
      --assets-dest ../app/src/main/assets

然后在app启动的时候，将图片文件夹drawable-mdpi和bundle文件都copy到一个sd卡或者应用内目录A中，设置bundle的路径也指向A，如果要替换图片，将新下载的图片也copy到A目录下面的drawable-mdpi文件夹中，亲测可用。

最后不得不感叹，RN的代码博大精深，值得深究！

[原文地址](http://xujinyang.github.io/2016/10/09/React-Native-update_rn_img/)












































