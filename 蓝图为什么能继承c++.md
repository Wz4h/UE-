# UE 底层原理：蓝图为什么能够继承 C++ 类？

> 本文从一个常见问题出发，逐步认识 UE 的 `UClass` 类型系统、C++
> 类与蓝图类的关系，以及 `UClass` 的来源和实际用途。
>
> **讲解主线：** 提出问题 → UE 为什么使用 UClass → C++ 如何建立 UClass →
> 蓝图如何继承 → 蓝图资产如何生成类 → `_C` 是什么 → UClass
> 的来源与应用。

------------------------------------------------------------------------

## 一、提出问题：蓝图为什么能够继承 C++ 类？

先从一个非常熟悉的操作开始。

我们在 C++ 中声明一个 Actor：

``` cpp
UCLASS(Blueprintable)
class AMyActor : public AActor
{
    GENERATED_BODY()

public:
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float Health = 100.f;

    UFUNCTION(BlueprintCallable)
    void Attack();
};
```

编译 C++ 项目后，回到 UE 编辑器，右键创建一个 Blueprint Class，选择
`AMyActor` 作为父类，命名为 `BP_Enemy`。

此时我们可以：

-   在蓝图中读取或修改 `Health`；
-   调用暴露出来的 `Attack()`；
-   给蓝图添加 `Mana` 等新变量；
-   添加事件图和函数；
-   把蓝图放进关卡，创建多个实例；
-   甚至再创建一个继承 `BP_Enemy` 的蓝图。

看起来十分自然，但如果从 C++ 角度思考，就会产生一个问题：

**C++ 是编译型语言。一个新的派生类通常需要编写 C++
代码，再由编译器处理继承关系。那么 UE
为什么只需要在编辑器里操作几下，就能产生一个继承 C++ 的新类？**

我们并没有重新调用 C++ 编译器，也没有生成一个新的普通 C++ 派生类源码。

这个问题的关键不在于"蓝图是一种可视化编程语言"，而在于：

**UE 在原生 C++ 类型系统之上，建立了自己的 UObject
类型描述和对象管理机制。**

其中最核心的类型之一，就是 `UClass`。

------------------------------------------------------------------------

## 二、UE 实际如何使用类：引出 UClass

### 2.1 普通 C++ 如何创建对象？

先看普通 C++：

``` cpp
class Animal
{
public:
    int Age = 0;
};

Animal* AnimalPtr = new Animal();
```

编译器在编译时就知道 `Animal`
的类型、对象布局以及构造方式。这里直接通过原生 C++ 类型创建对象。

再考虑一个需求：**如果我们只有一个字符串 `"Animal"`，能不能直接让标准
C++ 根据这个名字创建对应对象？**

标准 C++
并没有内建一套能够仅凭类名字符串就通用地创建任意类对象、枚举其所有成员属性的完整反射系统。我们可以自己注册工厂函数和元数据，但那就需要额外设计一套机制。

UE 恰好需要这样的能力。

### 2.2 UE 为什么需要在运行时认识一个类？

UE 编辑器要在 Details
面板展示对象属性；内容浏览器要认识资产类型；蓝图编辑器要列出可调用函数；关卡需要按指定类型创建
Actor。

这些需求都有一个共同点：

**程序不能只依赖编译时写死的 C++ 类型，还需要在运行时查询类型信息。**

例如：

-   这个对象实际是什么类？
-   它的父类是谁？
-   有哪些反射属性？
-   哪些函数暴露给蓝图？
-   如何根据指定的类创建对象？

因此 UE 为 UObject 类建立了反射类型描述。

### 2.3 UClass 是什么？

`UClass` 本身也是 UE 的一个对象类型，用于描述一个 UObject 类。

例如：

``` cpp
UClass* ActorClass = AActor::StaticClass();
```

这里获得的不是一个 `AActor` 实例，而是**描述 AActor 这个类的 UClass
对象**。

可以把它理解成：

``` text
C++ 原生类型：AActor
        │
        │ UE 的反射注册机制
        ▼
描述这个类型的 UClass 对象
        │
        ├── 父类信息
        ├── 反射属性信息
        ├── 反射函数信息
        └── 类默认对象等信息
```

注意：`UClass` 并不是取代 C++ 类，也不意味着所有 C++
函数调用都要通过反射。

UE 同时使用：

1.  **原生 C++ 机制**：处理原生对象布局、构造、编译后的函数执行等；
2.  **UObject
    类型系统**：提供统一的类描述、反射、动态对象创建及其他引擎能力。

这两套机制是协同工作的。

### 2.4 对象创建为什么与 UClass 有关？

