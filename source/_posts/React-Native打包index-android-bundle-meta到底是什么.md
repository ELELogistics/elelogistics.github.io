---
title: React-Native打包index.android.bundle.meta到底是什么?
date: 2016-09-19 14:11:26
author : 进击的小羊
tags: React-Native
---
在React-native升级到0.30的时候，打包bundle，发现多了一个文件index.android.bundle.meta，不知道是做什么用的，文档里面也没有找到相关的解释，打开发现就是一串编码过后的字符串,从meta的意思：变化来看，感觉是类似md5的文件标识,但是不能确定。

<!-- more -->

问了很多大神，都没有答案，所以试着自己去探索一下。

因为之前解析过React-native 源码，所以我事先知道rn的打包脚本在react-native/node_modules/react-native/local-cli 这个文件下,这个文件夹下面的文件有

     __mocks__             generate-android.js   rnpm
    __tests__             generator             runAndroid
    bundle                generator-android     runIOS
    cli.js                generator-ios         server
    cliEntry.js           generator-utils.js    setup_env.bat
    commands.js           init                  setup_env.sh
    default.config.js     library               upgrade
    dependencies          logAndroid            util
    generate              logIOS                wrong-react-native.js

其中runAndroid runIOS是我们经常用到的命令，再看我们的打包命令：

    react-native bundle --platform android  --dev false --entry-file index.android.js \
      --bundle-output ../app/src/main/assets/index.android.bundle \
      --assets-dest ../app/src/main/res/

bundle也是个文件夹，打包相关的命令应该就在他下面了

进去react-native/node_modules/react-native/local-cli/bundle/bundle.js 

```
const buildBundle = require('./buildBundle');
const outputBundle = require('./output/bundle');
const outputPrepack = require('./output/prepack');
const bundleCommandLineArgs = require('./bundleCommandLineArgs');

function bundleWithOutput(argv, config, args, output, packagerInstance) {
  if (!output) {
    output = args.prepack ? outputPrepack : outputBundle;
  }
  return buildBundle(args, config, output, packagerInstance);
}

function bundle(argv, config, args, packagerInstance) {
  return bundleWithOutput(argv, config, args, undefined, packagerInstance);
}

module.exports = {
  name: 'bundle',
  description: 'builds the javascript bundle for offline use',
  func: bundle,
  options: bundleCommandLineArgs,

  // not used by the CLI itself
  withOutput: bundleWithOutput,
};
```

恩，js代码能看懂，bundle-》bundleWithOutput-》buildBundle,方法链跳到了buildBundle.js中

```
function buildBundle(args, config, output = outputBundle, packagerInstance) {
	...
  const bundlePromise = output.build(packagerInstance, requestOpts)
    .then(bundle => {
      if (shouldClosePackager) {
        packagerInstance.end();
      }
      return saveBundle(output, bundle, args);
    });

  // Save the assets of the bundle
  const assets = bundlePromise
    .then(bundle => bundle.getAssets())
    .then(outputAssets => saveAssets(
      outputAssets,
      args.platform,
      args.assetsDest,
    ));

  // When we're done saving bundle output and the assets, we're done.
  return assets;
}
```

这里有俩个promise,一个是build&save 还有一个saveAssets,看到这个就知道我们应该找对了，bundle文件是保存在assets下面的，这里我们的目的是看meta文件是怎么生成的，所以忽略其他细节，有兴趣的朋友自己去查看，代码很简单。

继续./output/bundle.build 方法下面

```
function saveBundleAndMap(bundle, options, log) {
  const {
    bundleOutput,
    bundleEncoding: encoding,
    dev,
    sourcemapOutput
  } = options;

  log('start');
  const codeWithMap = createCodeWithMap(bundle, dev);
  log('finish');

  log('Writing bundle output to:', bundleOutput);

  const {code} = codeWithMap;
  const writeBundle = writeFile(bundleOutput, code, encoding);
  const writeMetadata = writeFile(
    bundleOutput + '.meta',
    meta(code, encoding),
    'binary');
  Promise.all([writeBundle, writeMetadata])
    .then(() => log('Done writing bundle output'));

  if (sourcemapOutput) {
    log('Writing sourcemap output to:', sourcemapOutput);
    const writeMap = writeFile(sourcemapOutput, codeWithMap.map, null);
    writeMap.then(() => log('Done writing sourcemap output'));
    return Promise.all([writeBundle, writeMetadata, writeMap]);
  } else {
    return writeBundle;
  }
}
```

眼尖的同学已经发现了.meta生成的代码，writeFile

```
function writeFile(file, data, encoding) {
  return new Promise((resolve, reject) => {
    fs.writeFile(
      file,
      data,
      encoding,
      error => error ? reject(error) : resolve()
    );
  });
}
```
将data用encoding编码写到file文件中，

      const writeMetadata = writeFile(
        bundleOutput + '.meta',
        meta(code, encoding),
        'binary');


- 这里的文件名就是:index.android.bundle.meta
- data是 meta(code, encoding)
- encoding 是binary 二进制

忽略code是什么，先看一下meta方法做了什么

```
module.exports = function(code, encoding) {
  const hash = crypto.createHash('sha1');
  hash.update(code, encoding);
  const digest = hash.digest('binary');
  const signature = Buffer(digest.length + 1);
  signature.write(digest, 'binary');
  signature.writeUInt8(
    constantFor(tryAsciiPromotion(code, encoding)),
    signature.length - 1);
  return signature;
};
```
先算出code的sha1，然后用binary编码

虽然和我们之前想的md5算法有差别，但是作用都是一样的，

再回去找一下code是什么，一番跳转，发现code就是bundle.getSource({dev})，也就是bundle中的内容

验证一下我们的想法：

==
发现用同样的源码打包俩次，获取的meta相同，如果改动一个字符，打出的meta就会改变==

**结论：**

index.android.bundle.meta中存储的是bundle的sha1值，每次打包都会生成一个meta唯一标识bundle，之后的代码中并没有实际作用，可以删除。


[原文地址](http://xujinyang.github.io/2016/09/19/React-Native%E6%89%93%E5%8C%85index-android-bundle-meta%E5%88%B0%E5%BA%95%E6%98%AF%E4%BB%80%E4%B9%88/)

