原文地址: [tranek/GASDocumentation](https://github.com/BillEliot/GASDocumentation)  
翻译地址: [BillEliot/GASDocumentation_Chinese](https://github.com/BillEliot/GASDocumentation_Chinese)  

GAS文档优化版:正在更新


> 最好的文档永远是该插件的代码.  
> 
> 我的引擎版本: UE5.4

---

<a name="intro"></a>
## GameplayAbilitySystem插件简介

> 能力系统插件,简称GAS

该插件对于单人和多人游戏提供了开箱即用的解决方案:  

* 执行基于等级的角色能力(Ability), 该能力可选花费和冷却时间. 
* 管理属于Actor的数值Attribute. ([Attribute](#concepts-a))
* 为Actor应用状态效果. ([GameplayEffect](#concepts-ge))
* 为Actor应用GameplayTag. ([GameplayTag](#concepts-gt))
* 生成视觉或声音效果. ([GameplayCue](#concepts-gc))
* 为以上提到的所有应用同步(Replication).

在多人游戏中, GAS提供客户端预测([client-side prediction](#concepts-p))支持:  

* 能力激活.
* 播放蒙太奇.
* 对`Attribute`的修改.
* 应用`GameplayTag`.
* 生成`GameplayCue`.
* 通过连接于`CharacterMovementComponent`的`RootMotionSource`函数形成的移动.

**GAS必须由C++创建**, 但是`GameplayAbility`和`GameplayEffect`可由设计师在蓝图中创建.  

GAS中的现存问题:

* `GameplayEffect`延迟调节(Latency Reconciliation).(不能预测能力冷却时间, 导致高延迟玩家相比低延迟玩家, 对于短冷却时间的能力有更低的激活速率.)
* 不能预测性地移除`GameplayEffect`. 然而我们可以反向预测性地添加`GameplayEffect`, 从而高效的移除它. 但是这不总是合适或者可行的, 因此这仍然是个问题.
* 缺乏样例模板项目, 多人联机样例和文档. 希望这篇文档会有所帮助.

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts"></a>

### Ability System Component

> 能力系统组件,简称ASC

`AbilitySystemComponent (ASC)` 是 **GAS 的核心组件**，继承自 `UActorComponent` 
	所有需要使用 [GameplayAbility](#concepts-ga)、拥有 [Attribute](#concepts-a)、或接收 [GameplayEffect](#concepts-ge) 的 Actor，都必须挂载一个 ASC。它负责管理和同步这些对象（`Attribute` 的同步除外，由 [AttributeSet](#concepts-as) 处理）。通常建议开发者继承它进行自定义扩展。

ASC 关联两个核心对象：

- **OwnerActor**：ASC 的归属对象，一般用于持久化数据。

>   Owner:所属者或者说所有者,是逻辑上的归属关系.网络下，Owner 会影响哪些 Actor 复制到哪些客户端
>   Outer:UObject的生命周期与Outer同步,比如Actor在构造函数里创建组件也就是子对象时,Outer就为this

- **AvatarActor**：ASC 的物理表现对象，通常是实际出现在场景里的 Pawn/Character。

>   代理Actor.ASC属于OwnerActor所以**AvatarActor**销毁时不影响ASC中的数据

两者可以是同一个 Actor（如 MOBA 中的小兵），也可以不同（如 MOBA 英雄：`OwnerActor=PlayerState`，`AvatarActor=Character`）。  
如果角色需要 **重生并保留 Attribute/GameplayEffect**（例如英雄角色），推荐将 ASC 放在 `PlayerState` 上。

> ⚠️ 注意：若 ASC 放在 `PlayerState`，需要提高其 **NetUpdateFrequency**（默认很低），否则客户端上的 Attribute 或 GameplayTag 更新会有延迟。确保启用 [Adaptive Network Update Frequency](https://docs.unrealengine.com/en-US/Gameplay/Networking/Actors/Properties/index.html#adaptivenetworkupdatefrequency)，Fortnite 就是这样做的。

为保证系统交互，`OwnerActor` 必须实现 `IAbilitySystemInterface`；若 `AvatarActor` 与 `OwnerActor` 不同，`AvatarActor` 也应实现该接口。  
该接口要求重写：

`UAbilitySystemComponent* GetAbilitySystemComponent() const override;`

系统内部通过此函数获取 ASC 指针。

ASC 的主要数据存储：

- **ActiveGameplayEffect** (`FActiveGameplayEffectContainer`)：保存当前激活的 GE。
  
- **ActivatableAbility** (`FGameplayAbilitySpecContainer`)：保存授予该ASC的GA也就是可激活的GA。
  

在遍历 `ActivatableAbility.Items` 时，需要在循环前添加：

`ABILITYLIST_SCOPE_LOCK();`

这样可防止遍历过程中 Ability 列表被修改。每次进入该宏会增加 `AbilityScopeLockCount`，离开时递减。  
⚠️ 不要在锁定域内移除 Ability，否则内部检查会阻止删除。

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-asc-rm"></a>
#### 同步模式

`ASC` 提供三种不同的 **同步模式** 用于同步 `GameplayEffect`、`GameplayTag` 和 `GameplayCue`。  
`Attribute` 的同步由 `AttributeSet` 负责。

|同步模式|使用场景|同步内容|
|---|---|---|
|**Full**|单人游戏|所有 `GameplayEffect` 同步到客户端。|
|**Mixed**|多人游戏，玩家控制的 Actor|`GameplayEffect` 仅同步到本地客户端；`GameplayTag` 和 `GameplayCue` 同步到所有客户端。|
|**Minimal**|多人游戏，AI 控制的 Actor|`GameplayEffect` 不同步；仅同步 `GameplayTag` 和 `GameplayCue` 到所有客户端。|

---

**注意事项**：

- 使用 **Mixed 模式** 时，ASC的`OwnerActor` 的 `Owner` 必须是 `Controller`。
  
    - `PlayerState` 默认的 `Owner` 就是 `Controller`。
      
    - `Character` 默认的`Owner` **不是** `Controller` ，如果要用 `Mixed` 模式，需要手动调用：
	    `OwnerActor->SetOwner(Controller);`
    -  应在 `Pawn::PossessedBy(AController* NewController)` 中设置Pawn或者Character的Owenr为新的 `Controller`。

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-asc-setup"></a>

#### ASC 创建与初始化流程

`ASC` 通常在 **OwnerActor 的构造函数** 中创建，并且必须标记为 **Replicated**（只能在 C++ 中完成）：

```C++
AMyPlayerState::AMyPlayerState() { 	
	// 创建 AbilitySystemComponent，并显式启用网络同步 	
	AbilitySystemComponent = CreateDefaultSubobject<UGDAbilitySystemComponent>(TEXT("AbilitySystemComponent")); 	      AbilitySystemComponent->SetIsReplicated(true); 
}
```
---


ASC 必须在 **服务端和客户端** 都初始化（即调用 `InitAbilityActorInfo(OwnerActor, AvatarActor)`）
> 单机游戏只需参考服务端的做法。

---

##### 情况 1：ASC 位于 Character（Pawn）

- **服务端**：在 `Pawn::PossessedBy()` 中初始化，并确保 Mixed 模式时把 `Owner` 指向 `Controller`。
  
- **客户端**：在 `PlayerController::AcknowledgePossession()` 中初始化。
  

```C++
void APACharacterBase::PossessedBy(AController* NewController)
{
    Super::PossessedBy(NewController);

    if (AbilitySystemComponent)
    {
        AbilitySystemComponent->InitAbilityActorInfo(this, this);
    }

    // Mixed 模式要求 ASC 的 Owner->Owner = Controller
    SetOwner(NewController);
}

void APAPlayerControllerBase::AcknowledgePossession(APawn* P)
{
    Super::AcknowledgePossession(P);

    if (APACharacterBase* CharacterBase = Cast<APACharacterBase>(P))
    {
        CharacterBase->GetAbilitySystemComponent()- >InitAbilityActorInfo(CharacterBase, CharacterBase); 
    }
}
```


---

##### 情况 2：ASC 位于 PlayerState

- **服务端**：在 `Pawn::PossessedBy()` 中初始化。
  
- **客户端**：在 `Character::OnRep_PlayerState()` 中初始化（确保 PlayerState 已经存在）。
  

```C++
void AGDHeroCharacter::PossessedBy(AController* NewController)
{
    Super::PossessedBy(NewController);

    if (AGDPlayerState* PS = GetPlayerState<AGDPlayerState>())
    {
        // 服务端设置 ASC（客户端在 OnRep_PlayerState 中处理）
        AbilitySystemComponent = PS->GetAbilitySystemComponent();

        // AI 没有 PlayerController，这里再初始化一次也没关系
        PS->GetAbilitySystemComponent()->InitAbilityActorInfo(PS, this);
    }
}

void AGDHeroCharacter::OnRep_PlayerState()
{
    Super::OnRep_PlayerState();

    if (AGDPlayerState* PS = GetPlayerState<AGDPlayerState>())
    {
        // 客户端设置 ASC（服务端在 PossessedBy 中处理）
        AbilitySystemComponent = PS->GetAbilitySystemComponent();

        // 客户端初始化 ASC
        AbilitySystemComponent->InitAbilityActorInfo(PS, this);
    }
}
```


✅ **总结**

- **ASC 在 Character 上**：服务端 → `PossessedBy()`，客户端 → `AcknowledgePossession()`。
  
- **ASC 在 PlayerState 上**：服务端 → `PossessedBy()`，客户端 → `OnRep_PlayerState()`。
  
- Mixed 模式下，如果 ASC 在 Character 上，记得 `SetOwner(Controller)`。
---

##### 初始化错误

如果遇到以下警告：

```C++
LogAbilitySystem: Warning: Can't activate LocalOnly or LocalPredicted Ability %s when not local!
```

说明 **ASC 没有在客户端正确初始化**。  
此时应检查 `PossessedBy()` 与 `OnRep_PlayerState()`/`AcknowledgePossession()` 是否都正确调用了 `InitAbilityActorInfo()`。

---

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-gt"></a>
### GameplayTag

**`FGameplayTag`** 由 **GameplayTagManager** 注册，采用层级式命名（如 `State.Debuff.Stun`）。  
它常用于 **分类与描述对象状态**，相较布尔值/枚举，更直观、可扩展，并支持父子层级逻辑。

---

标签与 ASC 的交互

- 若对象拥有 **AbilitySystemComponent (ASC)**，通常将标签附加到 ASC。
  
- `ASC` 实现了 `IGameplayTagAssetInterface`，可直接查询当前标签。
  

---

#### 响应Gameplay Tags的变化

`ASC`（Ability System Component）提供两种注册方式监听 Gameplay Tag 的变化：

1. **监听指定 Tag 的添加/移除**  
    调用 `RegisterGameplayTagEvent()` 绑定代理（Delegate），可在指定 Tag 被添加或移除时触发。通过 `EGameplayTagEventType` 参数，可以明确监听的是 Tag 的新增/移除，还是其 `TagMapCount` 变化。
    
    `AbilitySystemComponent->RegisterGameplayTagEvent(     FGameplayTag::RequestGameplayTag(FName("State.Debuff.Stun")),     EGameplayTagEventType::NewOrRemoved ).AddUObject(this, &AGDPlayerState::StunTagChanged);`
    
2. **监听所有 Tag 的变化**  
     调用`RegisterGenericGameplayTagEvent()` 可绑定代理，用于监听 ASC 中 **所有 Tag** 的变化。
     
     **[⬆ 返回目录](#table-of-contents)**

<a name="concepts-a"></a>
GameplayTagContainer 与同步

- **多个标签** 存放于 `FGameplayTagContainer`（比 `TArray<FGameplayTag>` 更高效，仍可转换为数组遍历）。
  
- 标签本质是 `FName`。启用 **Project Settings → GameplayTags → Fast Replication** 后：
  
    - 标签能被高效打包同步；
      
    - 要求客户端与服务端标签列表一致（通常不成问题）。  
        ✅ 建议始终启用。
        

---

**GameplayTagCountContainer**

- `ASC` 内部使用 **`FGameplayTagCountContainer`** 存储标签及其引用计数：
  
    `FGameplayTagCountContainer GameplayTagCountContainer;`
    
    - 自动收集 **GE 授予的标签** 与 **ASC 激活的 GameplayCue 标签**；
      
    - 也可通过 `ASC->AddLooseGameplayTag()` 手动添加。
    
- GameplayTagCountContainer内部维护 `TagMap[Tag] = Count`：
  
    - `Count == 0` → 逻辑判断时视为不存在；
      
    - `Count > 0` → 视为拥有该标签。
      

---

#### 定义与管理标签的方式

1. **配置文件声明**：`DefaultGameplayTags.ini`（推荐通过编辑器管理，无需手动写 ini）。
   
2. **C++ 动态注册**：在 `UAssetManager::StartInitialLoading()` 中调用  
    `UGameplayTagsManager::Get().AddNativeGameplayTag()`。
    
3. **宏定义方式**（推荐）：
   
    - 头文件：
      
        `UE_DECLARE_GAMEPLAY_TAG_EXTERN(Ability_ActivateFail_IsDead);`
        
    - cpp 文件：
      
        `UE_DEFINE_GAMEPLAY_TAG_COMMENT(Ability_ActivateFail_IsDead,    "Ability.ActivateFail.IsDead","技能释放失败，因为角色已经死亡。");`
---

GameplayTag 编辑与引用

![[attachments/Pasted image 20250818174501.png|503]]
- **搜索引用**
  
    - 在标签管理器中右击标签会显示标签的引用
      
    - ⚠️ 注意：不会显示引用该标签的 **C++ 类**。
    
- **重命名与重定向**
  
    - 重命名 `GameplayTag` 会创建 **重定向器(一种数据)**。
	    ![[attachments/Pasted image 20250818174616.png|308]]
    - 旧资源仍会通过**重定向器**指向新标签。
	  
    - 建议做法：重命名标签后右击内容更新重定向器然后删除重定向起(在标签管理器界面)
    
- **Fast Replication 与深度优化**
  
    - 编辑器提供选项可将常见复制标签标记以进行深度优化。
      
    - 由 `GameplayEffect` 添加的标签会自动同步。
    
- **LooseGameplayTag**
  
    - `ASC` 支持通过代码添加同步或者不同步的 LooseGameplayTag，需手动管理。
      
    - 示例：`State.Dead` 使用 LooseGameplayTag，在角色死亡时客户端立即响应。
      
    - 重生时必须手动将 `TagMapCount` 置零。
      
    - 使用接口函数添加或者移除：
      
        `UAbilitySystemComponent::AddLooseGameplayTag(Tag); UAbilitySystemComponent::RemoveLooseGameplayTag(Tag);`
        

---

C++ 获取 GameplayTag

- 请求单个标签引用：
  

`FGameplayTag Tag = FGameplayTag::RequestGameplayTag(FName("Your.GameplayTag.Name"));`

- 高阶操作（获取父/子标签、遍历层级等）通过 **GameplayTagManager**：
  

`#include "GameplayTagManager.h"  UGameplayTagManager::Get().FunctionName(...);`

- 相比字符串比较，使用 `GameplayTagManager` 处理标签关系更高效。
  

---

**UPROPERTY 和UFunction标签过滤**

- **GameplayTag 可使用 `Meta` 属性过滤显示：
  

`UPROPERTY(EditAnywhere, BlueprintReadWrite, meta = (Categories = "GameplayCue")) FGameplayTag TagVariable;`

- 仅显示父标签为 `"GameplayCue"` 的标签。
  
- 蓝图函数参数过滤：
`UFUNCTION(BlueprintCallable, meta = (GameplayTagFilter = "GameplayCue")) void MyFunction(FGameplayTag TagParam);`

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-gt-change"></a>
### Attribute

#### Attribute定义
`Attribute` 由 `FGameplayAttributeData` 结构体定义，表示一个浮点值，可用于任何游戏相关数值，例如角色生命值、等级或药水剂量。**如果某个数值属于 Actor 且与游戏逻辑相关，就应考虑使用 `Attribute`。**

`Attribute` 应只通过 [GameplayEffect](#concepts-ge) 修改，以便 `ASC` 能够进行 [预测(Predict)](#concepts-p) 改变。

`Attribute` 一般由 `AttributeSet` 定义并存储，`AttributeSet` 会同步标记为 replication 的 `Attribute`。具体的定义方式可参阅 [AttributeSet](#concepts-as) 部分。

**Tip**：如果不希望某个 `Attribute` 在编辑器的 Attribute 列表中显示，可添加 `Meta = (HideInDetailsView)` 属性宏。

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-a-value"></a>
#### BaseValue vs. CurrentValue

每个 `Attribute` 包含两个值：**`BaseValue`** 和 **`CurrentValue`**。

- **BaseValue**：属性的永久值，只受 **Instant(立即)** GE影响(包括周期性GE)。
  
- **CurrentValue**：在 `BaseValue` 基础上，加上所有 **持续性GE**和**永久性GE** 的修改值后的值。
  

> 注意：新手常误将 `BaseValue` 当作属性的最大值，这种做法是错误的。用于限制属性最大值，应该单独定义为另一个 `MaxAttribute`,如MaxHealth。

属性值限制（Clamp）相关的处理：

- 在AttributeSet->[PreAttributeChange()](#concepts-as-preattributechange) 中处理 `CurrentValue` 的修改
  
- 在 AttributeSet->PreAttributeBaseChange() 中处理 `GameplayEffect` 对 `BaseValue` 的修改
  

**GameplayEffect 类型与修改属性的关系**：

- **Instant（即时）**：可永久修改 `BaseValue`
- **Periodic（周期性）**：视为周期性 Instant，可修改 `BaseValue`
- **Duration（持续）/Infinite（无限）**：可修改 `CurrentValue`
  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-a-meta"></a>
#### 元属性 MetaAttribute

某些 不复制的`Attribute` 被视为占位符，用于预测或与其他 `Attribute` 交互的临时值，这类 `Attribute` 称为 **Meta Attribute**。

`/** 伤害元属性 */
`UPROPERTY(BlueprintReadOnly, Category = "Aura|属性集", meta = (AllowPrivateAccess))  `
`FGameplayAttributeData IncomingDamage;
例如，通常我们会定义 **伤害值** 为 Meta Attribute，而不是直接通过 `GameplayEffect` 修改生命值 `Attribute`。这样做的好处是：

- 伤害值可以在 [GameplayEffectExecutionCalculation](#concepts-ge-ec) 中被 buff 或 debuff 调整
  
- 可以在 `AttributeSet` 中进一步处理，例如在最终从生命值扣除伤害值前，先减去当前护盾值
  
- Meta Attribute 在不同 `GameplayEffect` 之间不是持久化的，并可被任何一方重写
  

Meta Attribute 的设计优势在于，它将“我们应该造成多少伤害？”和“我们该如何处理伤害值？”的问题解耦：

1. **GameplayEffect**：确定伤害量
   
2. **AttributeSet**：决定如何使用伤害量

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-a-changes"></a>
#### 响应Attribute变化

要监听 `Attribute` 的变化，以便更新 UI 或处理其他游戏逻辑，可以使用：

`UAbilitySystemComponent::GetGameplayAttributeValueChangeDelegate()`

该函数返回一个委托（Delegate），可绑定在 `Attribute` 变化时自动调用的函数。委托回调提供一个 `FOnAttributeChangeData` 参数，其中包含：

- `NewValue`：变化后的值
  
- `OldValue`：变化前的值
  
- `FGameplayEffectModCallbackData`：相关 GameplayEffect 回调数据（**注意**：此字段仅能在服务端使用）
  

也可以通过能力任务类`AbilityTask_WaitAttributeChange`(蓝图中为WaitAttributeChange)在能力激活时进行监听
**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-a-derived"></a>
#### 自动更新Attribute值

如果希望一个属性自动基于其他属性更新,可以给ASC应用一个永久GE,修改器使用**基于 Attribute 或 [MMC](#concepts-ge-mmc) **。
如图是MaxMana(最大法力值)等于智力的两倍
![[attachments/Pasted image 20250818183930.png]]

> **注意**：在 PIE 中打开多个窗口时，需要在编辑器首选项中禁用 `Run Under One Process`，否则除了第一个窗口外，基于属性计算的属性的更新将无效


**[⬆ 返回目录](#table-of-contents)**

### AttributeSet

> [!NOTE]
> 简称AS

#### 创建AttributeSet

`AttributeSet`用于管理`Attribute`. 开发者应该继承[UAttributeSet](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/UAttributeSet/index.html),ASC拥有的属性
- 在OwnerActor的构造函数中创建`AttributeSet`会自动注册到其`ASC`. **这必须在C++中完成.**  如OwnerActor是PlayerState:
![[attachments/Pasted image 20250818184519.png]]

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-as-design"></a>
#### 设计AttributeSet

一个 `ASC` 可能包含 **一个或多个 `AttributeSet`**。由于 `AttributeSet` 消耗的内存极低，具体使用多少个由开发者决定。

**方案 1：单一大型 AttributeSet**

- 创建一个包含所有 Attribute 的大型 `AttributeSet`，共享于游戏中所有 Actor
  
- Actor 只使用其中需要的 Attribute，忽略不需要的部分
  

**方案 2：多个按需 AttributeSet**

- 将 Attribute 分组为多个 `AttributeSet`，按需添加到 Actor
  
    - 例如：生命相关的 AttributeSet、魔法相关的 AttributeSet 等
      
    - 在 MOBA 游戏中，英雄可能需要魔法 AttributeSet，而小兵则不需要
      

> 注意：虽然可以拥有多个 AttributeSet，但 **同一 ASC 不应包含多个同一类的 AttributeSet**。  
> 如果存在多个同类 AttributeSet，ASC 无法确定使用哪一个，会随机选择一个，可能导致不可预期的行为。

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-as-design-subcomponents"></a>
**使用单独Attribute的子组件**

假设某个 Pawn 拥有多个可被单独伤害的组件，例如独立护甲片。

**方案 1：固定数量的可伤害组件**

- 如果可被伤害组件的最大数量明确，可以将多个生命值 Attribute 放入同一个 `AttributeSet` 中：
  
    `DamageableCompHealth0, DamageableCompHealth1, …`
    
- 每个 Attribute 表示逻辑上的一个“slot”
  
- 在可被伤害组件的实例中，可指定带 slot 编号的 Attribute，使 `GameplayAbility` 或 [Execution](#concepts-ge-ec) 确定伤害应用到哪个 Attribute
  
- 即使 Pawn 当前拥有 0 个或少于最大数量的组件也无妨，未使用的 Attribute 仅占用极少内存
  

**方案 2：子组件数量不固定或每个组件需要大量 Attribute**

- 如果子组件数量可能无限、可以分离给其他玩家使用（如武器），或其他原因无法使用固定 AttributeSet
  
- 建议**不使用 Attribute**，而在组件内部直接保存普通浮点数
  
- 参阅 [Item Attribute](#concepts-as-design-itemattributes) 获取更多设计方案

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-as-design-addremoveruntime"></a>
#### 运行时添加和移除AttributeSet

`AttributeSet` 可以在运行时从 `ASC` 中添加或移除，但**移除操作存在风险**：

- 如果客户端提前于服务端移除 AttributeSet，而某个 Attribute 的变化已同步到客户端，客户端找不到对应的 AttributeSet 会导致游戏崩溃。
  

**示例：武器 AttributeSet 的动态管理**

- **添加武器到 Inventory：**
  

`AbilitySystemComponent->SpawnedAttribute.AddUnique(WeaponAttributeSetPointer); AbilitySystemComponent->ForceReplication();`

- **从 Inventory 移除武器：**
  

`AbilitySystemComponent->SpawnedAttribute.Remove(WeaponAttributeSetPointer); AbilitySystemComponent->ForceReplication();`

> ⚠️ **注意**：动态移除 AttributeSet 应谨慎处理，确保服务端和客户端的同步顺序，避免 Attribute 丢失引发崩溃。

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-as-design-itemattributes"></a>
#### 如何处理物品Attribute

对于可装备物品（如武器弹药、盔甲耐久等），这些属性需要在物品自身存储数据，以支持**物品在生命周期中可被多个玩家装备**。常见实现方法有三种：

1. **在物品中使用普通浮点数（推荐）**
   
    - 简单直接，性能开销低
      
    - 适合数量不多且逻辑简单的属性
    
2. **在物品中使用单独的 `AttributeSet`**
   
    - 将属性封装在 AttributeSet 中，方便使用 GAS 系统的 Modifier/Execution 机制
      
    - 更适合复杂计算或需要与 AbilitySystem 交互的属性
    
3. **在物品中使用单独的 `ASC`（Ability System Component）**
   
    - 为每个物品独立创建 ASC，完全支持 GAS 系统
      
    - 开销较大，仅在物品属性复杂且需要完整 GAS 功能时使用
      

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-as-design-itemattributes-plainfloats"></a>
**在物品中使用普通浮点数**

在物品实例中直接存储普通浮点数而非 `Attribute`，像 Fortnite 和 [GASShooter](https://github.com/tranek/GASShooter) 就采用了这种方式来处理枪械弹药。

对于枪械，可以在实例中存储可同步的浮点数（`COND_OwnerOnly`），例如：

- 最大弹匣容量
  
- 当前弹匣弹药量
  
- 剩余总弹药量
  

如果某个枪械需要共享剩余弹药量，则可以将剩余弹药量移到角色的共享 `AttributeSet` 中，作为一个 `Attribute`。例如，换弹 Ability 可以通过一个 Cost 类型的 `GameplayEffect`（GE）从剩余弹药中填充枪械弹匣。

由于当前弹匣弹药量没有使用 `Attribute`，因此需要重写 `UGameplayAbility` 中的部分函数来检查和应用枪械实例中浮点数的消耗。在授予 Ability 时，可以将枪械作为 `SourceObject` 存入 `GameplayAbilitySpec`，从而在 Ability 中访问对应枪械实例。

为了避免在全自动射击时因同步延迟导致客户端弹药量异常（弹药减少后又突然增加），可以在玩家拥有 `IsFiring` 的 `GameplayTag` 时，在 `PreReplication()` 中禁用同步，从而实现本地预测：

```C++
void AGSWeapon::PreReplication(IRepChangedPropertyTracker& ChangedPropertyTracker)
{
    Super::PreReplication(ChangedPropertyTracker);

    DOREPLIFETIME_ACTIVE_OVERRIDE(
        AGSWeapon, PrimaryClipAmmo, 
        (IsValid(AbilitySystemComponent) && !AbilitySystemComponent->HasMatchingGameplayTag(WeaponIsFiringTag))
    );
    DOREPLIFETIME_ACTIVE_OVERRIDE(
        AGSWeapon, SecondaryClipAmmo, 
        (IsValid(AbilitySystemComponent) && !AbilitySystemComponent->HasMatchingGameplayTag(WeaponIsFiringTag))
    );
}
```


**优点**：

1. 避免了使用 `AttributeSet` 的限制（详见下文）。
   

**局限**：

1. 不能直接使用现有的 `GameplayEffect` 流程（如弹药消耗的 Cost GE 等）。
   
2. 需要重写 `UGameplayAbility` 中的关键函数以检查和应用枪械浮点数消耗。

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-as-design-itemattributes-attributeset"></a>
**在物品中使用AttributeSet**

在物品中使用单独的 `AttributeSet` 可以实现 [将其添加到玩家的 Inventory](#concepts-as-design-addremoveruntime) 的功能，但仍存在一些局限性。

较早版本的 [GASShooter](https://github.com/tranek/GASShooter) 就使用了这种方法：

- 武器类将最大弹匣量、当前弹匣弹药量、剩余弹药量等存储在自身的 `AttributeSet` 中。
  
- 如果枪械需要共享剩余弹药量，则将其移到角色的共享弹药 `AttributeSet` 中。
  

当服务端将武器添加到玩家 Inventory 时：

1. 武器的 `AttributeSet` 会被加入到玩家的 `ASC::SpawnedAttribute` 中。
   
2. 服务端随后将其同步到客户端。
   
3. 如果武器从 Inventory 移除，其 `AttributeSet` 也会从 `ASC::SpawnedAttribute` 中移除。
   

**注意事项**：

当 `AttributeSet` 存于除 OwnerActor 外的对象上时（例如某个武器），可能会遇到编译错误。解决方法是：

- 在 `BeginPlay()` 中构建 `AttributeSet`，而不是在构造函数中。
  
- 在武器类中实现 `IAbilitySystemInterface`，并在将武器添加到玩家 Inventory 时设置 `ASC` 指针。
  

示例：

```C++
void AGSWeapon::BeginPlay()
{
    if (!AttributeSet)
    {
        AttributeSet = NewObject<UGSWeaponAttributeSet>(this);
    }
    //...
}
```


可参考 [GASShooter 较早版本](https://github.com/tranek/GASShooter/tree/df5949d0dd992bd3d76d4a728f370f2e2c827735) 实际体验这种方案。

**优点**：

1. 可以直接使用现有的 `GameplayAbility` 和 `GameplayEffect` 工作流（如弹药消耗的 Cost GE）。
   
2. 对于小型物品集，设置快速方便。
   

**局限**：

1. 必须为每种武器类型创建独立的 `AttributeSet` 类。`ASC` 对于每个 `AttributeSet` 类只能存在一个实例，修改时会只寻找该类的第一个实例，其他相同类的实例会被忽略。
   
2. 由于每个 `AttributeSet` 类只能存在一个实例，在玩家 Inventory 中每种武器类型也只能有一个。
   
3. 移除 `AttributeSet` 风险较高。例如在 GASShooter 中，如果玩家因火箭弹自杀，武器会立即从 Inventory 移除（包括其 `ASC` 上的 `AttributeSet`），这时服务端同步武器弹药 Attribute 的变化会导致客户端崩溃。

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-as-design-itemattributes-asc"></a>
**在物品中使用单独的ASC**

在每个物品上都创建一个独立的 `AbilitySystemComponent`（ASC）是一种极端方案。我本人尚未实践过，也很少在其他项目中见到这种做法。该方案可能需要相当高的开发成本才能稳定使用。

Epic 社区中 Dave Ratti 对此的讨论指出：[community questions #6](https://epicgames.ent.box.com/s/m1egifkxv3he3u3xezb9hzbgroxyhx89)：

> **问题**：是否可行在同一 Owner 下存在多个 ASC（例如 Pawn、武器、物品或投射物），且它们的 Avatar 不同？
> 
> **回答摘要**：
> 
> - 最大的问题是实现 `IGameplayTagAssetInterface` 和 `IAbilitySystemInterface`。
>     
>     - `IGameplayTagAssetInterface` 可能可行，但需要聚合所有 ASC 的标签。注意：`HasAllMatchingGameplayTags` 的检查可能需要跨 ASC 聚合，仅转发并 OR 结果是不够的。
>         
>     - `IAbilitySystemInterface` 更棘手：哪个 ASC 是权威的？若应用 GE，应该作用于哪一个 ASC？拥有多个 ASC 会带来复杂性。
>         
> - 独立的 ASC 在 Pawn 和武器上可能有意义，例如区分描述武器的标签和描述 Pawn 的标签。标签可以在武器 ASC 中授予，并“应用”到 Owner，但 Attribute 和 GE 是独立的。总的来说，同一 Owner 下存在多个 ASC 可能会很棘手。
>     

**优点**：

1. 可以使用现有的 `GameplayAbility` 和 `GameplayEffect` 流程（如弹药消耗的 Cost GE）。
   
2. 可以复用 `AttributeSet` 类，每个武器 ASC 各自拥有独立实例。
   

**局限**：

1. 开发成本未知，可能很高。
   
2. 方案可行性存在不确定性，需谨慎评估。

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-as-attributes"></a>
#### 定义Attribute

**注意：Attribute 只能在 C++ 中的 AttributeSet 头文件中定义。**

建议在每个 `AttributeSet` 头文件顶部添加如下宏，它会自动为每个 Attribute 生成 getter 和 setter 函数：

```C++
// 使用 AttributeSet.h 中的宏
#define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
	GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)
```


例如，一个可同步的生命值 Attribute 可以这样定义：

```C++
UPROPERTY(BlueprintReadOnly, Category = "Health", ReplicatedUsing = OnRep_Health)
FGameplayAttributeData Health;

ATTRIBUTE_ACCESSORS(UGDAttributeSetBase, Health)
```


在头文件中定义对应的 OnRep 函数：

`UFUNCTION() virtual void OnRep_Health(const FGameplayAttributeData& OldHealth);`

在 `.cpp` 文件中使用预测系统（Prediction System）所需的 `GAMEPLAYATTRIBUTE_REPNOTIFY` 宏实现 OnRep：

```C++
void UGDAttributeSetBase::OnRep_Health(const FGameplayAttributeData& OldHealth)
{
	GAMEPLAYATTRIBUTE_REPNOTIFY(UGDAttributeSetBase, Health, OldHealth);
}
```


同时，需要将 Attribute 添加到 `GetLifetimeReplicatedProps` 中，以实现网络同步：

```C++
void UGDAttributeSetBase::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);

	DOREPLIFETIME_CONDITION_NOTIFY(UGDAttributeSetBase, Health, COND_None, REPNOTIFY_Always);
}
```

> **说明**：
> 
> - `REPNOTIFY_Always` 会让 OnRep 在客户端值与服务端同步值相同的情况下也触发，这对于使用预测系统的 Attribute 很重要。
>     
> - 如果 Attribute 不需要像 Meta Attribute 那样同步，则可以跳过 OnRep 和 `GetLifetimeReplicatedProps` 这两个步骤。
>

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-as-init"></a>
#### 初始化Attribute

初始化 Attribute（即将 BaseValue 和 CurrentValue 设置为初始值）有多种方法。Epic 建议使用 **即刻（Instant）GameplayEffect**，这也是样例项目中推荐的做法。


> **注意**：
> 
> - 在 Unreal Engine 4.24 之前，`FAttributeSetInitterDiscreteLevels` 不能与 `FGameplayAttributeData` 一起使用。
>     
>     - 它只能用于原生浮点数 Attribute，并且与非 POD 类型的 FGameplayAttributeData 冲突。
>         
> - 该问题在 4.24 中已修复：[UE-76557](https://issues.unrealengine.com/issue/UE-76557)。
>

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-as-preattributechange"></a>
#### PreAttributeChange()

`AttributeSet->PreAttributeChange()` 它在 Attribute 的 **CurrentValue 修改前** 被调用，是限制（Clamp）或调整即将生效的值的理想位置。

例如限制移动速度 Modifier：

```
if (Attribute == GetMoveSpeedAttribute())
{
    // 限制速度最小为 150，最大为 1000
    NewValue = FMath::Clamp<float>(NewValue, 150, 1000);
}
```


> **说明**：
> 
> - `GetMoveSpeedAttribute()` 由我们在 `AttributeSet.h` 中添加的宏（见 [定义 Attribute](#concepts-as-attributes)）自动生成。
>     
> - `PreAttributeChange()` 会响应 Attribute 的任何修改，无论是通过 Attribute 的 setter（宏生成）还是通过 [GameplayEffect](#concepts-ge) 修改。
>     

**注意事项**：

1. 在 `PreAttributeChange()` 中做的任何限制 **不会永久修改 ASC 中的 Modifier**，仅会修改查询时的返回值。
   
    - 这意味着像 [GameplayEffectExecutionCalculations](#concepts-ge-ec) 或 [ModifierMagnitudeCalculations](#concepts-ge-mmc) 等重新计算 CurrentValue 的函数需要再次执行限制操作。
    如下图的执行(好像是属性改变时才会调用,重新计算属性时不会调用PreAttributeChange())
     ![[attachments/Pasted image 20250818192010.png]]
    
2. Epic 建议 **不要在 PreAttributeChange() 中执行游戏逻辑事件**，它主要用于数值限制（Clamp）。
   
    - 对于响应 Attribute 修改的游戏逻辑事件，推荐使用 `UAbilitySystemComponent::GetGameplayAttributeValueChangeDelegate(FGameplayAttribute Attribute)`（见 [响应 Attribute 变化](#concepts-a-changes)）。

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-as-postgameplayeffectexecute"></a>
#### PostGameplayEffectExecute()

`PostGameplayEffectExecute(const FGameplayEffectModCallbackData& Data)` 仅在 **即刻（Instant）GameplayEffect** 修改 Attribute 的 BaseValue 后触发。它是处理 Attribute 进一步逻辑的理想位置。

例如，在样例项目中：

- 我们从生命值 Attribute 中减去最终伤害值（Meta Attribute）。
  
- 如果目标有护盾值 Attribute，会先从护盾值中扣除伤害，再扣除生命值。
  
- 同时，这里也处理被击打反应动画、浮动伤害数值显示，以及击杀者的经验和赏金分配。
  

设计上，伤害值元属性 MetaAttribute 总是通过 **即刻 GameplayEffect** 传递，而不会直接通过 Attribute Setter 修改。

其他只由 **即刻 GameplayEffect** 修改 BaseValue 的 Attribute，例如魔法值或耐力值，也可以在这里根据其最大值 Attribute 限制当前值。

> **注意**：  
> 当 `PostGameplayEffectExecute()` 被调用时，Attribute 的修改已经发生，但尚未同步到客户端。
> 
> - 因此在这里进行限制（Clamp）不会触发额外的客户端同步。
>     
> - 客户端接收到的 Attribute 值已经是限制后的最终值。
>  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-as-onattributeaggregatorcreated"></a>
####  OnAttributeAggregatorCreated()

> 聚合器就是在GE应用的最后,收集所有有修改单个属性的修改器变成对应聚合器,也就是聚合器是按修改的属性分类的,你可以在聚合器中的EvaluationMetaData(评估元数据)选择该属性需要哪些修改器,比如只要加法修改器
> 	提示:执行的最后也是输出修改器

`OnAttributeAggregatorCreated()` 会在为集合中的某个 Attribute 创建 Aggregator 时触发。它允许自定义 `FAggregatorEvaluateMetaData`，用于控制 Aggregator 如何基于所有应用的 **Modifier** 计算 Attribute 的 CurrentValue。

---

#### AggregatorEvaluateMetaData 作用

- 默认情况下，AggregatorEvaluateMetaData 用于确定哪些 Modifier 会被纳入最终 CurrentValue 计算。
  
- 示例：`MostNegativeMod_AllPositiveMods`
  
    - 允许 **所有正（Positive）Modifier**
      
    - 限制 **负（Negative）Modifier** 仅应用最负的那一个
      
    - 应用场景：Paragon 中的移动速度
      
        - 只允许最强的减速效果生效
          
        - 不受正向增益数量影响
    
- 不满足条件的 Modifier 仍存在于 ASC 中，只是不计入 CurrentValue。
  
    - 当条件变化（如最负 Modifier 过期），下一个满足条件的 Modifier 会自动生效。
      

---

**示例：为移动速度 Attribute 设置自定义 Evaluator**

```C++
void UGSAttributeSetBase::OnAttributeAggregatorCreated(
    const FGameplayAttribute& Attribute, 
    FAggregator* NewAggregator
) const
{
    Super::OnAttributeAggregatorCreated(Attribute, NewAggregator);

    if (!NewAggregator)
    {
        return;
    }

    if (Attribute == GetMoveSpeedAttribute())
    {
        NewAggregator->EvaluationMetaData = 
    &FAggregatorEvaluateMetaDataLibrary::MostNegativeMod_AllPositiveMods;
    }
}
```


> **注意**：  
> 自定义的 `AggregatorEvaluateMetaData` 应作为静态变量添加到 `FAggregatorEvaluateMetaDataLibrary`，便于在不同 Attribute 上复用。

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ge"></a>
###  Gameplay Effect
游戏效果,简称GE

#### 定义GameplayEffect

GE是修改自身或其他 **Attribute** / **GameplayTag** 的数据容器。

- 它可以：
  
    - **立即修改 Attribute**（如伤害或治疗）
      
    - **应用长期状态**（如移动速度加速或眩晕）
    
- `UGameplayEffect` 本质上只是数据类，不应包含额外逻辑。
  
- 设计师通常会创建许多 `UGameplayEffect` 蓝图子类。
  

`GameplayEffect` 的主要修改方式：

- **Modifier**
  
- **Execution (GameplayEffectExecutionCalculation)**
  

---

#### 持续类型（DurationType）

`GameplayEffect` 分为三种持续类型：

|类型|GameplayCue 事件|使用场景|
|---|---|---|
|**即刻 (Instant)**|Execute|对 Attribute 的 BaseValue 立即进行永久修改。不会应用 GameplayTag。|
|**持续 (Duration)**|Add & Remove|临时修改 Attribute 的 CurrentValue。当 Effect 过期或移除时，移除对应 GameplayTag。持续时间在 UGameplayEffect 类/蓝图中指定。|
|**无限 (Infinite)**|Add & Remove|临时修改 Attribute 的 CurrentValue，自身永不过期，需要由 Ability 或 ASC 手动移除。|

---

#### 周期性 GameplayEffect

- **持续 (Duration)** 和 **无限 (Infinite)** 类型可以选择 **周期性执行**
  
- 每隔 X 秒（周期定义）应用一次 Modifier 和 Execution
  
- 周期性 Effect 修改 BaseValue 或执行 GameplayCue 时，被视为 **即刻 Effect**
  
- 典型应用：随时间推移的持续伤害
  
- ⚠️ 注意：周期性 Effect **不能被预测**

---

#### 手动重新计算 Modifier

- 当需要基于非 Attribute 数据的 Modifier（MMC）手动更新时，可使用 `SetActiveGameplayEffectLevel()`：
  

```C++
UAbilitySystemComponent::SetActiveGameplayEffectLevel(
    FActiveGameplayEffectHandle ActiveHandle, 
    int32 NewLevel
);
```
意思是说某些自定义计算如MMC使用非属性数据,参与计算的非属性数据更新时修改器不会自动更新,可以调用这个函数来触发更新

- 内部更新过程关键函数：
  

```
MarkItemDirty(Effect);

Effect.Spec.CalculateModifierMagnitudes();

UpdateAllAggregatorModMagnitudes(Effect);
```


- 当 **Backing Attribute** 更新时，基于其的 Modifier 会自动刷新
  

---

#### GameplayEffect 的应用流程

1. **UGameplayEffect 通常不实例化**
   
2. Ability 或 ASC 想应用 GE 时，会从 **ClassDefaultObject** 创建 **GameplayEffectSpec**
   
3. 成功应用后，GameplayEffectSpec 被封装为 **FActiveGameplayEffect**
   
4. ASC 通过名为 **ActiveGameplayEffect** 的特殊容器追踪所有活跃的 Effect

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ge-applying"></a>
#### 应用GameplayEffect

`GameplayEffect` 可以通过 [GameplayAbility](#concepts-ga) 或 `ASC`（Ability System Component）中的多种函数应用。通常，这些函数都是 `ApplyGameplayEffectTo...` 的形式，本质上最终都会在目标 `ASC` 上调用：

`UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf()`

#### 在 `GameplayAbility` 之外应用

如果你想在 `GameplayAbility` 之外应用 `GameplayEffect`（例如投掷物效果），需要先获取目标的 `ASC`，然后调用其相关函数来应用效果：

`TargetASC->ApplyGameplayEffectToSelf(...);`

#### 监听 `GameplayEffect` 的应用

绑定持续（Duration）或无限（Infinite）的 `GameplayEffect`应用 代理:

```C++
AbilitySystemComponent->OnActiveGameplayEffectAddedDelegateToSelf.AddUObject(
    this, 
    &APACharacterBase::OnActiveGameplayEffectAddedCallback
);
```

绑定Instance的 `GameplayEffect` 应用代理:
```
AbilitySystemComponent->OnPeriodicGameplayEffectExecuteDelegateOnSelf()
```
#### 回调调用规则

- **服务端（Server）**：无论同步模式如何，都会调用该回调。
  
- **自主代理（Autonomous Proxy）**：只有在 `Full` 或 `Mixed` 同步模式下，且针对同步的 `GameplayEffect` 时调用。
  
- **模拟代理（Simulated Proxy）**：仅在 `Full` 同步模式下调用。

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ga-removing"></a>
#### 移除GameplayEffect

`GameplayEffect` 可以通过 [GameplayAbility](#concepts-ga) 或 `ASC`（Ability System Component）中的多种函数移除。通常，这些函数都是 `RemoveActiveGameplayEffect` 的形式，本质上最终都会调用：

`FActiveGameplayEffectContainer::RemoveActiveEffects()`

#### 在 `GameplayAbility` 之外移除

如果你想在 `GameplayAbility` 之外移除 `GameplayEffect`，需要先获取目标的 `ASC`，然后调用其相关函数：

`TargetASC->RemoveActiveGameplayEffect(...);`

#### 监听 `GameplayEffect` 的移除

同上ASC中对应的函数来绑定GE的移除代理

**回调调用规则**

- **服务端（Server）**：无论同步模式如何，都会调用该回调。
  
- **自主代理（Autonomous Proxy）**：只有在 `Full` 或 `Mixed` 同步模式下，且针对可同步的 `GameplayEffect` 时调用。
  
- **模拟代理（Simulated Proxy）**：仅在 `Full` 同步模式下调用。

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ge-mods"></a>
#### GameplayEffectModifier


`Modifier` 是修改 `Attribute` 的唯一方法，并且是 **可以预测（Prediction）修改 Attribute** 的机制。

> 属性集中的宏还有GE中的执行最后输出修改器来修改属性


- 一个 `GameplayEffect` 可以包含 **0 个或多个 Modifier**。
  
- 每个 `Modifier` 只能通过指定的操作修改一个 `Attribute`。
  

##### Modifier 操作类型

|操作|描述|
|---|---|
|Add|将计算结果加到指定的 `Attribute` 上。使用负数即可实现减法操作。|
|Multiply|将指定 `Attribute` 与计算结果相乘。|
|Divide|将指定 `Attribute` 除以计算结果。|
|Override|使用计算结果覆盖指定 `Attribute` 的值（优先于最后应用的 Modifier）。|

#### Attribute 值计算

`Attribute` 的 `CurrentValue` 是其 **所有 Modifier 与 BaseValue 的总合结果**。  
计算公式在 `GameplayEffectAggregator.cpp` 中的 `FAggregatorModChannel::EvaluateWithBase` 定义如下：

`((InlineBaseValue + Additive) * Multiplicitive) / Division`

> **注意**
> 
> - 对于百分比修改，推荐使用 `Multiply`，以确保在加法操作之后生效。
>     
> - [预测（Prediction）](#concepts-p) 对百分比修改存在一些问题。
>     

---

#### Modifier 类型

Modifier 可以分为四种类型，它们都会生成浮点数结果用于修改指定 Attribute：

| Modifier 类型                  | 描述                                                                                                                                                                                                                                                                                                                          |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Scalable Float**           | 基于 `FScalableFloat` 结构体，可指向 Data Table 中某行值，支持按 Ability 等级读取。若未指定 Data Table/Row，默认值为 1。可进一步乘系数进行调整。                                                                                                                                                                                                                        |
| **Attribute Based**          | 使用 Source（创建者）或 Target（接收者）上的 `CurrentValue` 或 `BaseValue` 作为基础值。可应用系数和 Pre/Post 加权。`Snapshotting` 表示在创建 `GameplayEffectSpec` 时捕获 Attribute；`No Snapshotting` 表示在应用时捕获。                                                                                                                                                     |
| **Custom Calculation Class** | 适用于复杂计算，通过 `ModifierMagnitudeCalculation` 类实现，可使用系数和 Pre/Post 加权修改浮点值结果。                                                                                                                                                                                                                                                    |
| **Set By Caller**            | 运行时由 Ability 或 `GameplayEffectSpec` 创建者设置的值。例如：根据蓄力技能长短动态设置伤害。实现为 `GameplayEffectSpec` 中的 `TMap<FGameplayTag, float>`，Modifier 会查找对应 `GameplayTag` 的值。  <br>⚠️ 如果 `SetByCaller` 的值不存在，游戏会抛出运行时错误并返回 0，这可能在 `Divide` 操作中造成问题。  <br>`FName` 形式不可用，只能使用 `GameplayTag`。参见 [SetByCallers](#concepts-ge-spec-setbycaller) 获取更多信息。 |
**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ge-mods-multiplydivide"></a>
##### Multiply和Divide Modifier

##### 公式实现

>我还没看过,这里认为Issue是正确的,我的看法是复杂计算用MMC或者执行


多个对于同一Attribute的Multiply/Divide Modifier时，GAS对于Instant GE，和Duration | Infinite GE 的处理不一样。  
比方说，连续两个针对同一个属性的Multiply Modifier，系数都是1.5，那么：

- Instant GE，最终效果是 `1.5 * 1.5 = 2.25`
- Duration | Infinite GE，最终效果是 `1 + (1.5 - 1) + (1.5 - 1) = 2`  
    持续时间 | 无限 GE，最终效果是 `1 + (1.5 - 1) + (1.5 - 1) = 2`

原因是：  
对于Instant GE，会使用`UAbilitySystemComponent::ExecuteGameplayEffect`函数来计算，会通过遍历Modifier来迭代计算其对于属性的影响。

```c
// FActiveGameplayEffectsContainer::ExecuteActiveEffectsFrom() 函数
for (int32 ModIdx = 0; ModIdx < SpecToUse.Modifiers.Num(); ++ModIdx)
{
	const FGameplayModifierInfo& ModDef = SpecToUse.Def->Modifiers[ModIdx];

	// ... 省略
		
	FGameplayModifierEvaluatedData EvalData(ModDef.Attribute, ModDef.ModifierOp, SpecToUse.GetModifierMagnitude(ModIdx, true));
	ModifierSuccessfullyExecuted |= InternalExecuteMod(SpecToUse, EvalData);
}
```

而对于Duration | Infinite GE，则会通过`UAbilitySystemComponent::OnAttributeAggregatorDirty`函数来响应其对属性的影响，而这个函数会调用到`EvaluateWithBase()`函数，通过`SumMods`预计算所有`Multiply Modifier`的系数和，来计算。

```c
// FAggregatorModChannel::EvaluateWithBase() 函数
float Additive = SumMods(Mods[EGameplayModOp::Additive], GameplayEffectUtilities::GetModifierBiasByModifierOp(EGameplayModOp::Additive), Parameters);
float Multiplicitive = SumMods(Mods[EGameplayModOp::Multiplicitive], GameplayEffectUtilities::GetModifierBiasByModifierOp(EGameplayModOp::Multiplicitive), Parameters);
float Division = SumMods(Mods[EGameplayModOp::Division], GameplayEffectUtilities::GetModifierBiasByModifierOp(EGameplayModOp::Division), Parameters);

if (FMath::IsNearlyZero(Division))
{
	ABILITY_LOG(Warning, TEXT("Division summation was 0.0f in FAggregatorModChannel."));
	Division = 1.f;
}

return ((InlineBaseValue + Additive) * Multiplicitive) / Division;
```

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ge-mods-gameplaytags"></a>
##### Modifier的GameplayTag

每个 [Modifier](#concepts-ge-mods) 都可以设置 **SourceTag** 和 **TargetTag**。  
它们的作用类似于 `GameplayEffect` 的 [Application Tag Requirements](#concepts-ge-tags)：也就是应用修改器所需的标签要求

- 只有在 **Effect 应用后** 才会检查这些标签。
  
- 对于 **周期性（Periodic）的无限（Infinite）Effect(持续性也是?)**，这些标签 **仅在第一次应用时检查**，不会在每次 Tick 时重新判断。
  

---

#### Attribute Based Modifier 的额外过滤器

`Attribute Based Modifier` 还可以设置 **SourceTagFilter** 和 **TargetTagFilter**：

- 作用：在计算要捕获属性 的 Magnitude 时，过滤掉不符合条件的 Modifier。
  
- 规则：如果源或目标中 **缺少过滤器要求的全部标签**，则该 Modifier 会被排除。
  

---

#### 修改器标签捕获时机

- **Source ASC 标签**：在创建 `GameplayEffectSpec` 时捕获。
  
- **Target ASC 标签**：在 Effect 实际应用时捕获。
  

当需要判断一个 **Infinite** 或 **Duration** 类型的 Effect 的 Modifier 是否满足条件（即 **Aggregator Qualify**）时，如果设置了过滤器，则会使用这些捕获的标签进行比对。(Instance的也会判断只是不放入聚合器?)

---

这样整理后，层次更清晰：

- 普通 Modifier 的 Source/TargetTag → 类似 GE Application Tag,即应用修改器的标签需求
  
- Attribute Based Modifier → 额外多了 Source/TargetTagFilter，用于计算基于的属性的 Magnitude 时修改器的过滤
  
- 标签捕获时机区分 Source 和 Target

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ge-stacking"></a>
#### 堆栈

- **默认行为**：  
    每次应用一个 `GameplayEffect` 时，最后都是一个新的 `GameplayEffectSpec` 实例并应用到目标上。它不关心目标上是否已经存在同类型的 `Spec`。
    
- **堆栈化（Stacking）例外**：  
    如果 `GameplayEffect` 的配置启用了 **Stacking**，那么再次应用时不会产生一个新的 `Spec` 实例，而是修改已有实例的 **堆栈数（Stack Count）**。
    
    - 堆栈数的变化会触发相关事件（比如 HUD 更新）。
      
    - 堆栈化只适用于 **Duration** 和 **Infinite** 类型的 `GameplayEffect`。
      
    - **Instant GE** 永远不会堆叠，因为它没有挂在 ASC 上的生命周期。
      

---

###### 堆栈类型

|堆栈类型|说明|
|---|---|
|**Aggregate by Source**|目标 ASC 上为 **每个源 ASC** 建立独立的堆栈。一个 Source 可以在该堆栈中累积 X 层。不同 Source 的堆栈互不影响。|
|**Aggregate by Target**|目标 ASC 上只有一个全局堆栈，**不区分 Source**。所有来源共享一个堆栈上限（Shared Stack Limit）。|

---

###### 堆栈的生命周期与刷新策略

- 堆栈和 `GameplayEffect` 的持续时间绑定在一起，受以下机制影响：
  
    1. **过期策略（Expiration Policy）**
       
        - 堆栈到期时是完全移除还是逐层减少。
        
    2. **持续 刷新策略（Duration Refresh Policy）**
       
        - 新的应用是否刷新持续时间。
        
    3. **周期性 重置策略（Period Reset Policy）**
       
        - 周期性执行是否重新计时。
          

这些策略在 **GameplayEffect 蓝图详情面板** 里都有浮动提示，非常直观。



- AsyncTask 最后要调用 **EndTask()** 来回收，比如在 UMG Widget 的 `Destruct` 事件中调用以避免泄露。
  
    

---

⚠️ 容易混淆的点：

- **堆栈 ≠ 多个 Spec 并存**。一旦开启堆叠，目标上只会有 **一个 GE Spec 实例**，堆栈数变化只是修改这个实例里的 `StackCount` 字段。
  
- 如果 **没有启用堆栈**，哪怕你应用 10 次，目标 ASC 上也会出现 10 个独立的 GE Spec 实例（每个都有自己的 Duration、Periodic Tick 等）



**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ge-ga"></a>
#### 授予Ability

`GameplayEffect`可以授予(Grant)新的[GameplayAbility](#concepts-ga)到`ASC`. 只有`持续(Duration)`和`无限(Infinite)GameplayEffect`可以授予Ability.  

一个普遍用法是当想要强制另一个玩家做某些事的时候, 像击退或拉取时移动他们, 就会对它们应用一个`GameplayEffect`来授予其一个自动激活的Ability 从而使其做出相应的动作.

>重写GA的OnGiveAbility()函数来调用激活函数实现授予即激活的被动能力

>GE激活后授予的GA能获得GE的`TMap<FGameplayTag, float> SetByCallerTagMagnitudes;`

设计师可以决定一个`GameplayEffect`能够授予哪些Ability, 授予的Ability等级, 将其绑定在什么输入键上以及该Ability的移除策略.  
![[attachments/Pasted image 20250818224223.png|545]]

|     移除策略     |                                 描述                                  |
| :----------: | :-----------------------------------------------------------------: |
| 立即取消Ability  |       当授予Ability的`GameplayEffect`从目标移除时, 授予的Ability就会立即取消并移除.       |
| 结束时移除Ability |            允许授予的Ability完成(也就是调用EndAbility后), 之后将其从目标移除.             |
|      无       | 授予的Ability不受从目标移除的授予`GameplayEffect`的影响, 目标将会一直拥有该Ability直到之后被手动移除. |

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ge-tags"></a>
#### GameplayEffect组件
>编辑器有详细的提示

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ge-spec"></a>
#### GameplayEffectSpec
游戏效果规格,简称GESpec

GameplayEffectSpec (GESpec) 可以理解为 **`GameplayEffect` 的实例**。  
它保存了：

- 对应的 `GameplayEffect` 类引用
  
- 创建时的等级与创建者信息

>这里的实例指的是运行时类,也就是专们存储GE静态信息并附带一些额外运行时信息

与 `GameplayEffect`（设计时由设计师预先创建）不同，**`GameplayEffectSpec` 可以在运行时动态创建和修改**。  
在应用 `GameplayEffect` 时，系统会基于该 `GameplayEffect` 生成一个 `GameplayEffectSpec`，并将其真正作用于目标 (Target)。

---

#### 创建

- `GameplayEffectSpec` 通过
  
    `UAbilitySystemComponent::MakeOutgoingSpec()`
    
    创建（BlueprintCallable）。
    
- 它 **不必立即应用**，常见做法是将其传递给由 Ability 生成的投掷物 (Projectile)，在击中目标后再应用。
  
- 当 `GameplayEffectSpec` 成功应用时，会返回一个新的 **`FActiveGameplayEffect`** 结构体，表示正在生效的效果。
  

---

#### `GameplayEffectSpec` 关键内容

- **源 GameplayEffect 类**：该 `Spec` 基于哪个 `GameplayEffect` 创建。
  
- **等级 (Level)**：默认与创建它的 Ability 等级一致，也可自定义。
  
- **持续时间 (Duration)**：默认取自 `GameplayEffect`，但可修改。
  
- **周期 (Period)**：用于周期性效果，默认取自 `GameplayEffect`，也可修改。
  
- **堆栈数 (Stack Count)**：当前堆叠层数，最大堆叠上限由 `GameplayEffect` 决定,可以修改
  
- **`GameplayEffectContextHandle`**：GE上下文句柄
  
- **标签系统**
- `CapturedSourceTags`:创建GE时捕获和源标签 ;

-  `CapturedTargetTags`: 应用GE时捕获的目标标签;

- `CapturedRelevantAttributes` :与自定义计算相关的标签,如MMC和执行 ;

    - `DynamicGrantedTags`：在目标上额外授予的标签（区别于 `GameplayEffect`组件的 Granted Tags）。
      
    - `DynamicAssetTags`：额外添加的 Asset 标签（区别于 `GameplayEffect`组件的 Asset Tags）。
    
- SetByCallerTagMagnitudes：允许在运行时由调用者动态指定标签和对应数值。

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ge-spec-setbycaller"></a>
##### SetByCallerMagnitudes

`SetByCallerMagnitudes` 允许 **`GameplayEffectSpec`** 携带 **`GameplayTag`** 和 FName对应的数值。  
这些数值存储在：GEspec的


- `SetByCallerTagMagnitudes`
- `SetByCallerNameMagnitudes`
  
    

它们可以作为 **`GameplayEffect` 的 Modifier 输入**，或者作为一种通用的数据传递方式。  
常见用法是：Ability 在运行时计算出数值，通过 `SetByCaller` 传递到 **`GameplayEffectExecutionCalculations` (Execution Calculation)** 或 **`ModifierMagnitudeCalculations` (MMC)** 中。

---

**使用方式**

| 用途           | 说明                                                                                                                           |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| **Modifier** | 必须 **提前在 `GameplayEffect` 中定义**，且只能使用 `GameplayTag` 形式。  <br>如果定义了但 `GameplayEffectSpec` 没有对应的值，应用时会报错并返回 `0`。这在除法运算中可能引发错误。 |
| **其他用途**     | 无需提前定义。读取时如果 `GameplayEffectSpec` 中不存在对应的键，将返回一个默认值，并可选择是否输出警告。                                                              |

C++通过GEspec读取,蓝图通过对应函数库

---

**建议**

强烈推荐使用 **`GameplayTag`** 形式而不是 `FName`：

- 避免蓝图中因拼写错误导致的运行时问题。
  
- 在网络同步时更高效（`GameplayTag` 的传输比 `FName` 更紧凑，且 `TMap` 会自动同步）。

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ge-context"></a>

#### GameplayEffectContext

 GameplayEffectContext是一个结构体，用于存储关于 **`GameplayEffectSpec`应用过程中的各种数据 和 目标数据（TargetData）** 的信息。

它也是一个很好的 **可继承结构体**，可以在以下系统间传递任意数据：

- **ModifierMagnitudeCalculation (MMC)**
  
- **GameplayEffectExecutionCalculation (Execution)**
  
- **AttributeSet**
  
- **GameplayCue**
  

---

##### 自定义GameplayEffectContext

如果你想在 `GameplayEffectContext` 中添加自定义数据，可以按以下步骤操作：

1. **继承** `FGameplayEffectContext`。
   
2. **重写** `FGameplayEffectContext::GetScriptStruct()`。
   
3. **重写** `FGameplayEffectContext::Duplicate()`。
   
4. **同步数据**（如有必要），重写 `FGameplayEffectContext::NetSerialize()`。
   
5. 对结构体实现 **`TStructOpsTypeTraits`**，与父结构体相同。
   
6. 在 **`AbilitySystemGlobals`** 中重写 `AllocGameplayEffectContext()`，返回新的子结构体对象。
   

---


- 例如：霰弹枪的攻击可以击中多个敌人，每个目标的数据都可以通过这个上下文结构体传递到 `GameplayCue`，便于播放特效或执行逻辑。 

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ge-mmc"></a>
#### ModifierMagnitudeCalculation

> 修改器幅度计算,简称MMC

ModifierMagnitudeCalculation是一种可以在 `GameplayEffect` 的 Modifier中使用的计算类。它的功能类似于 GameplayEffectExecutionCalculation，但更**简单、可预测**：  
其唯一职责就是在 `CalculateBaseMagnitude_Implementation()` 中返回一个浮点数。该函数可以通过 **C++** 或 **蓝图** 继承并重写。

##### 适用范围

MMC 可用于所有类型的 `GameplayEffect`：

- 即刻 (Instant)
  
- 持续 (Duration)
  
- 无限 (Infinite)
  
- 周期性 (Periodic)
  

##### 特点与优势

- 可以完全访问 `GameplayEffectSpec`，从而读取：
  
    - **GameplayTag**
      
    - **SetByCaller 值**
    
- 可以捕获来自 **源 (Source)** 或 **目标 (Target)** 的任意数量的 `Attribute`。
  
- 支持 **快照 (Snapshot)** 与 **非快照** 捕获方式：
  
    - **快照属性**：在 `GameplayEffectSpec` **创建时**捕获Source和 **应用时**捕获Target，不会随着 Attribute 的后续变化而更新。
      
    - **非快照属性**：在 `应用时` 捕获，会随着持续或无限 GE 修改时自动更新。
      

|是否快照|来源|捕获时机|是否随 GE 修改自动更新|
|---|---|---|---|
|是|Source|Spec 创建时|否|
|是|Target|Spec 应用时|否|
|否|Source|Spec 应用时|是|
|否|Target|Spec 应用时|是|

> ⚠️ 注意：MMC 捕获 Attribute 时，取值来自 ASC 内部的 **聚合计算结果 (CurrentValue)**。此过程不会触发 `AttributeSet::PreAttributeChange()`，因此若有 **数值限制（Clamp）** 逻辑，需要在 MMC 内重新实现。

##### 结果修正

MMC 返回的浮点值，最终还会受到 `GameplayEffect` Modifier 中的 **系数与前后修正因子** 影响。

---

##### 示例：基于目标魔法值的中毒效果

下面的例子展示了一个 **PoisonMana MMC**：

- 捕获目标的 `Mana` 和 `MaxMana`
  
- 根据目标魔法比例与标签调整最终减少量
  

```C++
UPAMMC_PoisonMana::UPAMMC_PoisonMana()
{
    // 捕获 Mana
    ManaDef.AttributeToCapture = UPAAttributeSetBase::GetManaAttribute();
    ManaDef.AttributeSource = EGameplayEffectAttributeCaptureSource::Target;
    ManaDef.bSnapshot = false;

    // 捕获 MaxMana
    MaxManaDef.AttributeToCapture = UPAAttributeSetBase::GetMaxManaAttribute();
    MaxManaDef.AttributeSource = EGameplayEffectAttributeCaptureSource::Target;
    MaxManaDef.bSnapshot = false;

    RelevantAttributesToCapture.Add(ManaDef);
    RelevantAttributesToCapture.Add(MaxManaDef);
}

float UPAMMC_PoisonMana::CalculateBaseMagnitude_Implementation(const FGameplayEffectSpec& Spec) const
{
    const FGameplayTagContainer* SourceTags = Spec.CapturedSourceTags.GetAggregatedTags();
    const FGameplayTagContainer* TargetTags = Spec.CapturedTargetTags.GetAggregatedTags();

    FAggregatorEvaluateParameters Params;
    Params.SourceTags = SourceTags;
    Params.TargetTags = TargetTags;

    float Mana = 0.f;
    GetCapturedAttributeMagnitude(ManaDef, Spec, Params, Mana);
    Mana = FMath::Max(Mana, 0.0f);

    float MaxMana = 0.f;
    GetCapturedAttributeMagnitude(MaxManaDef, Spec, Params, MaxMana);
    MaxMana = FMath::Max(MaxMana, 1.0f); // 避免除零

    float Reduction = -20.0f;
    if (Mana / MaxMana > 0.5f)
    {
        Reduction *= 2; // 如果魔法超过一半，效果加倍
    }
    
    if (TargetTags->HasTagExact(FGameplayTag::RequestGameplayTag(FName("Status.WeakToPoisonMana"))))
    {
        Reduction *= 2; // 如果目标有易中毒标签，效果加倍
    }
    
    return Reduction;
}
```


##### 使用要点

- 若尝试捕获 Attribute 却未在构造函数中加入 `RelevantAttributesToCapture`，会报缺失错误。
  
- 如果 MMC 只依赖 Tag 或 SetByCaller 值（无需捕获 Attribute），则无需填充 `RelevantAttributesToCapture`。

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ge-ec"></a>
#### GameplayEffectExecutionCalculation (ExecCalc)
>消息效果执行计算,也就是执行


GameplayEffectExecutionCalculation（简称 **ExecutionCalculation**、**Execution** 或 **ExecCalc**）是 `GameplayEffect` 修改 `属性` 最强大的方式。

与 ModifierMagnitudeCalculation (MMC)类似，它也可以捕获 `Attribute` 并选择性地创建 **快照 (Snapshot)**。但不同的是：

- **MMC** 只能返回一个浮点值，通常用于修改单一 `Attribute`。
  
- **ExecCalc** 可以一次性修改 **多个 Attribute**，并能够执行任意复杂逻辑。
  

正因为这种强大与灵活，**ExecCalc 不具备可预测性 (Non-Predictable)**，并且只能在 **C++** 中实现。

---

##### 使用范围

ExecCalc 只能由以下类型的 `GameplayEffect` 使用：

- 即刻 (Instant)
  
- 周期性 (Periodic)
  

插件代码中凡是涉及 “Execute” 的，一般都指这两种 GE 类型。

---

##### Attribute 捕获机制

与 MMC 相同，ExecCalc 在捕获 Attribute 时也分为 **快照** 与 **非快照**：

- **快照 (Snapshot)**：
  
    - Source → 在 **Spec 创建** 时捕获
      
    - Target → 在 **Spec 应用** 时捕获
    
- **非快照**：
  
    - Source → 在 **Spec 应用** 时捕获
      
    - Target → 在 **Spec 应用** 时捕获
      

捕获 Attribute 时，取值来自 ASC 内部的 **CurrentValue**（即应用了 Modifier 的聚合值）。  
⚠️ 注意：这不会触发 `AttributeSet::PreAttributeChange()`，因此任何 **Clamp 或数值限制** 都需要在 ExecCalc 中自行处理。

---

##### Attribute 捕获设置方式
在执行的源文件声明如下

```C++
// 自定义结构体  
struct FDamageStatics  
{  
	// 声明捕获定义
    FGameplayEffectAttributeCaptureDefinition SourceLevelDef;  
    FGameplayEffectAttributeCaptureDefinition TargetLevelDef;  
  
  
    // 在结构体构造函数中初始化捕获定义  
    FDamageStatics()  
    {       
    SourceLevelDef = FGameplayEffectAttributeCaptureDefinition(  
	    UAuraAttributeSet::GetLevelAttribute(),  
	    EGameplayEffectAttributeCaptureSource::Source,  
	    false  
	);  
	
	TargetLevelDef = FGameplayEffectAttributeCaptureDefinition(  
	    UAuraAttributeSet::GetLevelAttribute(),  
	    EGameplayEffectAttributeCaptureSource::Target,  
	    false  
	);  
};
```

```C++
// 单例访问器  
// 静态文件函数  
static const FDamageStatics& DamageStatics()  
{  
    // 静态局部变量，确保只创建一次  
    static FDamageStatics DamageStatics;  
  
    return DamageStatics;  
}
```

```在执行的构造函数中添加捕获的定义
UAuraDamageExecution::UAuraDamageExecution()  
{  
    RelevantAttributesToCapture.Add(DamageStatics().SourceLevelDef);  
    RelevantAttributesToCapture.Add(DamageStatics().TargetLevelDef);   
}
```
> ⚠️ 每个结构体必须有 **唯一名称**，因为它们共享命名空间。若多个结构体重名，可能导致捕获到错误的 Attribute 值。

---

##### 网络调用规则

对于以下几种 **GameplayAbility**：

- Local Predicted
  
- Server Only
  
- Server Initiated
  

`ExecCalc` **只会在服务端调用**。

---

##### 应用场景

ExecCalc 最常见的用途是实现一个复杂的数值公式：

- 从 **Source** 和 **Target** 捕获多个 Attribute（如攻击力、防御、暴击率等）。
  
- 从 **Spec 的 SetByCaller** 中读取额外参数（如技能配置的伤害）。
  
- 综合计算最终的结果，再修改目标的多个 Attribute。
  

例如 ActionRPG 示例项目中的 **GDDamageExecCalculation**：

- 从 `Spec.SetByCaller` 中读取伤害值。
  
- 根据目标的护盾 Attribute 进行伤害削减。

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ge-ec-senddata"></a>
##### 发送数据到Execution Calculation

除了捕获`Attribute`, 还有几种方法可以发送数据到`ExecutionCalculation`.  


<a name="concepts-ge-ec-senddata-setbycaller"></a>
`ExecutionCalculation` 中常见的数据传递方式有以下几种：

- **SetByCaller**
  
    - 任何写入到 `GameplayEffectSpec` 的 SetByCaller 值可直接读取：
      
```C++
const FGameplayEffectSpec& Spec = ExecutionParams.GetOwningSpec();
float Damage = FMath::Max<float>(
    Spec.GetSetByCallerMagnitude(
        FGameplayTag::RequestGameplayTag(FName("Data.Damage")),
        false, -1.0f
    ),
    0.0f
);
```


- CalculationModifier
  >计算修改器
  
    - 在 `GameplayEffect` 中硬编码值，并通过捕获的 Attribute 作为 支持数据传递。
      
    - 例如 GE 中设置 `Armor +50`，ExecCalc 中读取：
      
```C++
float Armor = 0.0f;
ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(
    DamageStatics().ArmorDef,
    EvaluationParameters,
    Armor
);
```

    也可设为 Override，直接传固定值。

- Transient Aggregator
  
    - 使用临时变量与 `GameplayTag` 绑定，在 ExecCalc 中读取。
      
    - 示例：为 `Data.Damage` 添加 50：
      
```C++
    ValidTransientAggregatorIdentifiers.AddTag(
	    FGameplayTag::RequestGameplayTag("Data.Damage")
	);
	float Damage = 0.0f;
	ExecutionParams.AttemptCalculateTransientAggregatorMagnitude(
	    FGameplayTag::RequestGameplayTag("Data.Damage"),
	    EvaluationParameters,
	    Damage
	);
```


- **GameplayEffectContext**
  
    - 可通过 `GameplayEffectSpec` 的自定义 Context 传递数据：
      
```C++
const FGameplayEffectSpec& Spec = ExecutionParams.GetOwningSpec();
FGSGameplayEffectContext* ContextHandle =
    static_cast<FGSGameplayEffectContext*>(Spec.GetContext().Get());
```


- 若需修改 `Spec` 或 Context，可使用：
        
```C++
FGameplayEffectSpec* MutableSpec = ExecutionParams.GetOwningSpecForPreExecuteMod();
FGSGameplayEffectContext* ContextHandle =
    static_cast<FGSGameplayEffectContext*>(MutableSpec->GetContext().Get());
```

        ⚠️ 注意：`GetOwningSpecForPreExecuteMod()` 允许非 const 修改，捕获 Attribute 后修改需格外小心。


**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ge-car"></a>
#### 自定义应用需求组件

`CustomApplicationRequirement(CAR)`类为设计师提供对于`GameplayEffect`是否可以应用的高阶控制, 而不是对`GameplayEffect`进行简单的`GameplayTag`检查. 这可以通过在蓝图中重写`CanApplyGameplayEffect()`和在C++中重写`CanApplyGameplayEffect_Implementation()`实现.  

`CAR`的应用场景:  

* 目标需要有一定数量的`Attribute`.
* 目标需要有一定数量的`GameplayEffect`堆栈.

`CAR`还有很多高阶功能, 像检查`GameplayEffect`实例是否已经位于目标上, 修改当前实例的[持续时间](#concepts-ge-duration)而不是应用一个新实例(对于`CanApplyGameplayEffect()`返回false).

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ge-cost"></a>
#### CostGameplayEffect
>花销

- `GameplayAbility`可以选择指定一个专门用于Ability花销（Cost）的`GameplayEffect`。
    
- **花销(Cost)**：指`ASC`激活`GameplayAbility`所必需消耗的`Attribute`值。
    
- 如果某个`GA`没有提供`Cost GE`，它将无法被激活。
    
- **Cost GE设计**：
    
    - 应该是一个即刻(Instant)的`GameplayEffect`。
        
    - 包含对一个或多个自`Attribute`的减值Modifier。
        
    - 默认情况下，Cost GE支持预测(predictable)，建议保持这一特性。
        
    - 对于复杂花销计算，推荐使用**Modifier Magnitude Calculation(MMC)而非`ExecutionCalculations`。
        
- **使用策略**：
    
    - 初期做法：为每个带花费的`GA`创建独一无二的Cost GE。
        
    - 高阶技巧：对多个GA复用一个Cost GE，通过修改在`GA`上定义的花费值来更新`GameplayEffectSpec`，仅适用于**实例化(Instanced)Ability**。
        
- **复用Cost GE的两种方法**：
    
    1. **使用MMC**：
        
        - 创建一个MMC，从`GameplayAbility`实例读取花销值。
            
        - 示例代码：
            
```C++
float UPGMMC_HeroAbilityCost::CalculateBaseMagnitude_Implementation(const FGameplayEffectSpec & Spec) const
{
    const UPGGameplayAbility* Ability = Cast<UPGGameplayAbility(Spec.GetContext().GetAbilityInstance_NotReplicated());
    if (!Ability) return 0.0f;
    return Ability->Cost.GetValueAtLevel(Ability->GetAbilityLevel());
}
```

            
GameplayAbility子类中花费属性示例：

```C++
UPROPERTY(BlueprintReadOnly, EditAnywhere, Category = "Cost") FScalableFloat Cost;
```

 2. 重写`UGameplayAbility::GetCostGameplayEffect()`：
在运行时创建一个读取GameplayAbility中花销值的GameplayEffect。

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ge-cooldown"></a>
#### CooldownGameplayEffect

- `GameplayAbility`可以选择指定一个专门用于冷却(Cooldown)的`GameplayEffect`。
    
- **冷却功能**：
    
    - 决定激活Ability后多久可以再次激活。
        
    - 如果某个GA在冷却中，它将无法被激活。
        
    - `Cooldown GE`通常是**不带Modifier的持续(Duration)GameplayEffect**。
        
    - GA通过检查GE的组件中授予目标的标签来判断是否存在来判断冷却，而非直接检查`Cooldown GE`。
        
    - 默认情况下，Cooldown GE是可预测的，建议保持预测功能。
        
    - 对于复杂冷却计算，可使用**MMC**而非`ExecutionCalculations`。
        
- **使用策略**：
    
    - 初期做法：为每个带冷却的GA创建独一无二的Cooldown GE。
        
    - 高阶技巧：对多个GA复用一个Cooldown GE，通过修改`GameplayEffectSpec`中冷却时间和Cooldown Tag来实现，仅适用于**实例化(Instanced)Ability**。
        
- **复用Cooldown GE的两种方法**：
    1. **使用SetByCaller**：
        
        - 最简单方式：通过`GameplayTag`设置`SetByCaller`为共享Cooldown GE的持续时间。
            
        - 在GameplayAbility子类中定义：
            
```C++
UPROPERTY(BlueprintReadOnly, EditAnywhere, Category = "Cooldown")
FScalableFloat CooldownDuration;

UPROPERTY(BlueprintReadOnly, EditAnywhere, Category = "Cooldown")
FGameplayTagContainer CooldownTags;

// 临时容器，用于返回GetCooldownTags()的并集
UPROPERTY()
FGameplayTagContainer TempCooldownTags;
```

- 重写`GetCooldownTags()`返回Cooldown Tag与现有Cooldown GE标签的并集：
            
```C++
const FGameplayTagContainer* UPGGameplayAbility::GetCooldownTags() const
{
    FGameplayTagContainer* MutableTags = const_cast<FGameplayTagContainer*>(&TempCooldownTags);
    const FGameplayTagContainer* ParentTags = Super::GetCooldownTags();
    if (ParentTags)
    {
        MutableTags->AppendTags(*ParentTags);
    }
    MutableTags->AppendTags(CooldownTags);
    return MutableTags;
}
```

- 重写`ApplyCooldown()`，注入Cooldown Tag并通过SetByCaller设置持续时间
```C++
void UPGGameplayAbility::ApplyCooldown(const FGameplayAbilitySpecHandle Handle,
                                       const FGameplayAbilityActorInfo* ActorInfo,
                                       const FGameplayAbilityActivationInfo ActivationInfo) const
{
    UGameplayEffect* CooldownGE = GetCooldownGameplayEffect();
    if (CooldownGE)
    {
        FGameplayEffectSpecHandle SpecHandle = MakeOutgoingGameplayEffectSpec(CooldownGE->GetClass(),          GetAbilityLevel());
        SpecHandle.Data.Get()->DynamicGrantedTags.AppendTags(CooldownTags);
        SpecHandle.Data.Get()->SetSetByCallerMagnitude(
            FGameplayTag::RequestGameplayTag(FName("OurSetByCallerTag")),
            CooldownDuration.GetValueAtLevel(GetAbilityLevel())
        );
        ApplyGameplayEffectSpecToOwner(Handle, ActorInfo, ActivationInfo, SpecHandle);
    }
}
```

1. **使用MMC**：
        
    - 设置与SetByCaller类似，但无需在Cooldown GE中设置SetByCaller。
            
        - 将持续时间设置为**Custom Calculation类**，并指向新创建的MMC。
            
        - 属性与TempCooldownTags同上。
            
        - `GetCooldownTags()`和`ApplyCooldown()`重写逻辑同上，不再设置SetByCaller：
            
```C++
const FGameplayTagContainer* UPGGameplayAbility::GetCooldownTags() const
{
    FGameplayTagContainer* MutableTags = const_cast<FGameplayTagContainer*>(&TempCooldownTags);
    const FGameplayTagContainer* ParentTags = Super::GetCooldownTags();
    if (ParentTags)
    {
        MutableTags->AppendTags(*ParentTags);
    }
    MutableTags->AppendTags(CooldownTags);
    return MutableTags;
}
```

            
```C++
float UPGMMC_HeroAbilityCooldown::CalculateBaseMagnitude_Implementation(const FGameplayEffectSpec & Spec) const
{
    const UPGGameplayAbility* Ability = Cast<UPGGameplayAbility>(Spec.GetContext().GetAbilityInstance_NotReplicated());
    if (!Ability) return 0.0f;
    return Ability->CooldownDuration.GetValueAtLevel(Ability->GetAbilityLevel());
}
```



**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ge-cooldown-tr"></a>
##### 获取Cooldown GameplayEffect的剩余时间

```c++
bool APGPlayerState::GetCooldownRemainingForTag(FGameplayTagContainer CooldownTags, float & TimeRemaining, float & CooldownDuration)
{
	if (AbilitySystemComponent && CooldownTags.Num() > 0)
	{
		TimeRemaining = 0.f;
		CooldownDuration = 0.f;

		FGameplayEffectQuery const Query = FGameplayEffectQuery::MakeQuery_MatchAnyOwningTags(CooldownTags);
		TArray< TPair<float, float> > DurationAndTimeRemaining = AbilitySystemComponent->GetActiveEffectsTimeRemainingAndDuration(Query);
		if (DurationAndTimeRemaining.Num() > 0)
		{
			int32 BestIdx = 0;
			float LongestTime = DurationAndTimeRemaining[0].Key;
			for (int32 Idx = 1; Idx < DurationAndTimeRemaining.Num(); ++Idx)
			{
				if (DurationAndTimeRemaining[Idx].Key > LongestTime)
				{
					LongestTime = DurationAndTimeRemaining[Idx].Key;
					BestIdx = Idx;
				}
			}

			TimeRemaining = DurationAndTimeRemaining[BestIdx].Key;
			CooldownDuration = DurationAndTimeRemaining[BestIdx].Value;

			return true;
		}
	}

	return false;
}
```

**Note:** 在客户端上查询剩余冷却时间要求其可以接收同步的`GameplayEffect`, 这依赖于它们`ASC`的[同步模式](#concepts-asc-rm).  

**[⬆ 返回目录](#table-of-contents)**

 <a name="concepts-ge-cooldown-listen"></a>
##### 监听冷却开始和结束

- **监听冷却开始**：
    
    - 可以通过以下两种方式：
        
        1. 绑定 `AbilitySystemComponent->OnActiveGameplayEffectAddedDelegateToSelf` → 当 `Cooldown GE` 被应用时触发。
            
        2. 绑定 `AbilitySystemComponent->RegisterGameplayTagEvent(CooldownTag, EGameplayTagEventType::NewOrRemoved)` → 当 `Cooldown Tag` 被添加时触发。
            
    - **推荐方法**：监听 `Cooldown GE` 应用。
        
        - 优点：
            
            - 可访问触发它的 `GameplayEffectSpec`。
                
            - 可以判断当前Cooldown GE是客户端预测的还是服务端校正的。
                
- **监听冷却结束**：
    
    - 可以通过以下两种方式：
        
        1. 绑定 `AbilitySystemComponent->OnAnyGameplayEffectRemovedDelegate()` → 当 `Cooldown GE` 被移除时触发。
            
        2. 绑定 `AbilitySystemComponent->RegisterGameplayTagEvent(CooldownTag, EGameplayTagEventType::NewOrRemoved)` → 当 `Cooldown Tag` 被移除时触发。
            
    - **推荐方法**：监听 `Cooldown Tag` 移除。
        
        - 原因：
            
            - 服务端校正的 `Cooldown GE` 到来时，会移除客户端预测的 `Cooldown GE`。
                
            - 直接监听 `OnAnyGameplayEffectRemovedDelegate()` 会误触，即使技能仍在冷却中。
                
            - 预测的 `Cooldown GE` 移除时，`Cooldown Tag` 不会改变。
                
            - 服务端校正的 `Cooldown GE` 应用时，`Cooldown Tag` 才会正确更新。
                
- **注意事项**：
    
    - 客户端监听某个 `GameplayEffect` 添加或移除，要求`GameplayEffect`为复制的
        
    - 这依赖于其 `ASC` 的同步模式 ([Replication Mode](#concepts-asc-rm))。
  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ge-cooldown-prediction"></a>
##### 预测冷却时间

- **客户端预测冷却**：
    
    - 当客户端预测的 `Cooldown GE` 被应用时，可以立即启动 UI 上的冷却计时器（例如技能图标灰化）。
        
    - **注意**：`GameplayAbility` 的实际冷却时间由服务端决定。
        
- **潜在问题**：
    
    - 受玩家延迟影响，客户端预测的冷却可能已经结束，但服务端仍认为 `GameplayAbility` 在冷却中。
        
    - 结果：客户端显示技能可用，但玩家无法立即再次激活，直到服务端冷却结束。
        
- **样例项目解决方案**：
    
    - 在客户端预测冷却开始时，将陨石技能图标灰化。
        
    - 当服务端校正的 `Cooldown GE` 到来时，启动冷却计时器，以确保 UI 与服务端冷却保持一致。
        
- **实际游戏影响**：
    
    - 高延迟玩家相比低延迟玩家，在冷却时间短的技能上触发率更低，处于劣势。
        
    - Fortnite 避免该问题的方法：通过自定义 bookkeeping，使武器使用无需依赖冷却 `GameplayEffect`。
        
- **未来展望**：
    
    - Epic 希望在未来的 GAS迭代版本中，实现真正的冷却预测：
        
        - 玩家可激活一个在客户端冷却完成但服务端仍处于冷却中的 `GameplayAbility`。 

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ge-duration"></a>
#### 4.5.16 修改已激活GameplayEffect的持续时间

为了修改`Cooldown GE`或其他任何`持续(Duration)`GameplayEffect的剩余时间, 我们需要修改`GameplayEffectSpec`的持续时间, 更新它的`StartServerWorldTime`, `CachedStartServerWorldTime`, `StartWorldTime`, 并且使用`CheckDuration()`重新检查持续时间. 在服务端上完成这些操作并将`FActiveGameplayEffect`标记为dirty, 其会将这些修改同步到客户端. **Note:** 该操作包含一个`const_cast`, 这可能不是`Epic`希望的修改持续时间的方法, 但是迄今为止它看起来运行得很好.  

```c++
bool UPAAbilitySystemComponent::SetGameplayEffectDurationHandle(FActiveGameplayEffectHandle Handle, float NewDuration)
{
	if (!Handle.IsValid())
	{
		return false;
	}

	const FActiveGameplayEffect* ActiveGameplayEffect = GetActiveGameplayEffect(Handle);
	if (!ActiveGameplayEffect)
	{
		return false;
	}

	FActiveGameplayEffect* AGE = const_cast<FActiveGameplayEffect*>(ActiveGameplayEffect);
	if (NewDuration > 0)
	{
		AGE->Spec.Duration = NewDuration;
	}
	else
	{
		AGE->Spec.Duration = 0.01f;
	}

	AGE->StartServerWorldTime = ActiveGameplayEffects.GetServerWorldTime();
	AGE->CachedStartServerWorldTime = AGE->StartServerWorldTime;
	AGE->StartWorldTime = ActiveGameplayEffects.GetWorldTime();
	ActiveGameplayEffects.MarkItemDirty(*AGE);
	ActiveGameplayEffects.CheckDuration(Handle);

	AGE->EventSet.OnTimeChanged.Broadcast(AGE->Handle, AGE->StartWorldTime, AGE->GetDuration());
	OnGameplayEffectDurationChange(*AGE);

	return true;
}
```

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ge-dynamic"></a>
#### 4.5.17 运行时创建动态`GameplayEffect`

在运行时创建动态`GameplayEffect`是一个高阶技术, 你不必经常使用它.  

只有`即刻(Instant)GameplayEffect`可以在运行时由C++创建, `持续(Duration)`和`无限(Infinite)`GameplayEffect不能在运行时动态创建, 因为它们在同步时会寻找并不存在的`GameplayEffect`类定义. 为了实现该功能, 你应该创建一个原型`GameplayEffect`类, 就像平时在编辑器中做的那样, 之后根据运行时所需来定制化`GameplayEffectSpec`.  

运行时创建的`即刻(Instant)GameplayEffect`也可以在客户端[预测](#concepts-p)的`GameplayAbility`中调用. 然而, 目前还不明确动态创建是否有副作用.  

样例项目会在角色`AttributeSet`中的值受到致命一击时创建该`GameplayEffect`来将金币和经验点数返还给击杀者.  

```c++
// Create a dynamic instant Gameplay Effect to give the bounties
UGameplayEffect* GEBounty = NewObject<UGameplayEffect>(GetTransientPackage(), FName(TEXT("Bounty")));
GEBounty->DurationPolicy = EGameplayEffectDurationType::Instant;

int32 Idx = GEBounty->Modifiers.Num();
GEBounty->Modifiers.SetNum(Idx + 2);

FGameplayModifierInfo& InfoXP = GEBounty->Modifiers[Idx];
InfoXP.ModifierMagnitude = FScalableFloat(GetXPBounty());
InfoXP.ModifierOp = EGameplayModOp::Additive;
InfoXP.Attribute = UGDAttributeSetBase::GetXPAttribute();

FGameplayModifierInfo& InfoGold = GEBounty->Modifiers[Idx + 1];
InfoGold.ModifierMagnitude = FScalableFloat(GetGoldBounty());
InfoGold.ModifierOp = EGameplayModOp::Additive;
InfoGold.Attribute = UGDAttributeSetBase::GetGoldAttribute();

Source->ApplyGameplayEffectToSelf(GEBounty, 1.0f, Source->MakeEffectContext());
```

第二个样例展示了在一个客户端预测的`GameplayAbility`中创建运行时`GameplayEffect`, 使用风险自负(查看代码中的注释)!

```c++
UGameplayAbilityRuntimeGE::UGameplayAbilityRuntimeGE()
{
	NetExecutionPolicy = EGameplayAbilityNetExecutionPolicy::LocalPredicted;
}

void UGameplayAbilityRuntimeGE::ActivateAbility(const FGameplayAbilitySpecHandle Handle, const FGameplayAbilityActorInfo* ActorInfo, const FGameplayAbilityActivationInfo ActivationInfo, const FGameplayEventData* TriggerEventData)
{
	if (HasAuthorityOrPredictionKey(ActorInfo, &ActivationInfo))
	{
		if (!CommitAbility(Handle, ActorInfo, ActivationInfo))
		{
			EndAbility(Handle, ActorInfo, ActivationInfo, true, true);
		}

		// Create the GE at runtime.
		UGameplayEffect* GameplayEffect = NewObject<UGameplayEffect>(GetTransientPackage(), TEXT("RuntimeInstantGE"));
		GameplayEffect->DurationPolicy = EGameplayEffectDurationType::Instant; // Only instant works with runtime GE.

		// Add a simple scalable float modifier, which overrides MyAttribute with 42.
		// In real world applications, consume information passed via TriggerEventData.
		const int32 Idx = GameplayEffect->Modifiers.Num();
		GameplayEffect->Modifiers.SetNum(Idx + 1);
		FGameplayModifierInfo& ModifierInfo = GameplayEffect->Modifiers[Idx];
		ModifierInfo.Attribute.SetUProperty(UMyAttributeSet::GetMyModifiedAttribute());
		ModifierInfo.ModifierMagnitude = FScalableFloat(42.f);
		ModifierInfo.ModifierOp = EGameplayModOp::Override;

		// Apply the GE.

		// Create the GESpec here to avoid the behavior of ASC to create GESpecs from the GE class default object.
		// Since we have a dynamic GE here, this would create a GESpec with the base GameplayEffect class, so we
		// would lose our modifiers. Attention: It is unknown, if this "hack" done here can have drawbacks!
		// The spec prevents the GE object being collected by the GarbageCollector, since the GE is a UPROPERTY on the spec.
		FGameplayEffectSpec* GESpec = new FGameplayEffectSpec(GameplayEffect, {}, 0.f); // "new", since lifetime is managed by a shared ptr within the handle
		ApplyGameplayEffectSpecToOwner(Handle, ActorInfo, ActivationInfo, FGameplayEffectSpecHandle(GESpec));
	}
	EndAbility(Handle, ActorInfo, ActivationInfo, false, false);
}
```

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ge-containers"></a>
#### 4.5.18 GameplayEffect Containers

Epic的[Action RPG](https://www.unrealengine.com/marketplace/en-US/slug/action-rpg)样例项目实现了一个名为`FGameplayEffectContainer`的结构体, 它不属于原生GAS, 但是对于包含`GameplayEffect`和[TargetData](#concepts-targeting-data)极其好用, 它会使一些过程自动化, 比如从`GameplayEffect`中创建`GameplayEffectSpec`并在其`GameplayEffectContext`中设置默认值. 在`GameplayAbility`中创建`GameplayEffectContainer`并将其传递给已生成的投掷物是非常简单和显而易见的, 然而我没有选择在样例项目中实现`GameplayEffectContainer`, 因为我想向你展示的是没有它的原生GAS, 但是我高度建议你研究一下它并将其纳入到你的项目中.  

为了访问`GameplayEffectContainer`中的`GESpec`以求做一些诸如添加`SetByCaller`的操作, 请使用`FGameplayEffectContainer`结构体中的`GESpec`数组索引访问`GESpec`引用, 这要求你需要提前知道想要访问的`GESpec`的索引.  

![[attachments/c0eed7425982aa2e758dd11d77018ee9_MD5.png]]  

`GameplayEffectContainer`还包含一个可选的用于[定位(Target)](#concepts-targeting-containers)的高效方法.

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ga"></a>
### 4.6 Gameplay Abilities

<a name="concepts-ga-definition"></a>
#### 4.6.1 GameplayAbility定义

[GameplayAbility(GA)](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/Abilities/UGameplayAbility/index.html)是Actor在游戏中可以触发的一切行为和技能. 多个`GameplayAbility`可以在同一时刻激活, 例如奔跑和射击. 其可由蓝图或C++完成.  

`GameplayAbility`示例:  

* 跳跃
* 奔跑
* 射击
* 每X秒被动地阻挡一次攻击
* 使用药剂
* 开门
* 收集资源
* 建造

不应该使用`GameplayAbility`的场景:  

* 基础移动输入
* 一些与UI的交互 - 不要使用`GameplayAbility`从商店中购买物品

这些不是规定, 只是我的建议而已, 你的设计和实现可能是多样的.  

`GameplayAbility`自带有根据等级修改Attribute变化量或者`GameplayAbility`作用的默认功能.  

`GameplayAbility`运行在所属(Owning)客户端还是服务端取决于[网络执行策略(Net Execution Policy)](#concepts-ga-net)而不是Simulated Proxy. `网络执行策略(Net Execution Policy)`决定某个`GameplayAbility`是否是客户端可[预测](#concepts-p)的, 其对于[可选的Cost和`Cooldown GameplayEffect`](#concepts-ga-commit)包含有默认行为. `GameplayAbility`使用[AbilityTask](#concepts-at)用于随时间推移而发生的行为, 例如等待某个事件, 等待某个Attribute改变, 等待玩家选择一个目标或者使用`Root Motion Source`移动某个`Character`. **Simulated Client不会运行`GameplayAbility`,** 而是当服务端执行`Ability`时, 任何需要在Simulated Proxy上展现的视觉效果(像动画蒙太奇)将会被同步(Replicate)或者通过`AbilityTask`进行RPC或者对于像声音和粒子这样的装饰效果使用[GameplayCue](#concepts-gc).  

所有的`GameplayAbility`都会有它们各自由你的游戏逻辑重写的`ActivateAbility()`函数, 附加的逻辑可以添加到`EndAbility()`, 其会在`GameplayAbility`完成或取消时执行.  

一个简单的`GameplayAbility`流程图: ![[attachments/10a9b92a92f597843a5ecaeb47b13c25_MD5.png]]  

一个更复杂`GameplayAbility`流程图: ![[attachments/7c6a74d820c4d0c583493314e655a78d_MD5.png]]  

复杂的Ability可以使用多个相互交互(激活, 取消等等)的`GameplayAbility`实现.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ga-definition-reppolicy"></a>
##### 4.6.1.1 Replication Policy

不要使用该选项. 这个名字会误导你并且你并不需要它. [GameplayAbilitySpec](#concepts-ga-spec)默认会从服务端向所属(Owning)客户端同步, 上文提到过, **`GameplayAbility`不会运行在Simulated Proxy上,** 其使用`AbilityTask`和`GameplayCue`来同步或者RPC视觉变化到Simulated Proxy. Epic的Dave Ratti已经表明要在未来[移除该选项](https://epicgames.ent.box.com/s/m1egifkxv3he3u3xezb9hzbgroxyhx89)的意愿.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ga-definition-remotecancel"></a>
##### 4.6.1.2 Server Respects Remote Ability Cancellation

这个选项往往会引起麻烦. 它的意思是如果客户端的`GameplayAbility`由于玩家取消或者自然完成时, 就会强制它的服务端版本结束而不管其是否完成. 最重要的是之后的问题, 特别是对于高延迟玩家所使用的客户端预测的`GameplayAbility`. 一般情况下禁用该选项.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ga-definition-repinputdirectly"></a>
##### 4.6.1.3 Replicate Input Directly

设置该选项就会一直向服务端同步输入的按下(Press)和抬起(Release)事件. Epic不建议使用该选项而是依靠创建在已有输入相关的[AbilityTask](#concepts-at)中的`Generic Replicated Event`(如果你的[输入绑定在ASC](#concepts-ga-input)).  

Epic的注释:  

```c++
/** Direct Input state replication. These will be called if bReplicateInputDirectly is true on the ability and is generally not a good thing to use. (Instead, prefer to use Generic Replicated Events). */
UAbilitySystemComponent::ServerSetInputPressed()
```

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ga-input"></a>
#### 4.6.2 绑定输入到ASC

`ASC`允许你直接绑定输入事件并当你授予`GameplayAbility`时分配这些输入到`GameplayAbility`, 如果`GameplayTag`合乎要求, 当按下按键时, 分配到`GameplayAbility`的输入事件会自动激活各自的`GameplayAbility`. 分配的输入事件要求使用响应输入的内建`AbilityTask`.  

除了分配的输入事件可以激活`GameplayAbility`, `ASC`也接受一般的`Confirm`和`Cancel`输入, 这些特殊输入被`AbilityTask`用来确定像[Target Actor](#concepts-targeting-actors)的对象或取消它们.  

为了绑定输入到`ASC`, 你必须首先创建一个枚举来将输入事件名称转换为byte, 枚举名必须准确匹配项目设置中用于输入事件的名称, `DisplayName`就无所谓了.  

样例项目中:  

```c++
UENUM(BlueprintType)
enum class EGDAbilityInputID : uint8
{
	// 0 None
	None			UMETA(DisplayName = "None"),
	// 1 Confirm
	Confirm			UMETA(DisplayName = "Confirm"),
	// 2 Cancel
	Cancel			UMETA(DisplayName = "Cancel"),
	// 3 LMB
	Ability1		UMETA(DisplayName = "Ability1"),
	// 4 RMB
	Ability2		UMETA(DisplayName = "Ability2"),
	// 5 Q
	Ability3		UMETA(DisplayName = "Ability3"),
	// 6 E
	Ability4		UMETA(DisplayName = "Ability4"),
	// 7 R
	Ability5		UMETA(DisplayName = "Ability5"),
	// 8 Sprint
	Sprint			UMETA(DisplayName = "Sprint"),
	// 9 Jump
	Jump			UMETA(DisplayName = "Jump")
};
```

如果你的`ASC`位于`Character`, 那么就在`SetupPlayerInputComponent()`中包含用于绑定到`ASC`的函数.  

```c++
// Bind to AbilitySystemComponent
AbilitySystemComponent->BindAbilityActivationToInputComponent(PlayerInputComponent, FGameplayAbilityInputBinds(FString("ConfirmTarget"), FString("CancelTarget"), FString("EGDAbilityInputID"), static_cast<int32>(EGDAbilityInputID::Confirm), static_cast<int32>(EGDAbilityInputID::Cancel)));
```

如果你的`ASC`位于`PlayerState`, `SetupPlayerInputComponent()`中有一个潜在的竞争情况就是`PlayerState`还没有同步到客户端, 因此, 我建议尝试在`SetupPlayerInputComponent()`和`OnRep_PlayerState()`中绑定输入, 只有`OnRep_PlayerState()`自身是不充分的, 因为可能有种情况是当`PlayerState`在`PlayerController`告知客户端调用用于创建`InputComponent`的`ClientRestart()`前同步时, Actor的`InputComponent`可能为NULL. 样例项目演示了尝试使用一个布尔值控制流程从而在两个位置绑定, 这样实际上只绑定了一次.  

**Note:** 样例项目枚举中的`Confirm`和`Cancel`没有匹配项目设置中的输入事件名称(`ConfirmTarget`和`CancelTarget`), 但是我们在`BindAbilityActivationToInputComponent()`中提供了它们之间的映射, 这是特殊的, 因为我们提供了映射并且它们无需匹配, 但是它们是可以匹配的. 枚举中的其他输入都必须匹配项目设置中的输入事件名称.  

对于只能用一次输入激活的`GameplayAbility`(它们总是像MOBA游戏一样存在于相同的"槽"中), 我倾向在`UGameplayAbility`子类中添加一个变量, 这样我就可以定义他们的输入, 之后在授予Ability的时候可以从`ClassDefaultObject`中读取这个值.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ga-input-noactivate"></a>
##### 4.6.2.1 绑定输入时不激活Ability

如果你不想你的`GameplayAbility`在按键按下时自动激活, 但是仍想将它们绑定到输入以与`AbilityTask`一起使用, 你可以在`UGameplayAbility`子类中添加一个新的布尔变量, `bActivateOnInput`, 其默认值为`true`并重写`UAbilitySystemComponent::AbilityLocalInputPressed()`.  

```c++
void UGSAbilitySystemComponent::AbilityLocalInputPressed(int32 InputID)
{
	// Consume the input if this InputID is overloaded with GenericConfirm/Cancel and the GenericConfim/Cancel callback is bound
	if (IsGenericConfirmInputBound(InputID))
	{
		LocalInputConfirm();
		return;
	}

	if (IsGenericCancelInputBound(InputID))
	{
		LocalInputCancel();
		return;
	}

	// ---------------------------------------------------------

	ABILITYLIST_SCOPE_LOCK();
	for (FGameplayAbilitySpec& Spec : ActivatableAbilities.Items)
	{
		if (Spec.InputID == InputID)
		{
			if (Spec.Ability)
			{
				Spec.InputPressed = true;
				if (Spec.IsActive())
				{
					if (Spec.Ability->bReplicateInputDirectly && IsOwnerActorAuthoritative() == false)
					{
						ServerSetInputPressed(Spec.Handle);
					}

					AbilitySpecInputPressed(Spec);

					// Invoke the InputPressed event. This is not replicated here. If someone is listening, they may replicate the InputPressed event to the server.
					InvokeReplicatedEvent(EAbilityGenericReplicatedEvent::InputPressed, Spec.Handle, Spec.ActivationInfo.GetActivationPredictionKey());
				}
				else
				{
					UGSGameplayAbility* GA = Cast<UGSGameplayAbility>(Spec.Ability);
					if (GA && GA->bActivateOnInput)
					{
						// Ability is not active, so try to activate it
						TryActivateAbility(Spec.Handle);
					}
				}
			}
		}
	}
}
```

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ga-granting"></a>
#### 4.6.3 授予Ability

向`ASC`授予`GameplayAbility`会将其添加到`ASC`的`ActivatableAbilities`列表, 从而允许其在满足[`GameplayTag`需求](#concepts-ga-tags)时激活该`GameplayAbility`.  

我们在服务端授予`GameplayAbility`, 之后其会自动同步[GameplayAbilitySpec](#concepts-ga-spec)到所属(Owning)客户端, 其他客户端/Simulated proxy不会接受到`GameplayAbilitySpec`.  

样例项目在游戏开始时将`TArray<TSubclassOf<UGDGameplayAbility>>`保存在它读取和授予的`Character`类中.  

```c++
void AGDCharacterBase::AddCharacterAbilities()
{
	// Grant abilities, but only on the server	
	if (Role != ROLE_Authority || !AbilitySystemComponent.IsValid() || AbilitySystemComponent->CharacterAbilitiesGiven)
	{
		return;
	}

	for (TSubclassOf<UGDGameplayAbility>& StartupAbility : CharacterAbilities)
	{
		AbilitySystemComponent->GiveAbility(
			FGameplayAbilitySpec(StartupAbility, GetAbilityLevel(StartupAbility.GetDefaultObject()->AbilityID), static_cast<int32>(StartupAbility.GetDefaultObject()->AbilityInputID), this));
	}

	AbilitySystemComponent->CharacterAbilitiesGiven = true;
}
```

当授予这些`GameplayAbility`时, 我们就在使用`UGameplayAbility`类, Ability等级, 其绑定的输入和`SourceObject`或将该`GameplayAbility`设置到该`ASC`的源(Source)创建`GameplayAbilitySpec`.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ga-activating"></a>
#### 4.6.4 激活Ability

如果某个`GameplayAbility`被分配给了一个输入事件, 那么当输入按键按下并且它的`GameplayTag`需求满足时, 它将会自动激活, 这可能并非总是激活`GameplayAbility`的期望方式. `ASC`提供了另外四种激活`GameplayAbility`的方法: 通过`GameplayTag`, `GameplayAbility`类, `GameplayAbilitySpecHandle`和Event, 通过Event激活`GameplayAbility`允许你[传递一个该事件的数据负载(Payload)](#concepts-ga-data).  

```c++
UFUNCTION(BlueprintCallable, Category = "Abilities")
bool TryActivateAbilitiesByTag(const FGameplayTagContainer& GameplayTagContainer, bool bAllowRemoteActivation = true);

UFUNCTION(BlueprintCallable, Category = "Abilities")
bool TryActivateAbilityByClass(TSubclassOf<UGameplayAbility> InAbilityToActivate, bool bAllowRemoteActivation = true);

bool TryActivateAbility(FGameplayAbilitySpecHandle AbilityToActivate, bool bAllowRemoteActivation = true);

bool TriggerAbilityFromGameplayEvent(FGameplayAbilitySpecHandle AbilityToTrigger, FGameplayAbilityActorInfo* ActorInfo, FGameplayTag Tag, const FGameplayEventData* Payload, UAbilitySystemComponent& Component);

FGameplayAbilitySpecHandle GiveAbilityAndActivateOnce(const FGameplayAbilitySpec& AbilitySpec);
```

想要通过Event激活`GameplayAbility`, `GameplayAbility`必须设置它的`Trigger`, 分配一个`GameplayTag`并为`GameplayEvent`选择一个选项. 想要发送Event, 就得使用`UAbilitySystemBlueprintLibrary::SendGameplayEventToActor(AActor* Actor, FGameplayTag EventTag, FGameplayEventData Payload)`函数. 通过Event激活`GameplayAbility`允许你传递一个数据负载(Payload).  

`GameplayAbility Trigger`也允许你在某个`GameplayTag`添加或移除时激活该`GameplayAbility`.  

**Note:** 当从蓝图中的Event激活`GameplayAbility`时, 你必须使用`ActivateAbilityFromEvent`节点, 并且标准的`ActivateAbility`节点不能出现在图表中, 如果`ActivateAbility`节点存在, 它就会一直被调用而不调用`ActivateAbilityFromEvent`节点.  

**Note:** 不要忘记应该在`GameplayAbility`终止时调用`EndAbility()`, 除非你的`GameplayAbility`是像被动技能那样一直运行的`GameplayAbility`.  

对于**客户端预测**`GameplayAbility`的激活序列:  

1. **所属(Owning)客户端**调用`TryActivateAbility()`
2. 调用`InternalTryActivateAbility()`
3. 调用`CanActivateAbility()`并返回是否满足`GameplayTag`需求, `ASC`是否满足技能花费, `GameplayAbility`是否不在冷却期和当前是否没有其他实例被激活
4. 调用`CallServerTryActivateAbility()`并传入其生成的`Prediction Key`
5. 调用`CallActivateAbility()`
6. 调用`PreActivate()`, Epic称之为"boilerplate init stuff"
7. 调用`ActivateAbility()`最终激活Ability

**服务端**接收到`CallServerTryActivateAbility()`  

1. 调用`ServerTryActivateAbility()`
2. 调用`InternalServerTryActivateAbility()`
3. 调用`InternalTryActivateAbility()`
4. 调用`CanActivateAbility()`并返回是否满足`GameplayTag`需求, `ASC`是否满足技能花费, `GameplayAbility`是否不在冷却期和当前是否没有其他实例被激活
5. 如果成功则调用`ClientActivateAbilitySucceed()`告知客户端更新它的`ActivationInfo`(即该激活已由服务端确认)并广播`OnConfirmDelegate`代理. 这和输入确认(Input Confirmation)不一样.
6. 调用`CallActivateAbility()`
7. 调用`PreActivate()`, Epic称之为"boilerplate init stuff"
8. 调用`ActivateAbility()`最终激活Ability

如果服务端在任意时刻激活失败, 就会调用`ClientActivateAbilityFailed()`, 立即终止客户端的`GameplayAbility`并撤销所有预测的修改.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ga-activating-passive"></a>
##### 4.6.4.1 被动Ability

为了实现自动激活和持续运行的被动`GameplayAbility`, 需要重写`UGameplayAbility::OnAvatarSet()`, 该函数在授予`GameplayAbility`并设置`AvatarActor`且调用`TryActivateAbility()`时自动调用.  

我建议添加一个`布尔值`到你的自定义`UGameplayAbility`类来表明其在授予时是否应该被激活. 样例项目中的被动护甲叠层Ability是这样做的.  

被动`GameplayAbility`一般有一个`仅服务器(Server Only)`的[网络执行策略(Net Execution Policy)](#concepts-ga-net).  

```c++
void UGDGameplayAbility::OnAvatarSet(const FGameplayAbilityActorInfo * ActorInfo, const FGameplayAbilitySpec & Spec)
{
	Super::OnAvatarSet(ActorInfo, Spec);

	if (ActivateAbilityOnGranted)
	{
		bool ActivatedAbility = ActorInfo->AbilitySystemComponent->TryActivateAbility(Spec.Handle, false);
	}
}
```

Epic描述该函数为初始化被动Ability的正确位置和应该做一些类似`BeginPlay`的事情.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ga-cancelabilities"></a>
#### 4.6.5 取消Ability

为了从内部取消`GameplayAbility`, 可以调用`CancelAbility()`, 其会调用`EndAbility()`并设置它的`WasCancelled`参数为`true`.  

为了从外部取消`GameplayAbility`, `ASC`提供了一些函数:  

```c++
/** Cancels the specified ability CDO. */
void CancelAbility(UGameplayAbility* Ability);	

/** Cancels the ability indicated by passed in spec handle. If handle is not found among reactivated abilities nothing happens. */
void CancelAbilityHandle(const FGameplayAbilitySpecHandle& AbilityHandle);

/** Cancel all abilities with the specified tags. Will not cancel the Ignore instance */
void CancelAbilities(const FGameplayTagContainer* WithTags=nullptr, const FGameplayTagContainer* WithoutTags=nullptr, UGameplayAbility* Ignore=nullptr);

/** Cancels all abilities regardless of tags. Will not cancel the ignore instance */
void CancelAllAbilities(UGameplayAbility* Ignore=nullptr);

/** Cancels all abilities and kills any remaining instanced abilities */
virtual void DestroyActiveState();
```

**Note:** 我发现如果存在一个`非实例(Non-Instanced)GameplayAbility`时, `CancelAllAbilities`似乎不能正常运行, 它似乎会命中这个`非实例(Non-Instanced)GameplayAbility`并放弃继续处理. `CancelAbility`可以更好地处理`非实例(Non-Instanced)GameplayAbility`, 样例项目就是这样使用的(跳跃是一个非实例(Non-Instanced)GameplayAbility), 因人而异.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ga-definition-activeability"></a>
#### 4.6.6 获取激活的Ability

初学者经常会问"我怎样才能获取激活的Ability?", 也许是用来设置变量或取消它. 多个`GameplayAbility`可以在同一时刻激活, 因此没有"一个激活的Ability", 相反, 你必须搜索`ASC`的`ActivatableAbility`列表(`ASC`拥有的已授予`GameplayAbility`)并找到一个与你正在寻找的[资源或授予的GameplayTag](#concepts-ga-tags)相匹配的Ability.  

`UAbilitySystemComponent::GetActivatableAbilities()`会返回一个用于遍历的`TArray<FGameplayAbilitySpec>`.  

`ASC`还有另一个有用的函数, 它将一个`GameplayTagContainer`作为参数来协助搜索, 而无需手动遍历`GameplayAbilitySpec`列表. `bOnlyAbilitiesThatSatisfyTagRequirements`参数只会返回那些`GameplayTag`满足需求且可以立刻激活的`GameplayAbilitySpecs`, 例如, 你可能有两个基本的攻击`GameplayAbility`, 一个使用武器, 另一个使用拳头, 正确的激活取决于武器是否装备并设置了`GameplayTag`需求. 详见Epic关于函数的注释.  

```c++
UAbilitySystemComponent::GetActivatableGameplayAbilitySpecsByAllMatchingTags(const FGameplayTagContainer& GameplayTagContainer, TArray < struct FGameplayAbilitySpec* >& MatchingGameplayAbilities, bool bOnlyAbilitiesThatSatisfyTagRequirements = true)
```

一旦你获取到了寻找的`FGameplayAbilitySpec`, 那么就可以调用它的`IsActive()`.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ga-instancing"></a>
#### 4.6.7 实例化策略

`GameplayAbility`的实例化策略决定了当`GameplayAbility`激活时是否和如何实例化.  

|实例化策略|描述|何时使用的例子|
|:-:|:-:|:-:|
|按Actor实例化(Instanced Per Actor)|每个`ASC`只能有一个在激活之间复用的`GameplayAbility`实例.|这可能是你使用最频繁的实例化策略. 你可以对任一`Ability`使用并在激活之间提供持久化. 设计者可以在激活之间手动重设任意变量.|
|按操作实例化(Instanced Per Execution)|每有一个`GameplayAbility`激活, 就有一个新的`GameplayAbility`实例创建.|这些`GameplayAbility`的好处是每次激活时变量都会重置, 其性能要比`Instanced Per Actor`差, 因为每次激活时都会生成新的`GameplayAbility`. 样例项目没有使用该方式.|
|非实例化(Non-Instanced)|`GameplayAbility`操作其`ClassDefaultObject`, 没有实例创建.|它是三种方式中性能最好的, 但是使用它是最受限制的. `非实例化(Non-Instanced)GameplayAbility`不能存储状态, 这意味着没有动态变量和不能绑定到`AbilityTask`委托. 使用它的最佳场景就是需要频繁使用的简单Ability, 像MOBA或RTS游戏中小兵的基础攻击. 样例项目中的跳跃`GameplayAbility`就是`非实例化(Non-Instanced)`的.|

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ga-net"></a>
#### 4.6.8 网络执行策略(Net Execution Policy)

`GameplayAbility`的`网络执行策略(Net Execution Policy)`决定了谁该以什么顺序运行该`GameplayAbility`.  

|网络执行策略(Net Execution Policy)|描述|
|:-:|:-:|
|Local Only|`GameplayAbility`只运行在所属(Owning)客户端. 这对那些只需做本地视觉变化的Ability很有用. 单人游戏应该使用`Server Only`.|
|Local Predicted|`Local Predicted GameplayAbility`首先在所属(Owning)客户端激活, 之后在服务端激活. 服务端版本会纠正客户端预测的所有不正确的地方. 参见Prediction.|
|Server Only|`GameplayAbility`只运行在服务端. 被动`GameplayAbility`一般是`Server Only`. 单人游戏应该使用该项.|
|Server Initiated|`Server Initiated GameplayAbility`首先在服务端激活, 之后在所属(Owning)客户端激活. 我个人使用的不多(如果有的话).|

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ga-tags"></a>
#### 4.6.9 Ability标签

`GameplayAbility`自带有内建逻辑的`GameplayTagContainer`. 这些`GameplayTag`都不进行同步.  

|GameplayTagContainer|描述|
|:-:|:-:|
|Ability Tags|`GameplayAbility`拥有的`GameplayTag`, 这只是用来描述`GameplayAbility`的`GameplayTag`.|
|Cancel Abilities with Tag|当该`GameplayAbility`激活时, 其他`Ability Tags`中拥有这些`GameplayTag`的`GameplayAbility`将会被取消.|
|Block Abilities with Tag|当该`GameplayAbility`激活时, 其他`Ability Tags`中拥有这些`GameplayTag`的`GameplayAbility`将会阻塞激活.|
|Activation Owned Tags|当该`GameplayAbility`激活时, 这些`GameplayTag`会交给该`GameplayAbility`的拥有者.|
|Activation Required Tags|该`GameplayAbility`只有在其拥有者拥有所有这些`GameplayTag`时才会激活.|
|Activation Blocked Tags|该`GameplayAbility`在其拥有者拥有任意这些标签时不能被激活.|
|Source Required Tags|该`GameplayAbility`只有在`Source`拥有所有这些`GameplayTag`时才会激活. `Source GameplayTag`只有在该`GameplayAbility`由Event触发时设置.|
|Source Blocked Tags|该`GameplayAbility`在`Source`拥有任意这些标签时不能被激活. `Source GameplayTag`只有在该`GameplayAbility`由Event触发时设置.|
|Target Required Tags|该`GameplayAbility`只有在`Target`拥有所有这些`GameplayTag`时才会激活. `Target GameplayTag`只有在该`GameplayAbility`由Event触发时设置.|
|Target Blocked Tags|该`GameplayAbility`在`Target`拥有任意这些标签时不能被激活. `Target GameplayTag`只有在该`GameplayAbility`由Event触发时设置.|

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ga-spec"></a>
#### 4.6.10 Gameplay Ability Spec

`GameplayAbilitySpec`会在`GameplayAbility`授予后存在于`ASC`中并定义可激活`GameplayAbility` - `GameplayAbility`类, 等级, 输入绑定和必须与`GameplayAbility`类分开保存的运行时状态.  

当`GameplayAbility`在服务端授予时, 服务端会同步`GameplayAbilitySpec`到所属(Owning)客户端, 因此可以激活它.  

激活`GameplayAbilitySpec`会根据它的`实例化策略(Instancing Policy)`创建一个`GameplayAbility`实例(`Non-Instanced GameplayAbility`除外).  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ga-data"></a>

#### 4.6.11 传递数据到Ability

`GameplayAbility`的一般范式是`Activate->Generate Data->Apply->End`. 有时你需要调整现有数据, GAS提供了一些选项来获取外部数据到你的`GameplayAbility`.  

|方法|描述|
|:-:|:-:|
|通过Event激活`GameplayAbility`|使用含有数据负载(Payload)的Event激活`GameplayAbility`. 对于客户端预测的`GameplayAbility`, Event的负载(Payload)是由客户端同步到服务端的. 对于那些不适合任意现有变量的数据可以使用两个`Optional Object`或[TargetData](#concepts-targeting-data)变量. 该方法的缺点是不能使用输入绑定激活Ability. 为了通过Event激活`GameplayAbility`, 该`GameplayAbility`必须设置其Trigger, 分配一个`GameplayTag`并选择一个`GameplayEvent`选项. 想要发送事件, 就得使用`UAbilitySystemBlueprintLibrary::SendGameplayEventToActor(AActor* Actor, FGameplayTag EventTag, FGameplayEventData Payload)`函数.|
|使用`WaitGameplayEvent AbilityTask`|在`GameplayAbility`激活后, 使用`WaitGameplayEvent AbilityTask`来告知其监听带有负载(Payload)的事件. Event负载(Payload)及其发送过程与通过Event激活`GameplayAbility`是一样的. 该方法的缺点是Event不能通过`AbilityTask`同步且只能用于`Local Only`和`Server Only`的`GameplayAbility`. 你可以编写自己的`AbilityTask`以支持同步Event负载(Payload).|
|使用`TargetData`|自定义`TargetData`结构体是一种在客户端和服务端之间传递任意数据的好方法.|
|保存数据在`OwnerActor`或者`AvatarActor`|使用保存于`OwnerActor`, `AvatarActor`或者其他任意你可以获取到引用的对象中的可同步变量. 这种方法最灵活且可以用于由输入绑定激活的`GameplayAbility`, 然而, 它不能保证在使用时数据同步, 你必须提前做好准备 - 这意味着如果你设置了一个可同步的变量, 之后立即激活该`GameplayAbility`, 那么因为存在潜在的丢包情况, 不能保证接收者上发生的顺序.|

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ga-commit"></a>
#### 4.6.12 Ability花费(Cost)和冷却(Cooldown)

`GameplayAbility`默认带有对可选Cost和Cooldown的功能. Cost是`ASC`为了激活使用`即刻(Instant)GameplayEffect`([Cost GE](#concepts-ge-cost))实现的GameplayAbility所必须具有的预先定义的大量Attribute. Cooldown是用于阻止`GameplayAbility`再次激活(直到冷却完成)的计时器, 其由`持续(Duration)GameplayEffect`([Cooldown GE](#concepts-ge-cooldown))实现.  

在`GameplayAbility`调用`UGameplayAbility::Activate()`之前, 其会调用`UGameplayAbility::CanActivateAbility()`, 该函数会检查所属`ASC`是否满足Cost(`UGameplayAbility::CheckCost()`)并确保该`GameplayAbility`不在冷却期(`UGameplayAbility::CheckCooldown()`).  

在`GameplayAbility`调用`Activate()`之后, 其可以选择性使用`UGameplayAbility::CommitAbility()`随时提交Cost和Cooldown, `UGameplayAbility::CommitAbility()`会调用`UGameplayAbility::CommitCost()`和`UGameplayAbility::CommitCooldown()`, 如果它们不需要同时提交, 设计师可以选择分别调用`CommitCost()`或`CommitCooldown()`. 提交Cost和Cooldown会多次调用`CheckCost()`和`CheckCooldown()`. 所属(Owning)`ASC`的Attribute在`GameplayAbility`激活后可能改变, 从而导致提交时无法满足Cost. 如果[prediction key](#concepts-p-key)在提交时有效的话, 提交Cost和Cooldown是可以[客户端预测](#concepts-p)的.  

详见[CostGE](#concepts-ge-cost)和[CooldownGE](#concepts-ge-cooldown).  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ga-leveling"></a>
#### 4.6.13 升级Ability

有两个常见的方法用于升级Ability:  

|升级方法|描述|
|:-:|:-:|
|升级时取消授予和重新授予|自`ASC`取消授予(移除)`GameplayAbility`, 并在下一等级于服务端上重新授予. 如果此时`GameplayAbility`是激活的, 那么就会终止它.|
|增加`GameplayAbilitySpec`的等级|在服务端上找到`GameplayAbilitySpec`, 增加它的等级, 并将其标记为Dirty以同步到所属(Owning)客户端. 该方法不会在`GameplayAbility`激活时将其终止.|

两种方法的主要区别在于你是否想要在升级时取消激活的`GameplayAbility`. 基于你的`GameplayAbility`, 你很可能需要同时使用两种方法. 我建议在`UGameplayAbility`子类中增加一个布尔值以明确使用哪种方法.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ga-sets"></a>
#### 4.6.14 Ability集合

`GameplayAbilitySet`是很方便的`UDataAsset`类, 其用于保存输入绑定和初始`GameplayAbility`列表, 该`GameplayAbility`列表用于拥有授予`GameplayAbility`逻辑的Character. 子类也可以包含额外的逻辑和属性. Paragon中的每个英雄都拥有一个`GameplayAbilitySet`以包含其授予的所有`GameplayAbility`.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ga-batching"></a>
#### 4.6.15 Ability批处理

一般`GameplayAbility`的生命周期最少涉及2到3个自客户端到服务端的RPC.  

1. CallServerTryActivateAbility()
2. ServerSetReplicatedTargetData() (可选)
3. ServerEndAbility()

如果`GameplayAbility`在一帧同一原子(Atomic)组中执行这些操作, 我们就可以优化该工作流, 将所有2个或3个RPC批处理(整合)为1个RPC. `GAS`称这种RPC优化为`Ability批处理`. 常见的使用`Ability批处理`时机的例子是枪支命中判断. 枪支命中判断时, 会在一帧同一原子(Atomic)组中做射线检测, 发送[TargetData](#concepts-targeting-data)到服务端并结束Ability. GASShooter样例项目对其枪支命中判断使用了该技术.  

半自动枪支是最好的案例, 其将`CallServerTryActivateAbility()`, `ServerSetReplicatedTargetData()`(子弹命中结果)和`ServerEndAbility()`批处理成一个而不是三个`RPC`.  

全自动/爆炸开火枪支会对第一发子弹将`CallServerTryActivateAbility()`和`ServerSetReplicatedTargetData()`批处理成一个而不是两个RPC, 接下来的每一发子弹都是它自己的`ServerSetReplicatedTargetData()`RPC, 最后, 当停止射击时, `ServerEndAbility()`会作为单独的RPC发送. 这是最坏的情况, 我们只保存了第一发子弹的一个RPC而不是两个. 这种情况也可以通过[GameplayEvent](#concepts-ga-data)激活Ability来实现, 该`GameplayEvent`会自客户端向服务端发送带有`EventPayload`的子弹`TargetData`. 后者的缺点是`TargetData`必须在Ability外部生成, 尽管批处理方法在Ability内部生成`TargetData`.  

Ability批处理在[ASC](#concepts-asc)中是默认禁止的. 为了启用Ability批处理, 需要重写`ShouldDoServerAbilityRPCBatch()`以返回true:  

```c++
virtual bool ShouldDoServerAbilityRPCBatch() const override { return true; }
```

现在Ability批处理已经启用, 在激活你想使用批处理的Ability之前, 必须提前创建一个`FScopedServerAbilityRPCBatcher`结构体, 这个特殊的结构体会在其域内尝试批处理所有跟随它的Ability, 一旦出了`FScopedServerAbilityRPCBatcher`域, 所有已激活的Ability将不再尝试批处理. `FScopedServerAbilityRPCBatcher`通过在每个可以批处理的函数中设置特殊代码来运行, 其会拦截RPC的发送并将消息打包进一个批处理结构体, 当出了`FScopedServerAbilityRPCBatcher`后, 它会在`UAbilitySystemComponent::EndServerAbilityRPCBatch()`中自动将该批处理结构体RPC到服务端, 服务端在`UAbilitySystemComponent::ServerAbilityRPCBatch_Internal(FServerAbilityRPCBatch& BatchInfo)`中接收该批处理结构体, `BatchInfo`参数会包含几个flag, 其包括该Ability是否应该结束, 激活时输入是否按下和是否包含`TargetData`. 这是个可以设置断点以确认批处理是否正确运行的好函数. 作为选择, 可以使用`AbilitySystem.ServerRPCBatching.Log 1`来启用特别的Ability批处理日志.  

这个方法只能在C++中完成, 并且只能通过`FGameplayAbilitySpecHandle`来激活Ability.  

```c++
bool UGSAbilitySystemComponent::BatchRPCTryActivateAbility(FGameplayAbilitySpecHandle InAbilityHandle, bool EndAbilityImmediately)
{
	bool AbilityActivated = false;
	if (InAbilityHandle.IsValid())
	{
		FScopedServerAbilityRPCBatcher GSAbilityRPCBatcher(this, InAbilityHandle);
		AbilityActivated = TryActivateAbility(InAbilityHandle, true);

		if (EndAbilityImmediately)
		{
			FGameplayAbilitySpec* AbilitySpec = FindAbilitySpecFromHandle(InAbilityHandle);
			if (AbilitySpec)
			{
				UGSGameplayAbility* GSAbility = Cast<UGSGameplayAbility>(AbilitySpec->GetPrimaryInstance());
				GSAbility->ExternalEndAbility();
			}
		}

		return AbilityActivated;
	}

	return AbilityActivated;
}
```

GASShooter对半自动和全自动枪支使用了相同的批处理`GameplayAbility`, 并没有直接调用`EndAbility()`(它通过一个只能由客户端调用的Ability在该Ability外处理, 该只能由客户端调用的Ability用于管理玩家输入和基于当前开火模式对批处理Ability的调用). 因为所有的RPC必须在`FScopedServerAbilityRPCBatcher`域中, 所以我提供了`EndAbilityImmediately`参数以使仅客户端的控制/管理可以明确该Ability是否可以批处理`EndAbility()`(半自动)或不批处理`EndAbility()`(全自动)以及之后`EndAbility()`是否可以在其自己的RPC中调用.  

GASShooter暴露了一个蓝图节点以允许上文提到的仅客户端调用的Ability所使用的批处理Ability来触发批处理Ability. (译者注: 此处相当拗口, 但原文翻译确实如此, 配合项目浏览也许会更容易明白些.)  

![[attachments/789e45f3d6c054bea171333c58322c40_MD5.png]]  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-ga-netsecuritypolicy"></a>
#### 4.6.16 网络安全策略(Net Security Policy)

`GameplayAbility`的网络安全策略决定了Ability应该在网络的何处执行. 它为尝试执行限制Ability的客户端提供了保护.  

|网络安全策略|描述|
|:-:|:-:|
|ClientOrServer|没有安全需求. 客户端或服务端可以自由地触发该Ability的执行和终止.|
|ServerOnlyExecution|客户端对该Ability请求的执行会被服务端忽略, 但客户端仍可以请求服务端取消或结束该Ability.|
|ServerOnlyTermination|客户端对该Ability请求的取消或结束会被服务端忽略, 但客户端仍可以请求执行该Ability.|
|ServerOnly|服务端控制该Ability的执行和终止, 客户端的任何请求都会被忽略.|

<a name="concepts-at"></a>
### 4.7 Ability Tasks

<a name="concepts-at-definition"></a>
#### 4.7.1 AbilityTask定义

`GameplayAbility`只能在一帧中执行, 这本身并不能提供太多灵活性, 为了实现随时间推移而触发或响应一段时间后触发的委托操作, 我们需要使用`AbilityTask`.  

GAS自带很多`AbilityTask`:  

* 使用`RootMotionSource`移动Character的Task
* 播放动画蒙太奇的Task
* 响应`Attribute`变化的Task
* 响应`GameplayEffect`变化的Task
* 响应玩家输入的Task
* 更多

`UAbilityTask`的构造函数中强制硬编码允许最多1000个同时运行的`AbilityTask`, 当设计那些同时拥有数百个Character的游戏(像RTS)的`GameplayAbility`时要注意这一点.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-at-definition"></a>
#### 4.7.2 自定义AbilityTask

通常你需要创建自己的自定义`AbilityTask`(C++中). 样例项目带有两个自定义`AbilityTask`:  

1. `PlayMontageAndWaitForEvent`是默认`PlayMontageAndWait`和`WaitGameplayEvent`AbilityTask的结合体, 其允许动画蒙太奇自`AnimNotify`发送GameplayEvent回到启动它的`GameplayAbility`, 可以使用该Task在动画蒙太奇的某个特定时刻来触发操作.
2. `WaitReceiveDamage`可以监听`OwnerActor`接收伤害. 当英雄接收到一个伤害实例时, 被动护甲层`GameplayAbility`就会移除一层护甲.  

`AbilityTask`的组成:  

* 创建新的`AbilityTask`实例的静态函数
* 当`AbilityTask`完成目的时分发的委托(Delegate)
* 进行主要工作的`Activate()`函数, 绑定到外部的委托等等
* 进行清理工作的`OnDestroy()`函数, 包括其绑定到外部的委托
* 所有绑定到外部委托的回调函数
* 成员变量和所有内部辅助函数

**Note:** `AbilityTask`只能声明一种类型的输出委托, 所有的输出委托都必须是该种类型, 不管它们是否使用参数. 对于未使用的委托参数会传递默认值.  

`AbilityTask`只能运行在那些运行所属`GameplayAbility`的客户端或服务端, 然而, 可以通过设置`bSimulatedTask = true`使`AbilityTask`运行在Simulated Client上, 在`AbilityTask`的构造函数中, 重写`virtual void InitSimulatedTask(UGameplayTasksComponent& InGameplayTasksComponent);`并将所有成员变量设置为同步的, 这只在极少的情况下有用, 比如在移动`AbilityTask`中, 不想同步每次移动变化, 但是又需要模拟整个移动`AbilityTask`, 所有的`RootMotionSource AbilityTask`都是这样做的, 查看`AbilityTask_MoveToLocation.h/.cpp`以作为参考范例.  

如果你在`AbilityTask`的构造函数中设置了`bTickingTask = true;`并重写了`virtual void TickTask(float DeltaTime);`, `AbilityTask`就可以使用`Tick`, 这在你需要根据帧率平滑线性插值的时候很有用. 查看`AbilityTask_MoveToLocation.h/.cpp`以作为参考范例.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-at-using"></a>
#### 4.7.3 使用AbilityTask

在C++中创建并激活`AbilityTask`(GDGA_FireGun.cpp):  

```c++
UGDAT_PlayMontageAndWaitForEvent* Task = UGDAT_PlayMontageAndWaitForEvent::PlayMontageAndWaitForEvent(this, NAME_None, MontageToPlay, FGameplayTagContainer(), 1.0f, NAME_None, false, 1.0f);
Task->OnBlendOut.AddDynamic(this, &UGDGA_FireGun::OnCompleted);
Task->OnCompleted.AddDynamic(this, &UGDGA_FireGun::OnCompleted);
Task->OnInterrupted.AddDynamic(this, &UGDGA_FireGun::OnCancelled);
Task->OnCancelled.AddDynamic(this, &UGDGA_FireGun::OnCancelled);
Task->EventReceived.AddDynamic(this, &UGDGA_FireGun::EventReceived);
Task->ReadyForActivation();
```

在蓝图中, 我们只需使用为`AbilityTask`创建的蓝图节点, 不必调用`ReadyForActivate()`, 其由`Engine/Source/Editor/GameplayTasksEditor/Private/K2Node_LatentGameplayTaskCall.cpp`自动调用. `K2Node_LatentGameplayTaskCall`也会自动调用`BeginSpawningActor()`和`FinishSpawningActor()`(如果它们存在于你的`AbilityTask`类中, 查看`AbilityTask_WaitTargetData`), 再强调一遍, `K2Node_LatentGameplayTaskCall`只会对蓝图做这些自动操作, 在C++中, 我们必须手动调用`ReadyForActivation()`, `BeginSpawningActor()`和`FinishSpawningActor()`.  

![[attachments/888ed3c330cc4a58423f1ba9c3ce0459_MD5.png]]  

如果想要手动取消`AbilityTask`, 只需在蓝图(Async Task Proxy)或C++中对`AbilityTask`对象调用`EndTask()`.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-at-rms"></a>
#### 4.7.4 Root Motion Source Ability Task

GAS自带的`AbilityTask`可以使用挂载在`CharacterMovementComponent`中的`Root Motion Source`随时间推移而移动`Character`, 像击退, 复杂跳跃, 吸引和猛冲.  

**Note:** `RootMotionSource AbilityTask`预测支持的引擎版本是4.19和4.25+, 该预测在引擎版本4.20-4.24中存在bug, 然而, `AbilityTask`仍然可以使用较小的网络修正在多人游戏中执行功能, 并且在单人游戏中完美运行. 可以将4.25中对预测的修复自定义到4.20~4.24引擎中.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-gc"></a>
### 4.8 Gameplay Cues

<a name="concepts-gc-definition"></a>
#### 4.8.1 GameplayCue定义

`GameplayCue(GC)`执行非游戏逻辑相关的功能, 像音效, 粒子效果, 镜头抖动等等. `GameplayCue`一般是可同步(除非在客户端明确`执行(Executed)`, `添加(Added)`和`移除(Removed)`)和可预测的.  

我们可以在`ASC`中通过发送一个**强制带有"GameplayCue"父名**的相应`GameplayTag`和`GameplayCueManager`的事件类型(Execute, Add或Remove)来触发`GameplayCue`. `GameplayCueNotify`对象和其他实现`IGameplayCueInterface`的`Actor`可以基于`GameplayCue`的`GameplayTag(GameplayCueTag)`来订阅(Subscribe)这些事件.  

**Note:** 再次强调, `GameplayCue`的`GameplayTag`需要以`GameplayCue`为开头, 举例来说, 一个有效的`GameplayCue`的`GameplayTag`可能是`GameplayCue.A.B.C`.  

有两个`GameplayCueNotify`类, `Static`和`Actor`. 它们各自响应不同的事件, 并且不同的`GameplayEffect`类型可以触发它们. 根据你的逻辑重写相关的事件.  

|                                                            GameplayCue类                                                            |     事件     | GameplayEffect类型  |                                                                                                            描述                                                                                                            |
| :--------------------------------------------------------------------------------------------------------------------------------: | :--------: | :---------------: | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
| [GameplayCueNotify_Static](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/UGameplayCueNotify_Static/index.html) |  Execute   | Instant或Periodic  |                                                                     `Static GameplayCueNotify`直接操作`ClassDefaultObject`(意味着没有实例)并且对于一次性效果(像击打伤害)是极好的.                                                                     |
|              [GameplayCueNotify_Actor](https://docs.unrealengine.com/en-US/BlueprintAPI/GameplayCueNotify/index.html)              | Add或Remove | Duration或Infinite | `Actor GameplayCueNotify`会在添加(Added)时生成一个新的实例, 因为其是实例化的, 所以可以随时间推移执行操作直到被移除(Removed). 这对循环的声音和粒子效果是很好的, 其会在`持续(Duration)`或`无限(Infinite)`GameplayEffect被移除或手动调用移除时移除. 其也自带选项来管理允许同时添加(Added)多少个, 因此多个相同效果的应用只启用一次声音或粒子效果. |

`GameplayCueNotify`技术上可以响应任何事件, 但是这是我们一般使用它的方式.  

**Note:** 当使用`GameplayCueNotify_Actor`时, 要勾选`Auto Destroy on Remove`, 否则之后对`GameplayCueTag`的`添加(Add)`调用将无法进行.  

当使用除`Full`之外的`ASC`[同步模式](#concepts-asc-rm)时, `Add`和`Remove`GC事件会在服务端玩家中触发两次(Listen Server) - 一次是应用`GE`, 再一次是从"Minimal"`NetMultiCast`到客户端. 然而, `WhileActive`事件仍会触发一次. 所有的事件在客户端中只触发一次.  

样例项目包含了一个`GameplayCueNotify_Actor`用于眩晕和奔跑效果. 其还含有一个`GameplayCueNotify_Static`用于枪支弹药伤害. 这些`GC`可以通过[客户端触发](#concepts-gc-local)来进行优化, 而不是通过`GE`同步. 我选择了在样例项目中展示使用它们的基本方法.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-gc-trigger"></a>
#### 4.8.2 触发GameplayCue

##### GameplayEffect

在成功应用(未被Tag或Immunity阻塞)的`GameplayEffect`中填写所有应该触发的`GameplayCue`的`GameplayTag`.  

![[attachments/03a822b3796c45dd813785eb48c0fdef_MD5.png]]  

##### 手动调用

`UGameplayAbility`提供了蓝图节点来`Execute`, `Add`或`Remove` GameplayCue.  

![[attachments/7c2109bfb643c979cab43b16eebe5dfb_MD5.png]]  

在C++中, 你可以在`ASC`中直接调用函数(或者在你的`ASC`子类中暴露它们到蓝图):  

```c++
/** GameplayCues can also come on their own. These take an optional effect context to pass through hit result, etc */
void ExecuteGameplayCue(const FGameplayTag GameplayCueTag, FGameplayEffectContextHandle EffectContext = FGameplayEffectContextHandle());
void ExecuteGameplayCue(const FGameplayTag GameplayCueTag, const FGameplayCueParameters& GameplayCueParameters);

/** Add a persistent gameplay cue */
void AddGameplayCue(const FGameplayTag GameplayCueTag, FGameplayEffectContextHandle EffectContext = FGameplayEffectContextHandle());
void AddGameplayCue(const FGameplayTag GameplayCueTag, const FGameplayCueParameters& GameplayCueParameters);

/** Remove a persistent gameplay cue */
void RemoveGameplayCue(const FGameplayTag GameplayCueTag);
	
/** Removes any GameplayCue added on its own, i.e. not as part of a GameplayEffect. */
void RemoveAllGameplayCues();
```

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-gc-local"></a>
#### 4.8.3 客户端GameplayCue

从`GameplayAbility`和`ASC`中暴露的用于触发`GameplayCue`的函数默认是可同步的. 每个`GameplayCue`事件都是一个多播(Multicast)RPC. 这会导致大量RPC. GAS也强制在每次网络更新中最多能有两个相同的`GameplayCue`RPC. 我们可以通过使用客户端`GameplayCue`来避免这个问题. 客户端`GameplayCue`只能在单独的客户端上`Execute`, `Add`或`Remove`.  

可以使用客户端`GameplayCue`的场景:  

* 抛射物伤害
* 近战碰撞伤害
* 动画蒙太奇触发的`GameplayCue`

你应该添加到`ASC`子类中的客户端`GameplayCue`函数:  

```c++
UFUNCTION(BlueprintCallable, Category = "GameplayCue", Meta = (AutoCreateRefTerm = "GameplayCueParameters", GameplayTagFilter = "GameplayCue"))
void ExecuteGameplayCueLocal(const FGameplayTag GameplayCueTag, const FGameplayCueParameters& GameplayCueParameters);

UFUNCTION(BlueprintCallable, Category = "GameplayCue", Meta = (AutoCreateRefTerm = "GameplayCueParameters", GameplayTagFilter = "GameplayCue"))
void AddGameplayCueLocal(const FGameplayTag GameplayCueTag, const FGameplayCueParameters& GameplayCueParameters);

UFUNCTION(BlueprintCallable, Category = "GameplayCue", Meta = (AutoCreateRefTerm = "GameplayCueParameters", GameplayTagFilter = "GameplayCue"))
void RemoveGameplayCueLocal(const FGameplayTag GameplayCueTag, const FGameplayCueParameters& GameplayCueParameters);
```

```c++
void UPAAbilitySystemComponent::ExecuteGameplayCueLocal(const FGameplayTag GameplayCueTag, const FGameplayCueParameters & GameplayCueParameters)
{
	UAbilitySystemGlobals::Get().GetGameplayCueManager()->HandleGameplayCue(GetOwner(), GameplayCueTag, EGameplayCueEvent::Type::Executed, GameplayCueParameters);
}

void UPAAbilitySystemComponent::AddGameplayCueLocal(const FGameplayTag GameplayCueTag, const FGameplayCueParameters & GameplayCueParameters)
{
	UAbilitySystemGlobals::Get().GetGameplayCueManager()->HandleGameplayCue(GetOwner(), GameplayCueTag, EGameplayCueEvent::Type::OnActive, GameplayCueParameters);
	UAbilitySystemGlobals::Get().GetGameplayCueManager()->HandleGameplayCue(GetOwner(), GameplayCueTag, EGameplayCueEvent::Type::WhileActive, GameplayCueParameters);
}

void UPAAbilitySystemComponent::RemoveGameplayCueLocal(const FGameplayTag GameplayCueTag, const FGameplayCueParameters & GameplayCueParameters)
{
	UAbilitySystemGlobals::Get().GetGameplayCueManager()->HandleGameplayCue(GetOwner(), GameplayCueTag, EGameplayCueEvent::Type::Removed, GameplayCueParameters);
}
```

如果某个`GameplayCue`是客户端添加的, 那么它也应该自客户端移除. 如果它是通过同步添加的, 那么它也应该通过同步移除.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-gc-parameters"></a>
#### 4.8.4 GameplayCue参数

`GameplayCue`接受一个包含额外`GameplayCue`信息的`FGameplayCueParameters`结构体作为参数. 如果你在`GameplayAbility`或`ASC`中使用函数手动触发`GameplayCue`, 那么就必须手动填充传递给`GameplayCue`的`GameplayCueParameters`结构体. 如果`GameplayCue`由`GameplayEffect`触发, 那么下列的变量会自动填充到`FGameplayCueParameters`结构体中:  

* AggregatedSourceTags
* AggregatedTargetTags
* GameplayEffectLevel
* AbilityLevel
* [EffectContext](#concepts-ge-context)
* Magnitude(如果`GameplayEffect`具有在`GameplayCue`标签容器(TagContainer)上方的下拉列表中选择的`Magnitude Attribute`和影响该`Attribute`的相应`Modifier`)

当手动触发`GameplayCue`时, `GameplayCueParameters`中的`SourceObject`变量似乎是一个传递任意数据到`GameplayCue`的好地方.  

**Note:** 参数结构体中的某些变量, 像`Instigator`, 可能已经存在于`EffectContext`中. `EffectContext`也可以包含`FHitResult`用于存储`GameplayCue`在世界中生成的位置. 子类化`EffectContext`似乎是一个传递更多数据到`GameplayCue`的好方法, 特别是对于那些由`GameplayEffect`触发的`GameplayCue`.  

查看[UAbilitySystemGlobals](#concepts-asg)中用于填充`GameplayCueParameters`结构体的三个函数以获得更多信息. 它们是虚函数, 因此你可以重写它们以自动填充更多信息.  

```c++
/** Initialize GameplayCue Parameters */
virtual void InitGameplayCueParameters(FGameplayCueParameters& CueParameters, const FGameplayEffectSpecForRPC &Spec);
virtual void InitGameplayCueParameters_GESpec(FGameplayCueParameters& CueParameters, const FGameplayEffectSpec &Spec);
virtual void InitGameplayCueParameters(FGameplayCueParameters& CueParameters, const FGameplayEffectContextHandle& EffectContext);
```

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-gc-manager"></a>
#### 4.8.5 Gameplay Cue Manager

默认情况下, 游戏开始时`GameplayCueManager`会扫描游戏的全部目录以寻找`GameplayCueNotify`并将其加载进内存. 我们可以设置`DefaultGame.ini`来修改`GameplayCueManager`的扫描路径.  

```c++
[/Script/GameplayAbilities.AbilitySystemGlobals]
GameplayCueNotifyPaths="/Game/GASDocumentation/Characters"
```

我们确实想要`GameplayCueManager`扫描并找到所有的`GameplayCueNotify`, 然而, 我们不想要它异步加载每一个, 因为这会将每个`GameplayCueNotify`和它们所引用的音效和粒子特效放入内存而不管它们是否在关卡中使用. 在像Paragon这样的大型游戏中, 内存中会放入数百兆的无用资源并造成卡顿和启动时无响应.  

在启动时异步加载每个`GameplayCue`的一种可选方法是只异步加载那些会在游戏中触发的`GameplayCue`, 这会在异步加载每个`GameplayCue`时减少不必要的内存占用和潜在的游戏无响应几率, 从而避免特定`GameplayCue`在游戏中第一次触发时可能出现的延迟效果. SSD不存在这种潜在的延迟, 我还没有在HDD上测试过, 如果在UE编辑器中使用该选项并且编辑器需要编译粒子系统的话, 就可能会在GameplayCue首次加载时有轻微的卡顿或无响应, 这在构建版本中不是问题, 因为粒子系统肯定是编译好的.  

首先我们必须继承`UGameplayCueManager`并告知`AbilitySystemGlobals`类在`DefaultGame.ini`中使用我们的`UGameplayCueManager`子类.  

```c++
[/Script/GameplayAbilities.AbilitySystemGlobals]
GlobalGameplayCueManagerClass="/Script/ParagonAssets.PBGameplayCueManager"
```

在我们的`UGameplayCueManager`子类中, 重写`ShouldAsyncLoadRuntimeObjectLibraries()`.  

```c++
virtual bool ShouldAsyncLoadRuntimeObjectLibraries() const override
{
	return false;
}
```

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-gc-prevention"></a>
#### 4.8.6 阻止GameplayCue响应

有时我们不想响应`GameplayCue`, 例如我们阻止了一次攻击, 可能就不想播放附加在伤害`GameplayEffect`上的击打效果或者自定义的效果. 我们可以在[GameplayEffectExecutionCalculations](#concepts-ge-ec)中调用`OutExecutionOutput.MarkGameplayCuesHandledManually()`, 之后手动发送我们的`GameplayCue`事件到`Target`或`Source`的`ASC`中.  

如果你想某个特别指定`ASC`中的`GameplayCue`永不触发, 可以设置`AbilitySystemComponent->bSuppressGameplayCues = true;`.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-gc-batching"></a>
#### 4.8.7 GameplayCue批处理

每次`GameplayCue`触发都是一次不可靠的多播(NetMulticast)RPC. 在同一时刻触发多个`GameplayCue`的情况下, 有一些优化方法来将它们压缩成一个RPC或者通过发送更少的数据来节省带宽.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-gc-batching-manualrpc"></a>
##### 4.8.7.1 手动RPC

假设你有一个可以发射8枚弹丸的霰弹枪, 就会有8个轨迹和碰撞`GameplayCue`. GASShooter采用将它们联合成一个RPC的延迟(Lazy)方法, 其将所有的轨迹信息保存到[EffectContext](#concepts-ge-ec)作为[TargetData](#concepts-targeting-data). 尽管其将RPC数量从8降为1, 然而还是在这一个RPC中通过网络发送大量数据(~500 bytes). 一个进一步优化的方法是使用一个自定义结构体发送RPC, 在该自定义RPC中你需要高效编码命中位置(Hit Location)或者给一个随机种子以在接收端重现/近似计算碰撞位置, 客户端之后需要解包该自定义结构体并重现[客户端执行的`GameplayCue`](#concepts-gc-local).  

运行机制:  

1. 声明一个`FScopedGameplayCueSendContext`. 其会阻塞`UGameplayCueManager::FlushPendingCues()`直到其出域, 意味着所有`GameplayCue`都将会排队等候直到该`FScopedGameplayCueSendContext`出域.
2. 重写`UGameplayCueManager::FlushPendingCues()`以将那些可以基于一些自定义`GameplayTag`批处理的`GameplayCue`合并进自定义的结构体并将其RPC到客户端.
3. 客户端接收自定义结构体并将其解包进客户端执行的`GameplayCue`.

该方法也可以在你的`GameplayCue`需要特别指定的参数时使用, 这些需要特别指定的参数不能由`GameplayCueParameter`提供, 并且你不想将它们添加到`EffectContext`, 像伤害数值, 暴击标识, 破盾标识, 处决标识等等.  

[https://forums.unrealengine.com/development-discussion/c-gameplay-programming/1711546-fscopedgameplaycuesendcontext-gameplaycuemanager](https://forums.unrealengine.com/development-discussion/c-gameplay-programming/1711546-fscopedgameplaycuesendcontext-gameplaycuemanager)  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-gc-batching-gcsonge"></a>
##### 4.8.7.2 GameplayEffect中的多个GameplayCue

一个`GameplayEffect`中的所有`GameplayCue`已经由一个RPC发送了. 默认情况下, `UGameplayCueManager::InvokeGameplayCueAddedAndWhileActive_FromSpec()`会在不可靠的多播(NetMulticast)RPC中发送整个`GameplayEffectSpec`(除了转换为`FGameplayEffectSpecForRPC`)而不管`ASC`的同步模式, 取决于`GameplayEffectSpec`的内容, 这可能会使用大量带宽, 我们可以通过设置`AbilitySystem.AlwaysConvertGESpecToGCParams 1`来将其优化, 这会将`GameplayEffectSpec`转换为`FGameplayCueParameter`结构体并且RPC它而不是整个`FGameplayEffectSpecForRPC`, 这会节省带宽但是只有较少的信息, 取决于`GESpec`如何转换为`GameplayCueParameters`和你的`GameplayCue`需要知道什么.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-asg"></a>
### 4.9 Ability System Globals

[AbilitySystemGlobals](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/UAbilitySystemGlobals/index.html)类保存有关GAS的全局信息. 大多数变量可以在`DefaultGame.ini`中设置. 一般你不需要和该类互动, 但是应该知道它的存在. 如果你需要继承像[GameplayCueManager](#concepts-gc-manager)或[GameplayEffectContext](#concepts-ge-context)这样的对象, 就必须通过`AbilitySystemGlobals`来做.  

想要继承`AbilitySystemGlobals`, 需要在`DefaultGame.ini`中设置类名:  

```c++
[/Script/GameplayAbilities.AbilitySystemGlobals]
AbilitySystemGlobalsClassName="/Script/ParagonAssets.PAAbilitySystemGlobals"
```

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-asg-initglobaldata"></a>
#### 4.9.1 InitGlobalData()

从UE 4.24开始, 必须调用`UAbilitySystemGlobals::InitGlobalData()`来使用[TargetData](#concepts-targeting-data), 否则你会遇到关于`ScriptStructCache`的错误, 并且客户端会从服务端断开连接, 该函数只需要在项目中调用一次. Fortnite从AssetManager类的起始加载函数中调用该函数, Paragon是从UEngine::Init()中调用的. 我发现将其放到`UEngineSubsystem::Initialize()`是个好位置, 这也是样例项目中使用的. 我觉得你应该复制这段模板代码到你自己的项目中以避免出现`TargetData`的使用问题.  

如果你在使用`AbilitySystemGlobals GlobalAttributeSetDefaultsTableNames`时发生崩溃, 可能之后需要像Fortnite一样在`AssetManager`或`GameInstance`中调用`UAbilitySystemGlobals::InitGlobalData()`而不是在`UEngineSubsystem::Initialize()`中. 该崩溃可能是由于`Subsystem`的加载顺序引发的, `GlobalAttributeDefaultsTables`需要加载`EditorSubsystem`来绑定`UAbilitySystemGlobals::InitGlobalData()`中的委托.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-p"></a>
### 4.10 预测(Prediction)

GAS带有开箱即用的客户端预测功能, 然而, 它不能预测所有. GAS中客户端预测的意思是客户端无需等待服务端的许可而激活`GameplayAbility`和应用`GameplayEffect`. 它可以"预测"许可其可以这样做的服务端和其应用`GameplayEffect`的目标. 服务端在客户端激活之后运行`GameplayAbility`(网络延迟)并告知客户端它的预测是否正确, 如果客户端的预测出错, 那么它就会"回滚"其"错误预测"的修改以匹配服务端.  

GAS相关预测的最佳源码是插件源码中的`GameplayPrediction.h`.  

Epic的理念是只能预测"不受伤害(get away with)"的事情. 例如, Paragon和Fortnite不能预测伤害值, 它们很可能对伤害值使用了[ExecutionCalculations](#concepts-ge-ec), 而这无论如何是不能预测的. 这并不是说你不能试着预测像伤害值这样的事情, 无论如何, 如果你这样做了并且效果很好, 那就是极好的.  

> ... we are also not all in on a "predict everything: seamlessly and automatically" solution. We still feel player prediction is best kept to a minimum (meaning: predict the minimum amount of stuff you can get away with).

*来自Epic的Dave Ratti在新的[网络预测插件](#concepts-p-npp)中的注释.*  

**什么是可预测的:**  

* Ability激活
* 触发事件
* GameplayEffect应用
	+ Attribute修改(例外: 执行(Execution)目前无法预测, 只能是Attribute Modifiy)
	+ GameplayTag修改
* GameplayCue事件(可预测GameplayEffect中的和它们自己)
* 蒙太奇
* 移动(内建在UE4 UCharacterMovement中)

**什么是不可预测的:**  

* GameplayEffect移除
* GameplayEffect周期效果(dots ticking)

*源自`GameplayPrediction.h`*  

尽管我们可以预测`GameplayEffect`的应用, 但是不能预测`GameplayEffect`的移除. 绕过这条限制的一个方法是当想要移除`GameplayEffect`时, 可以预测性地执行相反的效果, 假设我们要降低40%移动速度, 可以通过应用增加40%移动速度的buff来预测性地将其移除, 之后同时移除这两个`GameplayEffect`. 这并不是对每个场景都适用, 因此仍然需要对预测性地移除`GameplayEffect`的支持. 来自Epic的Dave Ratti已经表达过在[GAS的迭代版本中](https://epicgames.ent.box.com/s/m1egifkxv3he3u3xezb9hzbgroxyhx89)增加它的期望.  

因为我们不能预测`GameplayEffect`的移除, 所以就不能完全预测`GameplayAbility`的冷却时间, 并且也没有相反的`GameplayEffect`这种变通方案可供使用. 服务端同步的`Cooldown GE`将会存于客户端上, 并且任何对其绕过的尝试(例如使用`Minimal`同步模式)都会被服务端拒绝. 这意味着高延迟的客户端会花费较长事件来告知服务端开始冷却和接收到服务端`Cooldown GE`的移除. 这意味着高延迟的玩家会比低延迟的玩家有更低的触发率, 从而劣势于低延迟玩家. Fortnite通过使用自定义Bookkeeping而不是`Cooldown GE`的方法避免了该问题.  

关于预测伤害值, 我个人不建议使用, 尽管它是大多数刚接触GAS的人最先做的事情之一, 我特别不建议尝试预测死亡, 虽然你可以预测伤害值, 但是这样做很棘手. 如果你错误预测地应用了伤害, 那么玩家将会看到敌人的生命值会跳动, 如果你尝试预测死亡, 那么这将会是特别尴尬和令人沮丧的, 假设你错误预测了某个`Character`的死亡, 那么它就会开启布娃娃模拟, 只有当服务端纠正后才会停止布娃娃模拟并继续向你射击.

**Note:** 修改`Attribute`的`即刻(Instant)GameplayEffect`(像`Cost GE`)在你自身可以无缝预测, 预测修改其他Character的`即刻(Instant)Attribute`会显示短暂的异常或者`Attribute`中的暂时性问题. 预测的`即刻(Instant)GameplayEffect`实际上被视为`无限(Infinite)GameplayEffect`, 因此如果错误预测的话还可以回滚. 当服务端的`GameplayEffect`被应用时, 其实是存在两个相同的`GameplayEffect`的, 会在短时间内造成`Modifier`应用两次或者不应用, 其最终会纠正自身, 但是有时候这个异常现象对玩家来说是显著的.  

GAS的预测实现尝试解决的问题:  

> 1. "我能这样做吗?"(Can I do this?) 预测的基本协议.
> 2. "撤销"(Undo) 当预测错误时如何消除其副作用.
> 3. "重现"(Redo) 如何避免已经在客户端预测但还是从服务端同步的重播副作用.
> 4. "完整"(Completeness) 如何确认我们真的预测了所有副作用.
> 5. "依赖"(Dependencies) 如何管理依赖预测和预测事件链.
> 6. "重写"(Override) 如何预测地重写由服务端同步/拥有的状态.

源自`GameplayPrediction.h`  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-p-key"></a>
#### 4.10.1 Prediction Key

GAS的预测建立在`Prediction Key`的概念上, 其是一个由客户端激活`GameplayAbility`时生成的整型标识符.  

* 客户端激活`GameplayAbility`时生成`Prediction Key`, 这是`Activation Prediction Key`.  
* 客户端使用`CallServerTryActivateAbility()`将该`Prediction Key`发送到服务端.
* 客户端在`Prediction Key`有效时将其添加到应用的所有`GameplayEffect`.
* 客户端的`Prediction Key`出域. 之后该`GameplayAbility`中的预测效果(Effect)需要一个新的[Scoped Prediction Window](#concepts-p-windows).
* 服务端从客户端接收`Prediction Key`.
* 服务端将`Prediction Key`添加到其应用的所有`GameplayEffect`.
* 服务端同步该`Prediction Key`回客户端.
* 客户端使用`Prediction Key`从服务端接收同步的`GameplayEffect`, 该`Prediction Key`用于应用`GameplayEffect`. 如果任何同步的`GameplayEffect`与客户端使用相同`Prediction Key`应用的`GameplayEffect`相匹配, 那么其就是正确预测的. 目标上暂时会有`GameplayEffect`的两份拷贝直到客户端移除它预测的那一个.
* 客户端从服务端上接收回`Prediction Key`, 这是同步的`Prediction Key`, 该`Prediction Key`现在被标记为陈旧(Stale).
* 客户端移除所有由同步的陈旧(Stale)`Prediction Key`创建的`GameplayEffect`. 由服务端同步的`GameplayEffect`会持续存在. 任何客户端添加的和没有从服务端接收到一个匹配的同步版本的都被视为错误预测.

在源于`Activation Prediction Key`激活的`GameplayAbility`中的一个`instruction "window"`原子(Atomic)组期间, `Prediction Key`是保证有效的, 你可以理解为`Prediction Key`只在一帧期间是有效的. 任何潜在行为`AbilityTask`的回调函数将不再拥有一个有效的`Prediction Key`, 除非该`AbilityTask`有内建的可以生成新[Scoped Prediction Window](#concepts-p-windows)的同步点(Sync Point).  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-p-windows"></a>
#### 4.10.2 在Ability中创建新的预测窗口(Prediction Window)

为了在`AbilityTask`的回调函数中预测更多的行为, 我们需要使用新的`Scoped Prediction Key`创建`Scoped Prediction Window`, 有时这被视为客户端和服务端间的同步点(Sync Point). 一些`AbilityTask`, 像所有输入相关的`AbilityTask`, 带有创建新`Scoped Prediction Window`的内建功能, 意味着`AbilityTask`回调函数中的原子(Atomic)代码有一个有效的`Scoped Prediction Key`可供使用. 像`WaitDelay`的其他Task没有创建新`Scoped Prediction Window`以用于回调函数的内建代码, 如果你需要在`WaitDelay`这样的`AbilityTask`后预测行为, 就必须使用`OnlyServerWait`选项的`WaitNetSync AbilityTask`手动来做, 当客户端触发`OnlyServerWait`选项的`WaitNetSync`时, 它会生成一个新的基于`GameplayAbility`的`Activation Prediction Key`的`Scoped Prediction Key`, RPC其到服务端, 并将其添加到所有新应用的`GameplayEffect`. 当服务端触发`OnlyServerWait`选项的`WaitNetSync`时, 它会在继续前等待直到接收到客户端新的`Scoped Prediction Key`, 该`Scoped Prediction Key`会执行和`Activation Prediction Key`同样的操作 —— 应用到`GameplayEffect`并同步回客户端标记为陈旧(Stale). `Scoped Prediction Key`直到出域前都有效, 也就表示`Scoped Prediction Window`已经关闭了. 所以只有原子(Atomic)操作, nothing latent, 可以使用新的`Scoped Prediction Key`.  

你可以根据需求创建任意数量的`Scoped Prediction Window`.  

如果你想添加同步点(Sync Point)功能到自己的自定义`AbilityTask`, 请查看那些输入`AbilityTask`是如何从根本上注入`WaitNetSync AbilityTask`代码到自身的.  

**Note:** 当使用`WaitNetSync`时, 会阻塞服务端`GameplayAbility`继续执行直到其接收到客户端的消息. 这可能会被破解游戏的恶意用户滥用以故意延迟发送新的`Scoped Prediction Key`, 尽管Epic很少使用`WaitNetSync`, 但如果你对此担忧的话, 其建议创建一个带有延迟的新`AbilityTask`, 它会自动继续运行而无需等待客户端消息.  

样例项目在奔跑`GameplayAbility`中使用了`WaitNetSync`以在每次应用耐力花费时创建新的`Scoped Prediction Window`, 这样我们就可以进行预测. 理想上当应用花费和冷却时间时我们就想要一个有效的`Prediction Key`.  

如果你有一个在所属(Owning)客户端执行两次的预测`GameplayEffect`, 那么你的`Prediction Key`就是陈旧(Stall)的, 并且正在经历"redo"问题. 你通常可以在应用`GameplayEffect`之前将`OnlyServerWait`选项的`WaitNetSync AbilityTask`放到正确的位置以创建新的`Scoped Prediction Key`来解决这个问题.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-p-spawn"></a>
#### 4.10.3 预测性地生成Actor

在客户端预测性地生成Actor是一项高级技术. GAS对此没有提供开箱即用的功能(`SpawnActor AbilityTask`只在服务端生成Actor). 其关键是在客户端和服务端都生成同步的Actor.  

如果Actor只是用于场景装饰或者不服务于任何游戏逻辑, 简单的解决方案就是重写Actor的`IsNetRelevantFor()`函数以限制服务端同步到所属(Owning)客户端, 所属(Owning)客户端会拥有其本地生成的版本, 而服务端和其他客户端会拥有服务端同步的版本.  

```c++
bool APAReplicatedActorExceptOwner::IsNetRelevantFor(const AActor * RealViewer, const AActor * ViewTarget, const FVector & SrcLocation) const
{
	return !IsOwnedBy(ViewTarget);
}
```

如果生成的Actor影响了游戏逻辑, 像投掷物就需要预测伤害值, 那么你需要本文档范围之外的高级知识, 在Epic Games的Github上查看UnrealTournament是如何生成投掷物的, 它有一个只在所属(Owning)客户端生成且与服务端同步的投掷物.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-p-future"></a>
#### 4.10.4 GAS中预测的未来

`GameplayPrediction.h`说明了在未来可能会增加预测`GameplayEffect`移除和周期`GameplayEffect`的功能.  

来自Epic的Dave Ratti已经[表达过对其修复的兴趣](https://epicgames.ent.box.com/s/m1egifkxv3he3u3xezb9hzbgroxyhx89), 包括预测冷却时间时的延迟问题和高延迟玩家对低延迟玩家的劣势.  

来自Epic之手的新[网络预测插件(Network Prediction plugin)](#concepts-p-npp)期望能与GAS充分交互, 就像在次之前的`CharacterMovementComponent`.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-p-npp"></a>
#### 4.10.5 网络预测插件(Network Prediction plugin)

Epic最近发起了一项倡议, 将使用新的网络预测插件替换`CharacterMovementComponent`, 该插件仍处于起步阶段, 但是在Unreal Engine Github上已经可以访问了, 现在说未来哪个引擎版本将首次搭载其试验版还为时尚早.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-targeting"></a>
### 4.11 Targeting

<a name="concepts-targeting-data"></a>
#### 4.11.1 Target Data

[FGameplayAbilityTargetData](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/Abilities/FGameplayAbilityTargetData/index.html)是用于通过网络传输定位数据的通用结构体. `TargetData`一般用于保存AActor/UObject引用, FHitResult和其他通用的Location/Direction/Origin信息. 然而, 本质上你可以继承它以增添想要的任何数据, 其可以简单理解为在[客户端和服务端的`GameplayAbility`中传递数据](#concepts-ga-data). 基础结构体`FGameplayAbilityTargetData`不能直接使用, 而是要继承它. GAS的`GameplayAbilityTargetTypes.h`中有一些开箱即用的派生`FGameplayAbilityTargetData`结构体.  

`TargetData`一般由[Target Actor](#concepts-targeting-actors)或者手动创建, 供[AbilityTask](#concepts-at)使用, 或者[GameplayEffect](#concepts-ge)通过[EffectContext](#concepts-ge-context)使用. 因为其位于`EffectContext`中, 所以[Execution](#concepts-ge-ec), [MMC](#concepts-ge-mmc), [GameplayCue](#concepts-gc)和[AttributeSet](#concepts-as)的后端函数可以访问该`TargetData`.  

我们一般不直接传递`FGameplayAbilityTargetData`而是使用[FGameplayAbilityTargetDataHandle](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/Abilities/FGameplayAbilityTargetDataHandle/index.html), 其包含一个`FGameplayAbilityTargetData`指针类型的TArray, 这个中间结构体可以为`TargetData`的多态性提供支持.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-targeting-actors"></a>
#### 4.11.2 Target Actor

`GameplayAbility`使用`WaitTargetData AbilityTask`生成[TargetActor](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/Abilities/AGameplayAbilityTargetActor/index.html)以在世界中可视化和获取定位信息. `TargetActor`可以选择使用[GameplayAbilityWorldReticles](#concepts-targeting-reticles)显示当前目标. 经确认后, 定位信息将作为[TargetData](#concepts-targeting-data)返回, 之后其可以传递给`GameplayEffect`.  

`TargetActor`是基于`AActor`的, 因此它可以使用任意种类的可视组件来表示其在何处和如何定位的, 例如静态网格物(Static Mesh)和贴花(Decal). 静态网格物(Static Mesh)可以用来可视化角色将要建造的物体, 贴花(Decal)可以用来表现地面上的效果区域. 样例项目使用带有地面贴花的[AGameplayAbilityTargetActor_GroundTrace](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/Abilities/AGameplayAbilityTargetActor_Grou-/index.html)表示陨石技能的伤害区域效果. 它也可以什么都不显示, 例如, GASShooter中的枪支命中判断, 要对目标的射线检测显示些什么就是没有意义的.  

其使用基本的射线或者Overlap捕获定位信息, 并根据`TargetActor`的实现将`FHitResult`或`AActor`数组转换为`TargetData`. `WaitTargetData AbilityTask`通过`TEnumAsByte<EGameplayTargetingConfirmation::Type> ConfirmationType`参数决定目标何时被确认, 当不使用`TEnumAsByte<EGameplayTargetingConfirmation::Type::Instant`时, `TargetActor`一般就在`Tick()`中执行射线/Overlap, 并根据它的实现来更新位置信息到`FHitResult`. 尽管是在`Tick()`中执行的射线/Overlap, 但是一般不用担心, 因为它是不同步的并且一般没有多个(尽管存在多个)`TargetActor`同时运行, 只是要留意它使用的是`Tick()`, 一些复杂的`TargetActor`可能会在其中做很多事情, 就像GASShooter中火箭筒的二技能. 当`Tick()`中的检测对客户端响应非常频繁时, 如果性能影响很大的话, 你可能就要考虑降低`TargetActor`的Tick频率. 对于`TEnumAsByte<EGameplayTargetingConfirmation::Type::Instant`, `TargetActor`会立即生成, 产生`TargetData`, 然后销毁, 并且从不会调用`Tick()`.  

|EGameplayTargetingConfirmation::Type|何时确认Target|
|:-:|:-:|
|Instant|该定位无需特殊逻辑即可立即进行, 或者用户输入决定何时开始.|
|UserConfirmed|当[Ability绑定到`Confirm`输入](#concepts-ga-input)且用户确认或调用`UAbilitySystemComponent::TargetConfirm()`时触发该定位. `TargetActor`也会响应绑定的`Cancel`输入或者调用`UAbilitySystemComponent::TargetCancel()`来取消定位.|
|Custom|GameplayTargeting Ability负责调用`UGameplayAbility::ConfirmTaskByInstanceName()`来决定何时准备好定位数据. `TargetActor`也可以响应`UGameplayAbility::CancelTaskByInstanceName()`来取消定位.|
|CustomMulti|GameplayTargeting Ability负责调用`UGameplayAbility::ConfirmTaskByInstanceName()`来决定何时准备好定位数据. `TargetActor`也可以响应`UGameplayAbility::CancelTaskByInstanceName()`来取消定位. 不应在数据生成后就结束AbilityTask, 因为其允许多次确认.|

并不是所有的`TargetActor`都支持每个`EGameplayTargetingConfirmation::Type`, 例如, `AGameplayAbilityTargetActor_GroundTrace`就不支持`Instant`确认.  

`WaitTargetData AbilityTask`将一个`AGameplayAbilityTargetActor`类作为参数, 其会在每次`AbilityTask`激活时生成一个实例并且在`AbilityTask`结束时销毁该`TargetActor`. `WaitTargetDataUsingActor AbilityTask`接受一个已经生成的`TargetActor`, 但是在该`AbilityTask`结束时仍会销毁它. 这两种`AbilityTask`都是低效的, 因为它们在每次使用时都要生成或需要一个新生成的`TargetActor`, 它们用于原型开发是很好的, 但是在实际发布版本中, 如果有持续产生`TargetData`的场景, 像全自动开火, 你可能就要探索优化它的办法. GASShooter有一个自定义的[AGameplayAbilityTargetActor](https://github.com/tranek/GASShooter/blob/master/Source/GASShooter/Public/Characters/Abilities/GSGATA_Trace.h)子类和一个完全重写的[WaitTargetDataWithReusableActor AbilityTask](https://github.com/tranek/GASShooter/blob/master/Source/GASShooter/Public/Characters/Abilities/AbilityTasks/GSAT_WaitTargetDataUsingActor.h), 其允许你复用`TargetActor`而无需将其销毁.  

`TargetActor`默认是不可同步的. 然而, 如果在你的游戏中向其他玩家展示本地玩家正在定位的目标是有意义的, 那么它也可以被设计成可同步的, `WaitTargetData AbilityTask`也确实包含其通过RPC和服务端通信的默认功能. 如果`TargetActor`的`ShouldProduceTargetDataOnServer`属性设置为false, 那么客户端就会在确认时通过`UAbilityTask_WaitTargetData::OnTargetDataReadyCallback()`中的`CallServerSetReplicatedTargetData()`RPC它的`TargetData`到服务端. 如果`ShouldProduceTargetDataOnServer`为true, 那么客户端就会发送一个通用确认事件, `EAbilityGenericReplicatedEvent::GenericConfirm`, 在`UAbilityTask_WaitTargetData::OnTargetDataReadyCallback()`中RPC到服务端, 服务端在接收到RPC时就会执行射线或者Overlap检测以生成数据. 如果客户端取消该定位, 它会发送一个通用取消事件, `EAbilityGenericReplicatedEvent::GenericCancel`, 在`UAbilityTask_WaitTargetData::OnTargetDataCancelledCallback`中RPC到服务端. 你可以看到, 在`TargetActor`和`WaitTargetData AbilityTask`中存在大量委托, `TargetActor`响应输入来产生广播`TargetData`的准备, 确认或者取消委托, `WaitTargetData`监听`TargetActor`的`TargetData`的准备, 确认和取消委托, 并将该信息转发回`GameplayAbility`和服务端. 如果是向服务端发送`TargetData`, 为了防止作弊, 可能需要在服务端做校验以保证该`TargetData`是合理的. 直接在服务端上产生`TargetData`可以完全避免这个问题, 但是可能会导致所属(Owning)客户端的错误预测.  

根据使用的不同`AGameplayAbilityTargetActor`子类, `WaitTargetData AbilityTask`节点会暴露不同的`ExposeOnSpawn`参数, 一些常用的参数包括:  

|常用`TargetActor`参数|定义|
|:-:|:-:|
|Debug|如果为真, 每当非发行版本中的`TargetActor`执行射线检测时, 其会绘制debug射线/Overlap信息. 请记住, `non-Instant TargetActor`会在`Tick()`中执行射线检测, 因此这些debug绘制调用也会在`Tick()`中触发.|
|Filter|[可选]当射线/Overlap触发时, 用于从Target中过滤(移除)Actor的特殊结构体. 典型的使用案例是过滤玩家的`Pawn`, 其要求Target是特殊类. 查看[Target Data Filters](#concepts-target-data-filters)以获得更多高级使用案例.|
|Reticle Class|[可选]`TargetActor`生成的`AGameplayAbilityWorldReticle`子类.|
|Reticle Parameters|[可选]配置你的Reticle. 查看[Reticles](#concepts-targeting-reticles).|
|Start Location|用于设置射线检测应该从何处开始的特殊结构体. 一般这应该是玩家的视口(Viewport), 枪口或者Pawn的位置.|

使用默认的`TargetActor`类时, Actor只有直接在射线/Overlap中时才是有效的, 如果它离开射线/Overlap(它移动开或你的视线转向别处), 就不再有效了. 如果你想`TargetActor`记住最后有效的Target, 就需要添加这项功能到一个自定义的`TargetActor`类. 我称之为持久化Target, 因为其会持续存在直到`TargetActor`接收到确认或取消消息, `TargetActor`会在它的射线/Overlap中找到一个新的有效Target, 或者Target不再有效(已销毁). GASShooter对火箭筒二技能的制导火箭定位使用了持久化Target.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-target-data-filters"></a>
#### 4.11.3 TargetData过滤器

同时使用`Make GameplayTargetDataFilter`和`Make Filter Handle`节点, 你可以过滤玩家的`Pawn`或者只选择某个特定类. 如果需要更多高级过滤条件, 可以继承`FGameplayTargetDataFilter`并重写`FilterPassesForActor`函数.  

```c++
USTRUCT(BlueprintType)
struct GASDOCUMENTATION_API FGDNameTargetDataFilter : public FGameplayTargetDataFilter
{
	GENERATED_BODY()

	/** Returns true if the actor passes the filter and will be targeted */
	virtual bool FilterPassesForActor(const AActor* ActorToBeFiltered) const override;
};
However, this will not work directly into the Wait Target Data node as it requires a FGameplayTargetDataFilterHandle. A new custom Make Filter Handle must be made to accept the subclass:

FGameplayTargetDataFilterHandle UGDTargetDataFilterBlueprintLibrary::MakeGDNameFilterHandle(FGDNameTargetDataFilter Filter, AActor* FilterActor)
{
	FGameplayTargetDataFilter* NewFilter = new FGDNameTargetDataFilter(Filter);
	NewFilter->InitializeFilterContext(FilterActor);

	FGameplayTargetDataFilterHandle FilterHandle;
	FilterHandle.Filter = TSharedPtr<FGameplayTargetDataFilter>(NewFilter);
	return FilterHandle;
}
```

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-targeting-reticles"></a>
#### 4.11.4 Gameplay Ability World Reticles

当使用已确认的`non-Instant` [TargetActor](#concepts-targeting-actors)定位时, [AGameplayAbilityWorldReticles(Reticles)](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/Abilities/AGameplayAbilityWorldReticle/index.html)可以可视化正在定位的目标. `TargetActor`负责所有`Reticle`生命周期的生成和销毁. `Reticle`是AActor, 因此其可以使用任意种类的可视组件作为表现形式. GASShooter中常见的一种实现方式是使用`WidgetComponent`在屏幕空间中显示UMG Widget(永远面对玩家的摄像机). `Reticle`不知道其正在定位的Actor, 但是你可以通过继承在自定义`TargetActor`中实现该功能. `TargetActor`一般在每次`Tick()`中更新`Reticle`的位置为Target的位置.  

GASShooter对火箭筒二技能制导火箭锁定的目标使用了`Reticle`. 敌人身上的红色标识就是`Reticle`, 相似的白色图像是火箭筒的准星.  

![[attachments/7a05152ed5871153b641841e4f366f1c_MD5.png]]  

`Reticle`带有一些面向设计师的`BlueprintImplementableEvents`(它们被设计用来在蓝图中开发):  

```c++
/** Called whenever bIsTargetValid changes value. */
UFUNCTION(BlueprintImplementableEvent, Category = Reticle)
void OnValidTargetChanged(bool bNewValue);

/** Called whenever bIsTargetAnActor changes value. */
UFUNCTION(BlueprintImplementableEvent, Category = Reticle)
void OnTargetingAnActor(bool bNewValue);

UFUNCTION(BlueprintImplementableEvent, Category = Reticle)
void OnParametersInitialized();

UFUNCTION(BlueprintImplementableEvent, Category = Reticle)
void SetReticleMaterialParamFloat(FName ParamName, float value);

UFUNCTION(BlueprintImplementableEvent, Category = Reticle)
void SetReticleMaterialParamVector(FName ParamName, FVector value);
```

`Reticle`可以选择使用`TargetActor`提供的[FWorldReticleParameters](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/Abilities/FWorldReticleParameters/index.html)来配置, 默认结构体只提供一个变量`FVector AOEScale`, 尽管你可以在技术上继承该结构体, 但是`TargetActor`只接受基类结构体, 不允许在默认`TargetActor`中子类化该结构体似乎有些短见, 然而, 如果你创建了自己的自定义`TargetActor`, 就可以提供自定义的`Reticle`参数结构体并在生成`Reticle`时手动传递它到`AGameplayAbilityWorldReticles`子类.  

`Reticle`默认是不可同步的, 但是如果在你的游戏中向其他玩家展示本地玩家正在定位的目标是有意义的, 那么它也可以被设计成可同步的.  

`Reticle`只会显示在默认`TargetActor`的当前有效Target上, 例如, 如果你正在使用`AGameplayAbilityTargetActor_SingleLineTrace`对目标执行射线检测, 敌人只有直接处于射线路径上时`Reticle`才会显示, 如果角色视线看向别处, 那么该敌人就不再是一个有效Target, 因此该`Reticle`就会消失. 如果你想`Reticle`保留在最后一个有效Target上, 就需要自定义`TargetActor`来记住最后一个有效Target并使`Reticle`保留在其上. 我称之为持久化Target, 因为其会持续存在直到`TargetActor`接收到确认或取消消息, `TargetActor`会在它的射线/Overlap中找到一个新的有效Target, 或者Target不再有效(已销毁). GASShooter对火箭筒二技能的制导火箭定位使用了持久化Target.  

**[⬆ 返回目录](#table-of-contents)**

<a name="concepts-targeting-containers"></a>
#### 4.11.5 Gameplay Effect Containers Targeting

[GameplayEffectContainer](#concepts-ge-containers)提供了一个可选的产生[TargetData](#concepts-targeting-data)的高效方法. 当`EffectContainer`在客户端和服务端上应用时, 该定位会立即进行, 它比[TargetActor](#concepts-targeting-actors)更有效率, 因为它是运行在定位对象的CDO(Class Default Object)上的(没有Actor的生成和销毁), 但是它不支持用户输入, 无需确认即可立即进行, 不能取消, 并且不能从客户端向服务端发送数据(在两者上产生数据), 它对即时射线检测和碰撞Overlap很有用. Epic的[Action RPG Sample Project](https://www.unrealengine.com/marketplace/en-US/slug/action-rpg)包含两种使用Container定位的样例 —— 定位Ability拥有者和从事件拉取`TargetData`, 它还在蓝图中实现了在距玩家某个偏移处(由蓝图子类设置)做球形射线检测(Sphere Trace), 你可以在C++或蓝图中继承`URPGTargetType`以实现自己的定位类型.  

**[⬆ 返回目录](#table-of-contents)**

<a name="cae"></a>
## 5. 常用的Abilty和Effect

<a name="cae-stun"></a>
### 5.1 眩晕(Stun)

一般在眩晕状态时, 我们就要取消Character所有已激活的`GameplayAbility`, 阻止新的`GameplayAbility`激活, 并在整个眩晕期间阻止移动. 样例项目的陨石`GameplayAbility`会在击中的目标上应用眩晕效果.  

为了取消目标活跃的`GameplayAbility`, 可以在眩晕[`GameplayTag`添加](#concepts-gt-change)时调用`AbilitySystemComponent->CancelAbilities()`.  

为了在眩晕时阻止新的`GameplayAbility`激活, 可以在`GameplayAbility`的[Activation Blocked Tags GameplayTagContainer](#concepts-ga-tags)中添加眩晕`GameplayTag`.  

为了在眩晕时阻止移动, 我们可以在拥有者拥有眩晕`GameplayTag`时重写`CharacterMovementComponent`的`GetMaxSpeed()`函数以返回0.  

**[⬆ 返回目录](#table-of-contents)**

<a name="cae-sprint"></a>
### 5.2 奔跑(Sprint)

样例项目提供了一个如何奔跑的例子 —— 按住左Shift时跑得更快.  

更快的移动由`CharacterMovementComponent`通过网络发送flag到服务端预测性地处理. 详见`GDCharacterMovementComponent.h/cpp`.  

`GA`处理响应左Shift输入, 告知`CMC`开始和停止奔跑, 并且在左Shift按下时预测性地消耗耐力. 详见`GA_Sprint_BP`.  

**[⬆ 返回目录](#table-of-contents)**

<a name="cae-ads"></a>
### 5.3 瞄准(Aim Down Sight)

样例项目处理它和奔跑时完全一样, 除了降低移动速度而不是提高它.  

详见`GDCharacterMovementComponent.h/cpp`是如何预测性地降低移动速度的.  

详见`GA_AimDownSight_BP`是如何处理输入的. 瞄准时是不消耗耐力的.  

**[⬆ 返回目录](#table-of-contents)**

<a name="cae-ls"></a>
### 5.4 生命偷取(Lifesteal)

我在伤害[ExecutionCalculation](#concepts-ge-ec)中处理生命偷取. `GameplayEffect`会有一个像`Effect.CanLifesteal`样的`GameplayTag`, `ExecutionCalculation`会检查`GameplayEffectSpec`是否有`Effect.CanLifesteal GameplayTag`, 如果该`GameplayTag`存在, `ExecutionCalculation`会使用作为Modifer的生命值[创建一个动态的`即刻(Instant)GameplayEffect`](#concepts-ge-dynamic), 并将其应用回源(Source)ASC.  

```c++
if (SpecAssetTags.HasTag(FGameplayTag::RequestGameplayTag(FName("Effect.Damage.CanLifesteal"))))
{
	float Lifesteal = Damage * LifestealPercent;

	UGameplayEffect* GELifesteal = NewObject<UGameplayEffect>(GetTransientPackage(), FName(TEXT("Lifesteal")));
	GELifesteal->DurationPolicy = EGameplayEffectDurationType::Instant;

	int32 Idx = GELifesteal->Modifiers.Num();
	GELifesteal->Modifiers.SetNum(Idx + 1);
	FGameplayModifierInfo& Info = GELifesteal->Modifiers[Idx];
	Info.ModifierMagnitude = FScalableFloat(Lifesteal);
	Info.ModifierOp = EGameplayModOp::Additive;
	Info.Attribute = UPAAttributeSetBase::GetHealthAttribute();

	SourceAbilitySystemComponent->ApplyGameplayEffectToSelf(GELifesteal, 1.0f, SourceAbilitySystemComponent->MakeEffectContext());
}
```

**[⬆ 返回目录](#table-of-contents)**

<a name="cae-random"></a>
### 5.5 在客户端和服务端中生成随机数

有时你需要在`GameplayAbility`中生成随机数用于枪支后坐力或者子弹扩散, 客户端和服务端都需要生成相同的随机数, 要做到这一点, 我们必须在`GameplayAbility`激活时设置相同的`随机数种子`, 你需要在每次激活`GameplayAbility`时设置`随机数种子`, 以防客户端错误预测激活或者它的随机数列表与服务端的不同步.  

|随机种子设置方法|描述|
|:-:|:-:|
|使用`Activation Prediction Key`|`GameplayAbility`的`Activation Prediction Key`是一个确保同步并且在客户端和服务端的`Activation()`中都可用的int16类型值. 你可以在客户端和服务端中设置其作为`随机数种子`. 该方法的缺点是每次游戏开始时`Prediction Key`总是从0开始并且会在生成key之后持续增加, 这意味着每场游戏都有着及其相同的随机数序列, 这可能满足或不满足你的随机数需要.|
|激活`GameplayAbility`时通过事件负载(Event Payload)发送种子|通过事件激活`GameplayAbility`并通过可同步的事件负载(Event Payload)从客户端发送随机生成的种子到服务端, 这允许更高的随机性, 但是客户端也容易被破解而每次只发送相同的种子. 通过事件激活`GameplayAbility`也会阻止其从用户绑定激活.|

如果你的随机偏差很小, 大多数玩家是不会注意到每次游戏的随机序列都是相同的, 那么使用`Activation Prediction Key`作为随机种子就应该适用于你. 如果你正在做一些更复杂的事, 需要防范破解者, 那么使用`Server Initiated GameplayAbility`会更好, 服务端可以创建`Prediction Key`或者生成随机数种子来通过事件负载(Event Payload)发送.  

**[⬆ 返回目录](#table-of-contents)**

<a name="cae-crit"></a>
### 5.6 暴击(Critical Hits)

我在伤害[ExecutionCalculation](#concepts-ge-ec)中处理暴击. `GameplayEffect`会有一个像`Effect.CanCrit`样的`GameplayTag`, `ExecutionCalculation`会检查`GameplayEffectSpec`是否有`Effect.CanCrit  GameplayTag`, 如果该`GameplayTag`存在, `ExecutionCalculation`就会生成一个与暴击率相关联的随机数(从Source捕获的`Attribute`), 如果成功的话就会增加暴击伤害(另一个从Source捕获的`Attribute`). 因为我没有预测伤害, 所以不必担心在客户端和服务端上同步随机数生成器的问题(`ExecutionCalculation`只运行在服务端上). 如果你尝试使用`MMC`预测性地执行伤害计算, 就必须从`GameplayEffectSpec->GameplayEffectContext->GameplayAbilityInstance`中获取随机数种子的引用.  

查看GASShooter是如何处理爆头问题的, 其概念是一样的, 除了不再依赖随机数作为概率而是检测`FHitResult`骨骼名.  

**[⬆ 返回目录](#table-of-contents)**

<a name="cae-nonstackingge"></a>
### 5.7 非堆栈GameplayEffect, 但是只有其最高级(Greatest Magnitude)才能实际影响Target

Paragon中的Slow Effect是非堆栈的. 应用每个实例并且像平常一样跟踪其生命周期, 但是只有最高级(Greatest Magnitude)的Slow Effect才能实际影响`Character`. GAS为这种场景提供了开箱即用的`AggregatorEvaluateMetaData`, 详见[AggregatorEvaluateMetaData()](#concepts-as-onattributeaggregatorcreated)及其实现.  

**[⬆ 返回目录](#table-of-contents)**

<a name="cae-paused"></a>
### 5.8 游戏暂停时生成`TargetData`

如果你需要在等待玩家从`WaitTargetData AbilityTask`生成[TargetData](#concepts-targeting-data)时暂停游戏, 我建议使用`slomo 0`而不是暂停.  

**[⬆ 返回目录](#table-of-contents)**

<a name="cae-onebuttoninteractionsystem"></a>
### 5.9 按钮交互系统(Button Interaction System)

GASShooter实现了一个按钮交互系统, 玩家可以按下或按住'E'键来和可交互对象交互, 像复活玩家, 打开武器箱, 打开或关闭滑动门.  

**[⬆ 返回目录](#table-of-contents)**

<a name="debugging"></a>
## 6. 调试GAS

通常在调试GAS相关的问题时, 你感兴趣的事情像:  

> "我的Attribute的值是多少?"
> "我有哪些GameplayTag?"
> "我现在有哪些GameplayEffect?"
> "我已经授予的Ability有哪些, 哪个正在运行, 哪个被堵止激活?"

GAS有两种技术可以在运行时解决这些问题 —— [showdebug abilitysystem](#debugging-sd)和在[GameplayDebugger](#debugging-gd)中Hook.  

**Tip:** UE4倾向于优化C++代码, 这使得某些函数变得很难调试, 当深入追踪代码时很少遇到这种情况. 如果将Visual Studio的解决方案配置设置为`DebugGame Editor`仍然不能追踪代码或者监视变量, 可以使用`PRAGMA_DISABLE_OPTIMIZATION_ACTUAL`和`PRAGMA_ENABLE_OPTIMIZATION_ACTUAL`宏包裹优化函数来关闭优化, 这不能在插件代码中使用除非从源码重新编译插件. 这可以或不可以用于inline函数, 取决于它的作用和位置. 确保完成调试后移除这两个宏!  

```c++
PRAGMA_DISABLE_OPTIMIZATION_ACTUAL
void MyClass::MyFunction(int32 MyIntParameter)
{
	// My code
}
PRAGMA_ENABLE_OPTIMIZATION_ACTUAL
```

**[⬆ 返回目录](#table-of-contents)**

<a name="debugging-sd"></a>
### 6.1 showdebug abilitysystem

在游戏中的控制台输入`showdebug abilitysystem`. 该特性被分为三"页", 三页都会显示当前拥有的`GameplayTag`, 在控制台输入`AbilitySystem.Debug.NextCategory`来换页.  

第一页显示了所有`Attribute`的`CurrentValue`: ![[attachments/82bce806c03018ab6f1e6a8d4aa16f0a_MD5.png]]  

第二页显示了所有应用到你的`持续(Duration)`和`无限(Infinite)GameplayEffect`, 它们的堆栈数, 使用的`GameplayTag`和`Modifier`. ![[attachments/abc8096d382fe7dafc3a7543689bdaab_MD5.png]]  

第三页显示了所有授予到你的`GameplayAbility`, 无论其是否正在运行, 无论其是否被阻止激活, 和当前正在运行的`AbilityTask`的状态.  ![[attachments/2e7de398871c8533a6d950662579d19a_MD5.png]]  

你可以使用`PageUp`和`PageDown`切换Target, 页面只显示你本地控制的`Character`中的`ASC`数据, 然而, 使用`AbilitySystem.Debug.NextTarget`和`AbilitySystem.Debug.PrevTarget`可以显示其他`ASC`的数据, 但是不会显示调试信息的上半部分, 也不会更新绿色目标长方体, 因此无法知道当前定位的是哪个`ASC`, 该BUG已经被提交到[https://issues.unrealengine.com/issue/UE-90437.](https://issues.unrealengine.com/issue/UE-90437).  

**Note:**  为了`showdebug abilitysystem`可以使用, 必须在GameMode中选择一个实际的HUD类, 否则就会找不到该命令并返回"Unknown Command".  

**[⬆ 返回目录](#table-of-contents)**

<a name="debugging-gd"></a>
### 6.2 Gameplay Debugger

GAS向Gameplay Debugger中添加了功能, 使用``反引号(`)``键以访问Gameplay Debugger. 按下小键盘的3键以启用Ability分类, 取决于你所拥有的插件, 分类可能是不同的. 如果你的键盘没有小键盘, 比如笔记本, 那么你可以在项目设置(Project Settings)里修改键盘绑定.  

当你想要查看其他Character的`GameplayTag`, `GameplayEffect`和`GameplayAbility`时可以使用Gameplay Debugger, 可惜的是它不能显示Target的`Attribute`中的`CurrentValue`. 它会定位屏幕中央的任何Character, 你可以通过选择编辑器世界大纲(World Outliner)或者看向另一个不同的Character并再次按下``反引号(`)``键来修改Target. 当前监视的Character上方有最大的红色圆.  

**[⬆ 返回目录](#table-of-contents)**

<a name="debugging-log"></a>
### 6.3 GAS日志(Logging)

GAS包含了大量不同详细级别的日志生成语句, 你很可能见到的是`ABILITY_LOG()`这样的语句. 默认的详细级别是`Display`, 更高的默认不会显示在控制台里.  

为了修改某个日志分类的详细级别, 在控制台中输入:  

```c++
log [category] [verbosity]
```

例如, 为了开启`ABILITY_LOG()`语句, 应该在控制台中输入:  

```c++
log LogAbilitySystem VeryVerbose
```

为了恢复默认, 输入:  

```c++
log LogAbilitySystem Display
```

为了显示所有的日志分类, 输入:  

```c++
log list
```

值得注意的GAS相关的日志分类:  

|日志分类|默认详细级别|
|:-:|:-:|
|LogAbilitySystem|Display|
|LogAbilitySystemComponent|Log|
|LogGameplayCueDetails|Log|
|LogGameplayCueTranslator|Display|
|LogGameplayEffectDetails|Log|
|LogGameplayEffects|Display|
|LogGameplayTags|Log|
|LogGameplayTasks|Log|
|VLogAbilitySystem|Display|

详情查看[日志Wiki](https://www.ue4community.wiki/Legacy/Logs,_Printing_Messages_To_Yourself_During_Runtime).  

**[⬆ 返回目录](#table-of-contents)**

<a name="optimizations"></a>
## 7. 优化

<a name="optimizations-abilitybatching"></a>
## 7.1 Ability批处理

[GameplayAbility](#concepts-ga)在一帧中的的激活、选择性地向服务器发送`TargetData`、结束可以被[批处理而将2-3个RPC压缩成一个RPC](#concepts-ga-batching). 这种RPC常用于枪支的命中判断.  

**[⬆ 返回目录](#table-of-contents)**

<a name="optimizations-gameplaycuebatching"></a>
## 7.2 GameplayCue批处理

如果你需要同时发送很多[GameplayCue](#concepts-gc), 可以考虑将其[批处理成一个RPC](#concepts-gc-batching), 目标就是减少RPC数量(`GameplayCue`是不可靠的网络多播(NetMulticast))并以尽可能少的数据量发送.  

**[⬆ 返回目录](#table-of-contents)**

<a name="optimizations-ascreplicationmode"></a>
## 7.3 AbilitySystemComponent同步模式(Replication Mode)

默认情况下, [ASC](#concepts-asc)处于[Full Replication](#concepts-asc-rm)模式, 这会同步所有的[GameplayEffect](#concepts-ge)到每个客户端(对单人游戏来说很好). 在多人游戏中, 设置玩家拥有的`ASC`为`Mixed Replication`模式, AI控制的Character为`Minimal Replication`模式, 这会将应用到玩家Character上的`GE`仅同步到该Character的拥有者, 应用到AI控制的Character上的`GE`永远不会同步到客户端. 无论是什么同步模式, [GameplayTag](#concepts-gt)仍会进行同步, [GameplayCue](#concepts-gc)仍会以不可靠的网络多播(NetMulticast)到所有客户端. 当所有客户端都不需要查看这些数据时, 这会减少从`GE`同步的网络数据.  

**[⬆ 返回目录](#table-of-contents)**

<a name="optimizations-attributeproxyreplication"></a>
## 7.4 Attribute代理同步

在有很多玩家的大型游戏中, 像Fortnite大逃杀(Fortnite Battle Royale), 有大量存在于常相关`PlayerState`上的[ASC](#concepts-asc), 其会同步大量[Attribute](#concepts-a). 为了优化该瓶颈, Fortnite禁止在`PlayerState::ReplicateSubobjects()`中完全同步`ASC`和它的[AttributeSet](#concepts-as)到模拟玩家控制的代理(Simulated Player-Controlled Proxy)上. Autonomou代理和AI控制的`Pawn`仍然根据其[同步模式](#concepts-asc-rm)来完全同步. Fortnite使用了玩家Pawn上的可同步代理结构体, 而不是同步常相关`PlayerState`上的`ASC`的`Attribute`. 当服务端ASC上的`Attribute`改变时, 其也会在代理结构体中改变, 客户端会从代理结构体接收同步的`Attribute`并将修改返回其本地`ASC`, 这允许`Attribute`同步使用`Pawn`的相关性(Relevancy)和`NetUpdateFrequency`, 该代理结构体也会使用位掩码(Bitmask)同步一个小的`GameplayTag`白名单集合. 该优化减少了网络数据量, 并允许我们利用Pawn相关性(Relevancy). AI控制的`Pawn`的`ASC`位于已经使用其相关性的`Pawn`上, 因此其并不需要该优化.  

> I’m not sure if it is still necessary with other server side optimizations that have been done since then (Replication Graph, etc) and it is not the most maintainable pattern.

来自Epic的Dave Ratti回答[社区问题#3](https://epicgames.ent.box.com/s/m1egifkxv3he3u3xezb9hzbgroxyhx89).  

**[⬆ 返回目录](#table-of-contents)**

<a name="optimizations-asclazyloading"></a>
## 7.5 ASC懒加载

Fortnite大逃杀(Fortnite Battle Royale)世界中有很多可损坏的`AActor`(树木, 建筑物等等), 每个都有一个`ASC`, 这会增加内存消耗, Fortnite通过只在需要时(当其第一次被玩家伤害时)懒加载`ASC`的方式优化该问题, 这会减少整体内存消耗, 因为某些`AActor`在一局比赛中可能永远不会被伤害.  

**[⬆ 返回目录](#table-of-contents)**

<a name="qol"></a>
# 8. Quality of Life Suggestions

<a name="qol-gameplayeffectcontainers"></a>
## 8.1 Gameplay Effect Containers

[GameplayEffectContainer](#concepts-ge-containers)将[GameplayEffectSpec](#concepts-ge-spec), [TargetData](#concepts-targeting-data), 简单定位和其他相关功能整合进简单易用的结构体, 这对转移`GameplayEffectSpec`到Ability生成的抛射物很有用, 该抛射物之后会在碰撞体上应用`GameplayEffectSpec`.  

**[⬆ 返回目录](#table-of-contents)**

<a name="qol-asynctasksascdelegates"></a>
## 8.2 将蓝图AsyncTask绑定到ASC委托

为了增加设计师友好的迭代次数, 特别是在为UI设计UMG Widget时, 可以创建蓝图AsyncTask(在C++中)以直接在UMG蓝图图表中绑定到`ASC`中常见的修改委托, 唯一的告诫就是其必须手动销毁(比如在widget销毁时), 否则就会永远存于内存中.样例项目包含三个蓝图AsyncTask.  

监听`Attribute`修改:  

![[attachments/7615f2cbcb71b1ec17fe13f200aa0a7c_MD5.png]]  

监听冷却时间修改:  

![[attachments/e8aa31b6963a4142023590947b5ff46b_MD5.png]]  

监听`GE`堆栈修改:  

![[attachments/d0b092d7380721504bc8c8851c258512_MD5.png]]  

**[⬆ 返回目录](#table-of-contents)**

<a name="troubleshooting"></a>
# 9. 疑难解答

<a name="troubleshooting-notlocal"></a>
## 9.1 LogAbilitySystem: Warning: Can't activate LocalOnly or LocalPredicted ability %s when not local!

你需要在[客户端初始化`ASC`](#concepts-asc-setup).  

**[⬆ 返回目录](#table-of-contents)**

<a name="troubleshooting-scriptstructcache"></a>
## 9.2 `ScriptStructCache`错误

你需要调用[UAbilitySystemGlobals::InitGlobalData()](#concepts-asg-initglobaldata).  

**[⬆ 返回目录](#table-of-contents)**

<a name="troubleshooting-replicatinganimmontages"></a>
## 9.3 动画蒙太奇不能同步到客户端

确保在[GameplayAbility](#concepts-ga)中正在使用的是`PlayMontageAndWait`节点而不是`PlayMontage`, 该[AbilityTask](#concepts-at)可以通过`ASC`自动同步蒙太奇而`PlayMontage`节点不可以.  

**[⬆ 返回目录](#table-of-contents)**

<a name="troubleshooting-duplicatingblueprintactors"></a>
## 9.4 复制的蓝图Actor会将AttributeSet设置为nullptr

这是一个[虚幻引擎的bug](https://issues.unrealengine.com/issue/UE-81109), 当使用从一个存在的蓝图Actor类复制的方式来创建新的类, 这会让这个类中将AttributeSet指针设置为空指针.  

对此有一些变通的方法, 我已经成功地不在我的类内创建定制的`AttributeSet`指针(头文件中没有指针, 也不在构造函数中调用`CreateDefaultSubobject`), 
而是直接在PostInitializeComponents()中向`ASC`添加`AttributeSets`(样本项目中没有显示).  
复制的AttributeSets将仍然存在于`ASC`的`SpawnedAttributes`数组中. 它看起来像这样:  

```c++
void AGDPlayerState::PostInitializeComponents()
{
	Super::PostInitializeComponents();

	if (AbilitySystemComponent)
	{
		AbilitySystemComponent->AddSet<UGDAttributeSetBase>();
		// ... 其他你可能拥有的属性集
	}
}
```

在这种情况下, 你想要读取或者修改`AttributeSet`的值, 就需要调用`ASC`实例中的函数, 而不是[`AttributeSet`中定义的宏](#concepts-as-attributes).  


```c++
/** 返回当前(最终)属性值 */
float GetNumericAttribute(const FGameplayAttribute &Attribute) const;

/** 设置一个属性的基础值。 当前激活的修改器不会被清除并将在NewBaseValue上发挥作用 */
void SetNumericAttributeBase(const FGameplayAttribute &Attribute, float NewBaseValue);
```

所以GetHealth()的实现将会如下:  

```c++
float AGDPlayerState::GetHealth() const
{
	if (AbilitySystemComponent)
	{
		return AbilitySystemComponent->GetNumericAttribute(UGDAttributeSetBase::GetHealthAttribute());
	}

	return 0.0f;
}
```

设置(初始化)生命值属性的实现将会是这样:  

```c++
const float NewHealth = 100.0f;
if (AbilitySystemComponent)
{
	AbilitySystemComponent->SetNumericAttributeBase(UGDAttributeSetBase::GetHealthAttribute(), NewHealth);
}
```

顺便提一下, 往`ASC`组件注册的每个`AttributeSet`类最多只有一个对象.  

**[⬆ Back to Top](#table-of-contents)**

<a name="acronyms"></a>
# 10. ASC常用术语缩略

|术语|缩略|
|:-:|:-:|
|AbilitySystemComponent|ASC|
|AbilityTask|AT|
|Action RPG Sample Project by Epic|ARPG, ARPG Sample|
|CharacterMovementComponent|CMC|
|GameplayAbility|GA|
|GameplayAbilitySystem|GAS|
|GameplayCue|GC|
|GameplayEffect|GE|
|GameplayEffectExecutionCalculation|ExecCalc, Execution|
|GameplayTag|Tag, GT|
|ModiferMagnitudeCalculation|ModMagCalc, MMC|

**[⬆ 返回目录](#table-of-contents)**

<a name="resources"></a>
# 11. 其他资源

[官方文档](https://docs.unrealengine.com/en-US/InteractiveExperiences/GameplayAbilitySystem/index.html)  

源代码! 特别是`GameplayPrediction.h`.  

[Epic的Action RPG样例项目](https://www.unrealengine.com/marketplace/en-US/slug/action-rpg)  

[来自Epic的Dave Ratti回复社区关于GAS的问题](https://epicgames.ent.box.com/s/m1egifkxv3he3u3xezb9hzbgroxyhx89)  

[Unreal Slackers Discord](https://unrealslackers.org)有一个专注于GAS`#gameplay-abilities-plugin`的文字频道  

[Dan 'Pan'的Github库](https://github.com/Pantong51/GASContent)  

[SabreDartStudios的YouTube视频](https://www.youtube.com/channel/UCCFUhQ6xQyjXDZ_d6X_H_-A)

**[⬆ 返回目录](#table-of-contents)**

<a name="changelog"></a>
# 12. GAS更新日志

这是从Unreal Engine官方升级日志和我遇到的且未记录的升级中整理的一份值得一看的升级列表, 如果你发现某些没有列在其中, 请提issue或者PR.  

**[⬆ 返回目录](#table-of-contents)**

<a name="changelog-4.26"></a>
## 4.26

* GAS plugin is no longer flagged as beta.
* Crash Fix: Fixed a crash when adding a gameplay tag without a valid tag source selection.
* Crash Fix: Added the path string arg to a message to fix a crash in UGameplayCueManager::VerifyNotifyAssetIsInValidPath.
* Crash Fix: Fixed an access violation crash in AbilitySystemComponent_Abilities when using a ptr without checking it.
* Bug Fix: Fixed a bug where stacking GEs that did not reset the duration on additional instances of the effect being applied.
* Bug Fix: Fixed an issue that caused CancelAllAbilities to only cancel non-instanced abilities.
* New: Added optional tag parameters to gameplay ability commit functions.
* New: Added StartTimeSeconds to PlayMontageAndWait ability task and improved comments.
* New: Added tag container "DynamicAbilityTags" to FGameplayAbilitySpec. These are optional ability tags that are replicated with the spec. They are also captured as source tags by applied gameplay effects.
* New: GameplayAbility IsLocallyControlled and HasAuthority functions are now callable from Blueprint.
* New: Visual logger will now only collect and store info about instant GEs if we're currently recording visual logging data.
* New: Added support for redirectors on gameplay attribute pins in blueprint nodes.
* New: Added new functionality for when root motion movement related ability tasks end they will return the movement component's movement mode to the movement mode it was in before the task started.

**[⬆ 返回目录](#table-of-contents)**

<a name="changelog-4.25.1"></a>
## 4.25.1

* Fixed! UE-92787 Crash saving blueprint with a Get Float Attribute node and the attribute pin is set inline
* Fixed! UE-92810 Crash spawning actor with instance editable gameplay tag property that was changed inline

**[⬆ 返回目录](#table-of-contents)**

<a name="changelog-4.25"></a>
## 4.25

* Fixed prediction of RootMotionSource AbilityTasks
* [GAMEPLAYATTRIBUTE_REPNOTIFY()](#concepts-as-attributes) now additionally takes in the old Attribute value. We must supply that as the optional parameter to our OnRep functions. Previously, it was reading the attribute value to try to get the old value. However, if called from a replication function, the old value had already been discarded before reaching SetBaseAttributeValueFromReplication so we'd get the new value instead.
* Added [NetSecurityPolicy](#concepts-ga-netsecuritypolicy) to UGameplayAbility.
* Crash Fix: Fixed a crash when adding a gameplay tag without a valid tag source selection.
* Crash Fix: Removed a few ways for attackers to crash a server through the ability system.
* Crash Fix: We now make sure we have a GamplayEffect definition before checking tag requirements.
* Bug Fix: Fixed an issue with gameplay tag categories not applying to function parameters in Blueprints if they were part of a function terminator node.
* Bug Fix: Fixed an issue with gameplay effects' tags not being replicated with multiple viewports.
* Bug Fix: Fixed a bug where a gameplay ability spec could be invalidated by the InternalTryActivateAbility function while looping through triggered abilities.
* Bug Fix: Changed how we handle updating gameplay tags inside of tag count containers. When deferring the update of parent tags while removing gameplay tags, we will now call the change-related delegates after the parent tags have updated. This ensures that the tag table is in a consistent state when the delegates broadcast.
* Bug Fix: We now make a copy of the spawned target actor array before iterating over it inside when confirming targets because some callbacks may modify the array.
* Bug Fix: Fixed a bug where stacking GamplayEffects that did not reset the duration on additional instances of the effect being applied and with set by caller durations would only have the duration correctly set for the first instance on the stack. All other GE specs in the stack would have a duration of 1 second. Added automation tests to detect this case.
* Bug Fix: Fixed a bug that could occur if handling gameplay event delegates modified the list of gameplay event delegates.
* Bug Fix: Fixed a bug causing GiveAbilityAndActivateOnce to behave inconsistently.
* Bug Fix: Reordered some operations inside FGameplayEffectSpec::Initialize to deal with a potential ordering dependency.
* New: UGameplayAbility now has an OnRemoveAbility function. It follows the same pattern as OnGiveAbility and is only called on the primary instance of the ability or the class default object.
* New: When displaying blocked ability tags, the debug text now includes the total number of blocked tags.
* New: Renamed UAbilitySystemComponent::InternalServerTryActiveAbility to UAbilitySystemComponent::InternalServerTryActivateAbility.Code that was calling InternalServerTryActiveAbility should now call InternalServerTryActivateAbility.
* New: Continue to use the filter text for displaying gameplay tags when a tag is added or deleted. The previous behaviour cleared the filter.
* New: Don't reset the tag source when we add a new tag in the editor.
* New: Added the ability to query an ability system component for all active gameplay effects that have a specified set of tags. The new function is called GetActiveEffectsWithAllTags and can be accessed through code or blueprints.
* New: When root motion movement related ability tasks end they now return the movement component's movement mode to the movement mode it was in before the task started.
* New: Made SpawnedAttributes transient so it won't save data that can become stale and incorrect. Added null checks to prevent any currently saved stale data from propagating. This prevents problems related to bad data getting stored in SpawnedAttributes.
* API Change: AddDefaultSubobjectSet has been deprecated. AddAttributeSetSubobject should be used instead.
* New: Gameplay Abilities can now specify the Anim Instance on which to play a montage.

**[⬆ 返回目录](#table-of-contents)**

<a name="changelog-4.24"></a>
## 4.24

* Fixed blueprint node Attribute variables resetting to None on compile.
* Need to call [UAbilitySystemGlobals::InitGlobalData()](#concepts-asg-initglobaldata) to use [TargetData](#concepts-targeting-data) otherwise you will get ScriptStructCache errors and clients will be disconnected from the server. My advice is to always call this in every project now whereas before 4.24 it was optional.
* Fixed crash when copying a GameplayTag setter to a blueprint that didn't have the variable previously defined.
* UGameplayAbility::MontageStop() function now properly uses the OverrideBlendOutTime parameter.
* Fixed GameplayTag query variables on components not being modified when edited.
* Added the ability for GameplayEffectExecutionCalculations to support scoped modifiers against "temporary variables" that aren't required to be backed by an attribute capture.
	+ Implementation basically enables GameplayTag-identified aggregators to be created as a means for an execution to expose a temporary value to be manipulated with scoped modifiers; you can now build formulas that want manipulatable values that don't need to be captured from a source or target.
	+ To use, an execution has to add a tag to the new member variable ValidTransientAggregatorIdentifiers; those tags will show up in the calculation modifier array of scoped mods at the bottom, marked as temporary variables—with updated details customizations accordingly to support feature
* Added restricted tag quality-of-life improvements. Removed the default option for restricted GameplayTag source. We no longer reset the source when adding restricted tags to make it easier to add several in a row.
* APawn::PossessedBy() now sets the owner of the Pawn to the new Controller. Useful because Mixed Replication Mode expects the owner of the Pawn to be the Controller if the ASC lives on the Pawn.
* Fixed bug with POD (Plain Old Data) in FAttributeSetInittterDiscreteLevels.

**[⬆ 返回目录](#table-of-contents)**
# GAS-