普通 UObject 可以使用 `NewObject` 创建：

``` cpp
UMyObject* Obj = NewObject<UMyObject>();
```

虽然这里没有显式写出 `UClass`，但创建过程会使用
`UMyObject::StaticClass()` 等类信息。

我们也可以动态指定具体类：

``` cpp
// BlueprintClass 是一个有效的 UClass*，
// 并且必须是 UMyObject 的子类。
UMyObject* Obj = NewObject<UMyObject>(
    GetTransientPackage(),
    BlueprintClass
);
```

对于 Actor，通常通过 `UWorld::SpawnActor` 创建：

``` cpp
UClass* ClassToSpawn = AMyActor::StaticClass();

AMyActor* Actor = GetWorld()->SpawnActor<AMyActor>(
    ClassToSpawn
);
```

如果把 `ClassToSpawn` 换成继承 `AMyActor`
的蓝图生成类，同样可以创建对象。

**这就是第一个重要结论：对于 UObject/Actor 的标准创建机制，UE 可以通过
UClass 在运行时决定实际创建哪一种类的对象。**

这里说的是 UObject 体系，不是说 UE 中的所有普通 C++ 对象都必须通过
UClass 创建。

------------------------------------------------------------------------

## 三、C++ 类如何建立对应的 UClass？

到这里会自然出现第二个问题：

**既然 UE 使用 UClass 描述类型，那么 `AMyActor` 对应的 UClass
从哪里来？**

### 3.1 并非所有 C++ 类都有 UClass

例如：

``` cpp
class FMyCalculator
{
public:
    int Add(int A, int B) { return A + B; }
};
```

这是普通 C++ 类，不会因为写在 UE 工程里就自动获得 `UClass`。

而下面的 UObject 派生类可以接入 UE 的反射体系：

``` cpp
UCLASS(Blueprintable)
class AMyActor : public AActor
{
    GENERATED_BODY()
};
```

两者最直观的区别，就是 `UCLASS()`、`GENERATED_BODY()`
及其背后的代码生成与注册机制。

### 3.2 UCLASS、UHT、GENERATED_BODY 各做什么？

**`UCLASS()`**：告诉 UE 的头文件工具，这里声明了一个参与反射系统的
UObject 类；其中的说明符还可以控制蓝图可继承性等行为。

**UHT（Unreal Header Tool）**：在构建过程中处理 UE
的反射声明，并生成相应的辅助 C++ 代码。

**`GENERATED_BODY()`**：将必要的生成声明接入原生 C++
类定义，与生成代码配合完成反射注册等工作。

注意不要说"UCLASS 宏自己在运行时生成 UClass"。宏、UHT、C++
编译和模块加载注册是不同阶段。

### 3.3 简化后的完整流程

``` text
编写 AMyActor.h
       │
       ▼
UHT 处理反射声明
       │
       ▼
生成辅助 C++ 代码
       │
       ▼
C++ 编译器编译原生代码
       │
       ▼
模块加载及反射注册
       │
       ▼
获得 AMyActor 对应的 UClass
```

最终我们可以调用：

``` cpp
UClass* Class = AMyActor::StaticClass();
```

并通过它查询类的父类：

``` cpp
UClass* ParentClass = Class->GetSuperClass();
```

所以：

**C++ 的 UObject 类通过 UHT 生成的辅助代码和 UE 的注册机制，建立对应的
UClass。**

原生类本身仍由 C++ 编译器编译；`UClass` 则提供 UE 所需的类型描述。

------------------------------------------------------------------------

## 四、蓝图为什么能够继承 C++ 类？

现在回到最初的问题。

假设我们已经有：

``` text
AActor
   └── AMyActor
```

这两个原生 UObject 类都拥有对应的 `UClass`。

我们在编辑器中创建一个继承 `AMyActor` 的 `BP_Enemy`。

UE 不需要重新编译一个原生 C++
派生类，而是可以通过蓝图编译器生成一个**蓝图生成类**，将它接入现有的
`UClass` 继承体系。

### 4.1 蓝图继承的是哪一个父类？

概念上可以这样看：

``` text
AActor 对应的 UClass
          │
          ▼
AMyActor 对应的 UClass
          │
          ▼
BP_Enemy_C 对应的 UBlueprintGeneratedClass
```

其中 `UBlueprintGeneratedClass` 是 `UClass` 的子类。

它的父类信息指向 `AMyActor` 对应的 `UClass`，因此 UE
能识别这条继承关系。

例如：

``` cpp
UClass* EnemyClass = /* BP_Enemy_C 对应的类 */;

bool bIsChild = EnemyClass->IsChildOf(
    AMyActor::StaticClass()
);
```

