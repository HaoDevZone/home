Android开发最新架构Jetpack以及Compose使用指南

### Jetpack应用架构指南

Android developer 地址：
https://developer.android.com/topic/architecture?hl=zh-cn

如果您不应使用应用组件存储应用数据和状态，那么您应该改为如何设计应用呢？

随着 Android 应用大小不断增加，您定义的架构务必要能允许应用扩缩、提升应用的稳健性并且方便对应用进行测试。

应用架构定义了应用的各个部分之间的界限以及每个部分应承担的职责。为了满足上述需求，您应该按照某些特定原则设计应用架构。

#### 单向数据流

单一数据源原则常常与单向数据流 (UDF) 模式一起使用。在 UDF 中，状态仅朝一个方向流动。修改数据的事件朝相反方向流动。

![架构图](https://developer.android.com/static/topic/libraries/architecture/images/mad-arch-overview.png)

### 现代应用架构

此现代应用架构鼓励采用以下方法及其他一些方法：

- 反应式分层架构。
- 应用的所有层中的单向数据流 (UDF)。
- 包含状态容器的界面层，用于管理界面的复杂性。
- 协程和数据流。
- 依赖项注入最佳实践。

