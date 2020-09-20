# Flutter 常用包

以下所有包可在[Flutter官方包管理网站](https://pub.flutter-io.cn/)查找

### 打开第三方应用：url_launcher

### 从图像库中拾取图像，并使用相机拍摄新照片：image_picker

### 透明图片：transparent_image

### 缓存网络图片：cached_network_image

### 轮播图：flutter_swiper

### http请求：http、Dio

### json序列化反序列化：json_serializable

**Json 转为 Dart bean**

1. 以下面的json文件为例

   ```json
   {
     "searchUrl": "ssss"
   }
   ```

2. 在 `pubspec.yaml` 中引入依赖

   ```yaml
   dependencies:
     json_annotation: ^3.0.1
   
   dev_dependencies:
     json_serializable: ^3.2.5
     build_runner: ^1.7.4
   ```
   
3. 创建下面的 `class`

   ```dart
   import 'package:json_annotation/json_annotation.dart';
   
   /// 生成 ConfigModel 对应转换类的文件名称
   part 'config_model.g.dart';
   
   @JsonSerializable()
   class ConfigModel {
   
     @JsonKey(name: 'searchUrl')
     final String searchUrl;
   
     ConfigModel(this.searchUrl);
   
     factory ConfigModel.fromJson(Map<String, dynamic> json) =>
         _$ConfigModelFromJson(json);
   
     Map<String, dynamic> toJson() => _$ConfigModelToJson(this);
   }
   ```

3. 在项目路径执行下面命令，自动生成对应Dart文件

   ```bash
   $ flutter packages pub run build_runner build
   ```

### 本地异步持久化数据：shared_preferences

### Html 文档解析：html

### Flutter 和 H5 混合开发组件：flutter_webview_plugin

### 跨组件状态共享：Provider

### 访问设备文件系统上的常用位置：path_provider

该类当前支持访问两个文件系统位置：

- **临时目录:** 可以使用 `getTemporaryDirectory()` 来获取临时目录； 系统可随时清除的临时目录（缓存）。在iOS上，这对应于[`NSTemporaryDirectory()`](https://developer.apple.com/reference/foundation/1409211-nstemporarydirectory) 返回的值。在Android上，这是[`getCacheDir()`](https://developer.android.com/reference/android/content/Context.html#getCacheDir())返回的值。
- **文档目录:** 可以使用`getApplicationDocumentsDirectory()`来获取应用程序的文档目录，该目录用于存储只有自己可以访问的文件。只有当应用程序被卸载时，系统才会清除该目录。在iOS上，这对应于`NSDocumentDirectory`。在Android上，这是`AppData`目录。
- **外部存储目录**：可以使用`getExternalStorageDirectory()`来获取外部存储目录，如SD卡；由于iOS不支持外部目录，所以在iOS下调用该方法会抛出`UnsupportedError`异常，而在Android下结果是android SDK中`getExternalStorageDirectory`的返回值

### 连接到WebSocket服务器: web_socket_channel

### 国际化：flutter_localizations

### 下拉刷新与上拉加载更多：flutter_easyrefresh

### 富文本编辑器
zefyr

### 瀑布流布局
flutter_staggered_grid_view