对于正确继承 `AMyActor` 的蓝图类，这个判断会返回 `true`。

### 4.2 这里有一个容易说错的地方

不能简单说：

> 蓝图资产继承了 UClass。

更准确的表述是：

> 蓝图编译后生成的 `UBlueprintGeneratedClass`，是 `UClass`
> 的一种；这个生成类通过 UE 的类型系统，继承原生 C++ 父类对应的 UClass。

这里存在**两种不同的"继承"**：

1.  `UBlueprintGeneratedClass : UClass`：这是引擎源码中元类对象类型的
    C++ 继承关系；
2.  `BP_Enemy_C` 的父类是 `AMyActor`：这是具体游戏类在 UE
    类型系统中的继承关系。

它们不能混为一谈。

### 4.3 为什么继承后能使用 C++ 属性和函数？

仅有父类信息还不够，真正的运行还依赖其他机制。

-   **原生基类布局与实现**：蓝图对象建立在其原生 UObject
    基类之上，继承原生实现。
-   **FProperty**：让 UE 认识反射属性，蓝图也可以添加新的反射属性。
-   **UFunction**：让 UE
    认识可反射调用的函数，并处理蓝图可调用、可重写等规则。
-   **Blueprint VM**：通常执行蓝图图表编译出来的脚本逻辑。

例如，`Health` 之所以能在蓝图中访问，不只是因为有父类关系，还因为它通过
`UPROPERTY(BlueprintReadWrite)` 等声明暴露了相应能力。

**因此，UClass
解决的是统一描述类及其继承关系的核心问题；属性访问和函数执行还需要配套机制。**

------------------------------------------------------------------------

## 五、蓝图本身是什么？它也能生成 UClass 吗？

讲到这里，观众可能会问：

**如果 C++ 类能对应一个 UClass，那么我在内容浏览器里创建的 `BP_Enemy`
究竟是什么？**

答案是：要把**蓝图资产**和**蓝图生成类**分开看。

### 5.1 内容浏览器中的 BP_Enemy 是资产

在内容浏览器中创建 `BP_Enemy`，得到的是一个蓝图资源。对于这里讨论的普通
Blueprint Class，它对应 `UBlueprint` 类型的资产对象。

这个资产保存蓝图的编辑信息，例如：

-   选择了什么父类；
-   声明了哪些变量；
-   编写了哪些函数和事件图；
-   节点如何连接；
-   定义了哪些组件。

**`UBlueprint` 资产本身不是 `BP_Enemy` 所定义的 Actor 类。**

### 5.2 蓝图编译生成类

蓝图编译器会根据资产中的定义，建立对应的 `UBlueprintGeneratedClass`。

例如：

``` text
BP_Enemy
（UBlueprint 资产）
       │
       │ 蓝图编译
       ▼
BP_Enemy_C
（UBlueprintGeneratedClass）
       │
       │ 可用于创建实例
       ▼
Enemy_01
（实际的 Actor 对象）
```

这里的"编译"不一定意味着你必须手动点击一次编译按钮后才首次拥有生成类。创建和加载蓝图时，UE
也会根据需要建立或加载生成类；后续编译可以更新它。

### 5.3 三者在开发中的区别

  ----------------------------------------------------------------------------
  对象                    它是什么                     主要用途
  ----------------------- ---------------------------- -----------------------
  `BP_Enemy`              `UBlueprint` 资产            保存、编辑蓝图定义

  `BP_Enemy_C`            `UBlueprintGeneratedClass`   描述可实例化的蓝图类
                          对象                         

  `Enemy_01`              Actor 实例                   在关卡中真正参与运行
  ----------------------------------------------------------------------------

因此，当你双击 `BP_Enemy`，是在编辑蓝图资产；当你把它拖入关卡时，UE
实际使用对应的生成类来创建 Actor。

### 5.4 为什么蓝图还能继承蓝图？

既然 `BP_Enemy` 能生成 `BP_Enemy_C`，而 `BP_Enemy_C` 又是 `UClass`
体系中的类，那么另一个蓝图就可以将它作为父类。

``` text
AMyActor（C++）
      │
      ▼
BP_Enemy_C（蓝图生成类）
      │
      ▼
BP_Boss_C（另一个蓝图生成类）
```

**蓝图继承 C++ 与蓝图继承蓝图，使用的是同一套 UObject 类继承体系。**

这正是统一类型系统带来的直接结果。

---------------------------------------------------------------------### 5.5 并非所有节点图资产都会生成新的 UClass

