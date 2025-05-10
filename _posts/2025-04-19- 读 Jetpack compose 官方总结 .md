> Jetpack Compose 是一款适用于 Android 的新式声明式界面工具包。Compose 提供声明性 API，让您可在不以命令方式改变前端视图的情况下呈现应用界面，从而使编写和维护应用界面变得更加容易。


## Compose 的三个阶段

Compose 有 3 个主要阶段：

**组合**：要显示什么样的界面。Compose 运行可组合函数并创建界面说明。    
**布局**：要放置界面的位置。该阶段包含两个步骤：测量和放置。对于布局树中的每个节点，布局元素都会根据 2D
坐标来测量并放置自己及其所有子元素。  
**绘制**：渲染的方式。界面元素会绘制到画布（通常是设备屏幕）中。

![compose 阶段](https://developer.android.com/static/develop/ui/compose/images/compose-phases.png?hl=zh-cn)

上图是 Compose 将数据转换为界面的三个阶段。

## 重组

组合只能通过初始组合生成且只能通过重组进行更新。重组是修改组合的唯一方式。
![recompose](https://developer.android.com/static/develop/ui/compose/images/lifecycle-composition.png?hl=zh-cn)
要点：可组合项的生命周期通过以下事件定义：进入组合，执行 0 次或多次重组，然后退出组合。

重组通常由对 State<T> 对象的更改触发。Compose 会跟踪这些操作，并运行组合中读取该特定 State<T> 的所有可组合项以及这些操作调用的无法跳过的所有可组合项。

在重组期间，如果某些符合条件的可组合函数的输入与上一个组合相比未发生变化，则可以完全跳过其执行。   
可组合函数符合跳过条件，除非：

- 函数的返回值类型不是 Unit
- 函数带有 @NonRestartableComposable 或 @NonSkippableComposable 注解
- 必需参数属于非稳定类型

有一个实验性编译器模式强跳过 这会放宽最后一个要求

为了让某个类型被视为稳定类型，它必须符合以下要求 合同：

- 对于相同的两个实例，其 equals 的结果将始终相同。
- 如果类型的某个公共属性发生变化，组合将收到通知。
- 所有公共属性类型也都是稳定。

此协定包含一些重要的常见类型 Compose 编译器会将其视为稳定版本，尽管它们并未明确显示 使用 @Stable 注解标记为稳定版：

- 所有基元值类型：Boolean、Int、Long、Float、Char 等。
- 字符串
- 所有函数类型 (lambda)

所有这些类型都可以遵循稳定协定，因为它们是不可变的。由于不可变类型绝不会发生变化，它们就永远不必通知组合更改方面的信息，因此遵循该协定就容易得多。

Compose 的 MutableState 类型是一种众所周知稳定但可变的类型。如果 MutableState 中存储了值，状态对象整体会被视为稳定对象，因为
State 的 .value 属性如有任何更改，Compose 就会收到通知。

当作为参数传递到可组合项的所有类型都很稳定时，系统会根据可组合项在界面树中的位置来比较参数值，以确保相等性。如果所有值自上次调用后未发生变化，则会跳过重组。

要点：如果所有输入都稳定并且没有更改，Compose 将跳过重组可组合项。比较使用了 equals 方法。

Compose 仅在可以证明稳定的情况下才会认为类型是稳定的。例如，接口通常被视为不稳定类型，并且具有可变公共属性的类型（实现可能不可变）的类型也被视为不稳定类型。

如果 Compose 无法推断类型是否稳定，但您想强制 Compose 将其视为稳定类型，请使用 @Stable 注解对其进行标记。

```kotlin
// Marking the type as stable to favor skipping and smart recompositions.
@Stable
interface UiState<T : Result<T>> {
    val value: T?
    val exception: Throwable?

    val hasError: Boolean
        get() = exception != null
}
```

在上面的代码段中，由于 UiState 是接口，因此 Compose 通常认为此类型不稳定。通过添加 @Stable 注解，您可以告知 Compose 此类型稳定，让
Compose 优先选择智能重组。这也意味着，如果将该接口用作参数类型，Compose 会将其所有实现视为稳定。

## 状态读取

Compose 会自动跟踪在系统读取该值时正在执行的操作。通过这项跟踪，Compose 能够在状态值更改时重新执行读取程序。

状态通常是使用 mutableStateOf() 创建，然后通过以下两种方式之一进行访问：直接访问 value 属性，或使用 Kotlin 属性委托。     
以下两端代码是等效的：

```kotlin
// State read without property delegate.
val paddingState: MutableState<Dp> = remember { mutableStateOf(8.dp) }
Text(
    text = "Hello",
    modifier = Modifier.padding(paddingState.value)
)
```

```kotlin
// State read with property delegate.
var padding: Dp by remember { mutableStateOf(8.dp) }
Text(
    text = "Hello",
    modifier = Modifier.padding(padding)
)
```

## 分阶段状态读取

1. @Composable 函数或 lambda 代码块中的状态读取会影响组合阶段，并且可能会影响后续阶段。当状态值发生更改时，Recomposer
   会安排重新运行所有要读取相应状态值的可组合函数。
   根据组合结果，Compose 界面会运行布局和绘制阶段。如果内容保持不变，并且大小和布局也未更改，界面可能会跳过这些阶段。
2. 布局阶段包含两个步骤：测量和放置。每个步骤的状态读取都会影响布局阶段，并且可能会影响绘制阶段。当状态值发生更改时，Compose
   界面会安排布局阶段。如果大小或位置发生更改，界面还会运行绘制阶段。
3. 绘制代码期间的状态读取会影响绘制阶段。常见示例包括 Canvas()、Modifier.drawBehind 和
   Modifier.drawWithContent。当状态值发生更改时，Compose 界面只会运行绘制阶段。
   ![](https://developer.android.com/static/develop/ui/compose/images/phases-state-read-draw.svg?hl=zh-cn)

## 优化状态读取

由于 Compose 会执行局部状态读取跟踪，因此我们可以在适当阶段读取每个状态，从而尽可能降低需要执行的工作量。

> Prefer lambda modifiers when using frequently changing state.

1. 尝试将状态读取定位到尽可能靠后的阶段，从而尽可能降低 Compose 需要执行的工作量。 当然，通常情况下，我们绝对有必要在组合阶段读取状态。
2. 即便如此，在某些情况下，我们也可以通过过滤状态更改来尽可能减少重组次数。derivedStateOf：将一个或多个状态对象转换为其他状态。


