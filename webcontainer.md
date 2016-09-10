# Web Container

### Web Container 部分整体结构示意图
![Web Architecture](http://i4.buimg.com/567571/cca4fa7d826aadca.jpg)

### Web Container Uml 图
![Web uml](http://i1.piimg.com/567571/061e763304d4d2e7.png)

### 项目链接
[http://112.124.41.46/yougoods-android/public-webcontainer](http://112.124.41.46/yougoods-android/public-webcontainer)

* Web Container 大致分为以下几个模块
  * WebContainer 继承自WebView，内含虚拟路由用于Web资源的路由导向
  * TemplateManager 用于管理本地模板，更新删除添加等，封装了对数据库的操作
  * JsBridge, JsApi 用于Java Script 与 Native Java 代码的交互
  * DownloadTask 下载模板借助TemplateManager 更新模板，这是最次要的模块，只是每次进入Activity都得做一次

1. WebContainer的使用
> 使用非常简单， Java 代码如下 读取url即可
```Java
      WebContainer mWebView = new WebContainer(this);
  setContentView(mWebView, new ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT));
  mJsBridge = new Bridge(mWebView);
  TemplateInfo templateInfo = null;
  try {
      templateInfo = TemplateManager.getInstance().getTemplateInfo(detectKey);
  }
  catch (IOException e) {
      e.printStackTrace();
  }
  if(templateInfo != null) {
      String localUrl = "file://" + templateInfo.mIndexHtmlPath;
      mWebView.loadUrl(localUrl);
  }
```
2. TemplateManager 主要接口

| 方法                                                                                  |                                参数                                |
|:--------------------------------------------------------------------------------------|:------------------------------------------------------------------:|
| public TemplateInfo getTemplateInfo(String detectKey)                                 |              detectKey 模板 detectKey   返回模板信息               |
| public void applyPatches(String detectKey, InputStream stream, int version)           | detectKey 模板detect key  stream 模板InputStream  version 模板版本 |
| public void applyPatches(String detectKey, File file, int version)                    |     detectKey 模板detect key  file  H5更新包  version 模板版本     |
| private void addTemplateFile(ZipFile zipFile, ZipEntry entry, File templateDirectory) | zipFile  更新文件包   更新文件ZipEntry  templateDirectory 模板目录 |
| private void applyDeletePatch(File templateDirectory, TemplatePatch patch)            |            templateDirectory 模板目录  patch  更新描述             |
| private void applyMovePatch(File templateDirectory, TemplatePatch patch)              |       templateDirectory 模板目录 patch             更新描述        |
3. JsBridge
获取WebView 实例并调取相应Js方法 Java 代码如下
```Java
/** 添加被Js调用的Java方法 */
    public void registerJava(JavaImplement serverHandler) {
        mJavaProxy.register(serverHandler);
    }

    /** 执行Java调用Js */
    public void invokeJs(String script) {
        mJsProxy.transact(script);
    }

    /** 执行Java调用Js */
    public void invokeJs(String invokeMethod, String paramJsonStr, Java2JsCallback java2JsCallback) {
        TransactInfo transactInfo = TransactInfo.createDirectInvoke(invokeMethod);
        mJsProxy.transact(transactInfo, paramJsonStr, java2JsCallback);
    }

    /** 派发Java调用Js后的回调执行，也就是Js调用了Java，把处理结果给Java */
    public void dispatchJava2JsCallback(TransactInfo transactInfo, String paramJsonStr) {
        mJsProxy.doJava2JsCallback(transactInfo, paramJsonStr);
    }

    /** 派发Js调用Java后的回调执行，也就是Java去调用Js，把处理结果给Js */
    public void dispatchJs2JavaCallback(TransactInfo transactInfo, String paramJsonStr) {
        mJsProxy.transact(transactInfo, paramJsonStr, null);
    }
```