这里需要再澄清一个容易混淆的概念：**UE 中许多资源都有节点编辑界面，但并不是所有节点图都属于 Blueprint 类编译系统。**

| 资源类型 | 是否生成新的蓝图类 | 说明 |
|---|---|---|
| 普通 Blueprint Class | 是 | 通常生成 `UBlueprintGeneratedClass` |
| Level Blueprint | 是 | 生成关卡脚本对应的生成类 |
| Animation Blueprint | 是 | 生成 `UAnimBlueprintGeneratedClass`，用于动画实例 |
| Widget Blueprint | 是 | 生成 `UWidgetBlueprintGeneratedClass` |
| Blueprint Function Library | 是 | 生成函数库蓝图类 |
| Blueprint Interface | 是 | 生成接口对应的类描述，但不能按普通 Actor 类实例化 |
| Control Rig Blueprint | 是 | 使用 Control Rig 专门的蓝图编译及生成类机制 |
| Behavior Tree 行为树 | 否 | `UBehaviorTree` 资产，保存行为树结构，不生成一个新的行为树类 |
| Material Graph 材质图 | 否 | 材质系统编译图表达式，不生成 Blueprint 类 |
| Niagara System | 否 | Niagara 的图和脚本采用自己的编译与执行机制 |

例如，**动画蓝图确实能生成 UClass**：

```text
ABP_Character（动画蓝图资产）
    │ 编译
    ▼
ABP_Character_C（UAnimBlueprintGeneratedClass）
    │ 属于
    ▼
UBlueprintGeneratedClass → UClass
```

而行为树不同：

```text
BT_Enemy（UBehaviorTree 资产实例）
    ├── 行为树节点和连接关系
    └── 由行为树运行系统解释/执行
        （不会为 BT_Enemy 生成 BP_Enemy_C 那样的新类）
```

注意，**“不生成新的 UClass”不等于“资产没有 UClass”**：`BT_Enemy` 本身是一个 `UBehaviorTree` 对象，它的原生 C++ 类型 `UBehaviorTree` 当然有自己的 `UClass`。只是每新建一棵行为树，并不会为这棵树再生成一个新的 UObject 派生类。

同样，行为树中使用的 **Blueprint BTTask** 是另一回事：它是节点的蓝图类，可以生成自己的 UClass；这并不意味着行为树资产本身会生成新类。

**因此，本期视频所说的“蓝图能够生成 UClass”，特指使用 Blueprint 类编译机制的蓝图资源，不能泛化为所有可视化节点资产。**

---

## 六、实际案例：为什么蓝图类路径后面经常有 `_C`？

前面的知识可以解释一个开发中常见的现象。

内容浏览器里我们看到的是：

``` text
BP_Enemy
```

但在 C++ 加载蓝图生成类时，经常会看到：

``` text
/Game/Enemies/BP_Enemy.BP_Enemy_C
```

为什么多了 `_C`？

因为两者指向的对象不同：

-   `BP_Enemy`：蓝图资产；
-   `BP_Enemy_C`：蓝图生成类。

例如，按类路径加载：

``` cpp
UClass* EnemyClass = LoadClass<AActor>(
    nullptr,
    TEXT("/Game/Enemies/BP_Enemy.BP_Enemy_C")
);
```

这里需要的是一个 `UClass`，以便后续创建 Actor，而不是用于编辑的
`UBlueprint` 资产。

随后：

``` cpp
if (EnemyClass)
{
    AActor* Enemy = GetWorld()->SpawnActor<AActor>(
        EnemyClass
    );
}
```

**`_C` 反映了蓝图资产与蓝图生成类的区别。**

不过，并不是在 C++ 中使用蓝图类都需要手动拼接 `_C`。例如，通过
`TSubclassOf`、`TSoftClassPtr`、编辑器属性选择器等方式持有类引用时，可以避免手动处理类路径。

还要注意：`_C` 是蓝图生成类常见的命名形式，不应理解为"所有 UClass
的名字都必须以 `_C` 结尾"。

------------------------------------------------------------------------

## 七、最终总结：UClass 的来源与使用场景

我们最初的问题是：

**蓝图为什么能够继承 C++？**

现在可以给出完整回答：

UE 在原生 C++ 类型系统之上，建立了以 `UClass` 为核心的 UObject
类型系统。原生 C++ 的 UObject 类能够通过反射注册机制建立对应的
`UClass`；蓝图资产则能够通过蓝图编译器生成
`UBlueprintGeneratedClass`。由于蓝图生成类也属于 `UClass`
体系，因此可以与原生 UObject 类建立统一的继承关系。

