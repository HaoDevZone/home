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

### Compose

- LessCode 用更少的代码做更多的事情，并避免整个类的bug，因此代码简单且易于维护
- Intuitive 你只需要描述你的UI，其余的就交给Compose了。当应用状态改变时，你的UI会自动更新
- accelerate development 与所有现有代码兼容，因此您可以随时随地采用。通过实时预览和完整的Android Studio支持快速迭代
- Powerful 创建美丽的应用程序，直接访问Android平台api和内置支持材料设计，黑色主题，动画等

  #### 布局

  - Column 能够将里面的子项按照从上到下的顺序依次排序，verticalArrangement控制垂直，必须指定高度，horizontalAlignment控制水平，必须指定宽度，如果
    指定Size，两者皆可使用，不指定会包裹内容。
  - Row 能够将里面的子项按照从左到右的方向水平排列，其中排列通过Argument控制。
  - Box 能将子项依次堆叠显示，显示在最上面的是最后添加的子项。相当于FrameLayout。
  - Surface 主要负责整个组件的形状（边框、圆角），阴影，背景等等。
  - Spacer 能够提供一个空白的布局。
  - BottomNavigation Material design 提供的底部导航栏布局
  - topAppBar 顶部导航栏
  - ModalBottomSheetLayout 可以在App底部弹出，并且将背景暗化。 
    
   `tips：这里只是简单说明，具体使用可以查看官方文档。`    
  #### modifier
    - 允许你改变 Composable 的尺寸、布局、外观或添加高级交互
    Material Design 是围绕三个元素建立的。颜色（Color）、排版（Typography）、形状（Shape) 。 

  #### 组件 
    - Text
    - Button
    - Image
    - Card
    - Surface
    - Spacer

  #### Compose编程思想

  - 声明式范式的转变
      之前通过实例化Widget树来初始化界面，每个widget都维护自己的内部状态，并且提供getter和setter方法，允许应用逻辑和Widget交互。
  - Compose声明
    widget相对无状态，并且不提供getter和setter方法。实际上，widget 不会以对象形式提供。您可以通过使用不同的参数调用同一可组合函数来更新界面。
  - 重组
    重组是指在输入更改时再次调用可组合函数的过程。当函数的输入更改时，会发生这种情况。当 Compose 根据新输入重组时，它仅调用可能已更改的函数或
    lambda，而跳过其余函数或 lambda。
      通过跳过所有未更改参数的函数或 lambda，Compose 可以高效地重组    
      可组合函数 - MAD 技巧

