# SU7
偷偷用小米的隐藏浏览器玩小米 Su7的车机

## 功能说明
这个项目旨在扩展小米Su7车机浏览器的功能，实现更多系统交互操作。

### 基础功能
- 🌐 打开外部网站
  - B站空间：[Bilibili](https://space.bilibili.com/40097653)
  - 知识星球：[zsxq](https://wx.zsxq.com/group/15522844582412)

### 系统交互
- 📂 文件管理
  - 打开文件管理器：`content://com.android.externalstorage.documents/root/`
  - 访问下载目录：`content://com.android.externalstorage.documents/document/primary:Download`
  - 访问图片目录：`content://com.android.externalstorage.documents/document/primary:Pictures`

- 📱 系统功能
  - 打开相机：`intent://camera#Intent;scheme=android-app;end`
  - 打开设置：`intent://settings#Intent;scheme=android-app;end`
  - 打开蓝牙设置：`intent://bluetooth#Intent;scheme=android-app;end`
  - 打开WiFi设置：`intent://wifi#Intent;scheme=android-app;end`

### 应用管理
- 📲 应用操作
  - 安装APK：使用`intent://install-package#Intent;scheme=package;end`
  - 卸载应用：使用`intent://uninstall-package#Intent;scheme=package;end`
  - 应用列表：`intent://list#Intent;scheme=android-app;end`

### 开发者选项
- 🛠️ ADB功能
  - 开启USB调试：需要在设置中启用开发者选项
  - ADB无线调试：需要连接同一WiFi网络
  - 端口转发：默认使用5555端口

### 安全提示
⚠️ 注意事项：
1. 使用这些功能可能会影响车机系统的稳定性
2. 某些功能可能需要root权限
3. 请在安全环境下使用这些功能
4. 建议在车辆停止状态下操作

### 使用方法
1. 在车机浏览器中打开此页面
2. 点击对应的链接即可触发相应功能
3. 部分功能可能需要允许权限
4. 如遇到权限提示，请选择"允许"或"始终允许"

### 已知问题
- 部分intent协议可能无法直接使用
- 某些系统功能可能被限制访问
- 不同版本的系统可能表现不同

### 贡献指南
欢迎提交Issue和Pull Request来完善功能！