但 `UClass` 的价值远远不止蓝图继承。

### 7.1 UClass 的两个主要来源

  -----------------------------------------------------------------------------
  来源                    主要过程                 结果
  ----------------------- ------------------------ ----------------------------
  原生 C++ UObject 类     UHT 生成辅助代码，C++    原生类对应的 `UClass`
                          编译，UE 注册            

  蓝图类资产              蓝图编译器处理蓝图定义   `UBlueprintGeneratedClass`
  -----------------------------------------------------------------------------

这里讨论的是常见 UObject 类的两个主要来源，而不是穷举 UE
内部所有类生成机制。

### 7.2 UClass 的六个典型使用场景

#### 场景一：动态创建对象

``` cpp
UClass* ClassToSpawn = AMyActor::StaticClass();

AMyActor* Actor = GetWorld()->SpawnActor<AMyActor>(
    ClassToSpawn
);
```

如果 `ClassToSpawn` 是兼容的蓝图生成类，就可以创建蓝图 Actor。

#### 场景二：判断对象类型

``` cpp
if (Actor && Actor->IsA(AMyActor::StaticClass()))
{
    // Actor 是 AMyActor 或其派生类的实例
}
```

不管实际类型来自原生 C++ 还是蓝图，都可以使用统一的类型判断方式。

#### 场景三：查询继承关系

``` cpp
UClass* Class = Actor->GetClass();

bool bResult = Class->IsChildOf(
    AMyActor::StaticClass()
);
```

这里可以判断一个类是否属于指定父类的继承体系。

#### 场景四：属性反射

``` cpp
UClass* Class = Actor->GetClass();

FProperty* Property = Class->FindPropertyByName(
    TEXT("Health")
);
```

通过 `UClass` 和 `FProperty`，UE 可以查询反射属性。Details
面板等系统就依赖这种类型元数据以及其他编辑器机制。

#### 场景五：函数反射

``` cpp
UFunction* Function = Actor->FindFunction(
    TEXT("SomeReflectedFunction")
);
```

查找的是反射系统中注册的函数。实际通过 `ProcessEvent`
调用时，还必须正确准备参数结构，不能简单地把任意普通 C++ 成员函数当成
`UFunction` 调用。

#### 场景六：获取类默认对象（CDO）

``` cpp
AMyActor* Defaults = GetMutableDefault<AMyActor>();
```

每个 UObject
类都有类默认对象，用于承载该类的默认属性值等信息。也可以通过
`UClass::GetDefaultObject()` 获取对应类的 CDO。

CDO 是理解蓝图默认值、对象初始化及属性继承的重要基础。

### 7.3 最终收尾讲稿

> 现在回到最开始的问题：蓝图为什么能够继承 C++？
>
> 关键在于 UE 不仅依赖原生 C++ 的类型系统，还为 UObject 建立了一套以
> UClass 为核心的类型系统。
>
> C++ 的 UObject 类通过反射注册机制建立对应的
> UClass；蓝图作为一种可编辑的资产，也能够通过蓝图编译生成自己的 UClass
> 派生对象。
>
> 这样，无论类来自 C++ 还是蓝图，UE
> 都能使用统一的类型描述和继承关系来管理它们。
>
> 而 UClass
> 不只是为蓝图继承服务。我们平时使用的对象创建、类型判断、属性反射、函数反射和类默认对象等机制，都与它密切相关。
>
> 所以，理解蓝图为什么能够继承 C++，实际上也是理解 UE 整个 UObject
> 类型系统的一个入口。

------------------------------------------------------------------------

## 视频制作建议

**建议标题：**《UE 底层原理：蓝图为什么能继承 C++？》

**讲解节奏：** 不要在开头一次性解释
UHT、UClass、UBlueprintGeneratedClass、Blueprint
VM。每出现一个新概念，都先给出它要解决的问题，再解释它的作用。

**推荐演示：**

1.  编写 `AMyActor`，创建 `BP_Enemy`，展示蓝图继承 C++ 的现象；
2.  打印 `AMyActor::StaticClass()` 与蓝图实例的 `GetClass()`；
3.  通过 `GetSuperClass()` 展示实际的父类关系；
4.  展示内容浏览器中的 `BP_Enemy` 与类路径中的 `BP_Enemy_C`；
5.  使用 `SpawnActor` 传入原生类和蓝图类，展示统一创建机制。

**本期边界：** 重点讲清楚 UClass
的意义、两种主要来源及蓝图继承关系。蓝图变量的具体内存布局、Blueprint VM
字节码、UHT 生成代码细节和 CDO
初始化流程，可以放到后续专题，不必一次讲完。
