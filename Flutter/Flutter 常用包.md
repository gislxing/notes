# Flutter 常用包

以下所有包可在[Flutter官方包管理网站](https://pub.flutter-io.cn/)查找

1. 打开第三方应用：url_launcher

2. 从图像库中拾取图像，并使用相机拍摄新照片：image_picker

3. 透明图片：transparent_image

4. 缓存网络图片：cached_network_image

5. 轮播图：flutter_swiper

6. http请求：http、Dio

7. json序列化：json_serializable

8. 本地异步持久化数据：shared_preferences

9. Html 文档解析：html

10. Flutter 和 H5 混合开发组件：flutter_webview_plugin

11. 跨组件状态共享：Provider

12. 访问设备文件系统上的常用位置：path_provider

    该类当前支持访问两个文件系统位置：

    - **临时目录:** 可以使用 `getTemporaryDirectory()` 来获取临时目录； 系统可随时清除的临时目录（缓存）。在iOS上，这对应于[`NSTemporaryDirectory()`](https://developer.apple.com/reference/foundation/1409211-nstemporarydirectory) 返回的值。在Android上，这是[`getCacheDir()`](https://developer.android.com/reference/android/content/Context.html#getCacheDir())返回的值。
    - **文档目录:** 可以使用`getApplicationDocumentsDirectory()`来获取应用程序的文档目录，该目录用于存储只有自己可以访问的文件。只有当应用程序被卸载时，系统才会清除该目录。在iOS上，这对应于`NSDocumentDirectory`。在Android上，这是`AppData`目录。
    - **外部存储目录**：可以使用`getExternalStorageDirectory()`来获取外部存储目录，如SD卡；由于iOS不支持外部目录，所以在iOS下调用该方法会抛出`UnsupportedError`异常，而在Android下结果是android SDK中`getExternalStorageDirectory`的返回值

13. 连接到WebSocket服务器: web_socket_channel

14. 国际化：flutter_localizations

15. 