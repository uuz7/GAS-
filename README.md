
> [!NOTE] 前言
> 学完aura看完Lyra大部分,GAS源码也看了不少,打算做一个文档当复习总结,计划是对已有文档进行AI排版优化跟上UE5再写一些见解,原文档: [GASDocumentation_Chinese](https://github.com/BillEliot/GASDocumentation_Chinese)

> 引擎版本5.4+
> 状态: 随缘更新

## GameplayAbilitySystem插件文档uuz7版

> GameplayAbilitySystem: 能力系统插件,简称GAS

|                 术语                 | 中文                                                                 |     缩略(怎么顺怎么来)      |
| :--------------------------------: | ------------------------------------------------------------------ | :-----------------: |
|       GameplayAbilitySystem        | 能力系统,插件名                                                           |         GAS         |
|            AttributeSet            | 属性集,管理属性                                                           |         AS          |
|           GameplayEffect           | 游戏效果,修改属性值                                                         |         GE          |
|         GameplayEffectSpec         | 游戏效果规格, GameplayEffect运行时实例                                        |       GESpec        |
|       GameplayEffectContext        | 游戏效果上下文, 记录游戏效果运行过程的一些数据,如谁发起的,作用谁,命中点位置等数据                        |      GEContext      |
|          GameplayAbility           | 能力                                                                 |         GA          |
|        GameplayAbilitySpec         | 能力规格, GameplayAbility运行时实例                                         |       GASpec        |
|            AbilityTask             | 能力任务, 能力的一个单元逻辑                                                    |     AbilityTask     |
|            GameplayCue             | 游戏提示,声音和粒子特效                                                       |     GameplayCue     |
|       AbilitySystemComponent       | 能力系统组件,主要属性:可激活能力、应用在自己的游戏效果,自身拥有的标签.主要功能: 作为中央组件与能力、游戏效果、游戏提示进行交互 |         ASC         |
| GameplayEffectExecutionCalculation | 执行                                                                 | ExecCalc, Execution |
|    ModiferMagnitudeCalculation     | 修改器幅度计算                                                            |   ModMagCalc, MMC   |
>Spec: 比如GESpec,  GESpec存储GE并多了些运行时数据,所以说是运行时实例
>Handle:也曾句柄,比如GameplayEffectContextHandle,一是标识数据,二是存储数据指针,三还可能是用于网络传输,我一直把句柄理解为勺子的一端,简单拿起勺子的一端就能获得更多

该插件对于单人和多人游戏提供了开箱即用的解决方案:

 执行基于等级的角色能力(Ability), 该能力可选花费和冷却时间.
 管理属于Actor的数值Attribute. (Attribute)
 为Actor应用状态效果. (GameplayEffect)
 为Actor应用GameplayTag. (GameplayTag)
 生成视觉或声音效果. (GameplayCue)
 为以上提到的所有应用同步(Replication).

在多人游戏中, GAS提供客户端预测(client-side prediction)支持:  

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

### Ability System Component


> 能力系统组件,简称ASC

`AbilitySystemComponent (ASC) 是 GAS 的核心组件，继承自 UActorComponent
	所有需要使用 GameplayAbility、拥有 Attribute、或接收 GameplayEffect 的 Actor，都必须挂载一个 ASC。它负责管理和同步这些对象（Attribute` 的同步除外，由 AttributeSet 处理）。通常建议开发者继承它进行自定义扩展。

ASC 关联两个核心对象：

- **OwnerActor**：ASC 的归属对象，一般用于持久化数据。

>   Owner:所属者或者说所有者,是逻辑上的归属关系.网络下，Owner 会影响哪些 Actor 复制到哪些客户端
>   Outer:UObject的生命周期与Outer同步,比如Actor在构造函数里创建组件也就是子对象时,Outer就为this

- **AvatarActor**：ASC 的物理表现对象，通常是实际出现在场景里的 Pawn/Character。

>   代理Actor.ASC属于OwnerActor所以**AvatarActor**销毁时不影响ASC中的数据

两者可以是同一个 Actor（如 MOBA 中的小兵），也可以不同（如 MOBA 英雄：`OwnerActor=PlayerState`，`AvatarActor=Character`）。  
如果角色需要 **重生并保留 Attribute/GameplayEffect**（例如英雄角色），推荐将 ASC 放在 `PlayerState` 上。

> ⚠️ 注意：若 ASC 放在 `PlayerState`，需要提高其 **NetUpdateFrequency**（默认很低），否则客户端上的 Attribute 或 GameplayTag 更新会有延迟。确保启用 Adaptive Network Update Frequency，Fortnite 就是这样做的。

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



<a name="concepts-gt"></a>
### AttributeSet

> 简称AS

#### 创建AttributeSet

`AttributeSet`用于管理`Attribute`. 开发者应该继承[UAttributeSet](https://docs.unrealengine.com/en-US/API/Plugins/GameplayAbilities/UAttributeSet/index.html),ASC拥有的属性
- 在OwnerActor的构造函数中创建`AttributeSet`会自动注册到其`ASC`. **这必须在C++中完成.**  如OwnerActor是PlayerState:
![[attachments/Pasted image 20250818184519.png]]



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



<a name="concepts-as-onattributeaggregatorcreated"></a>
#### OnAttributeAggregatorCreated()

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



<a name="concepts-ge"></a>
### Attribute

#### Attribute定义
`Attribute` 由 `FGameplayAttributeData` 结构体定义，表示一个浮点值，可用于任何游戏相关数值，例如角色生命值、等级或药水剂量。**如果某个数值属于 Actor 且与游戏逻辑相关，就应考虑使用 `Attribute`。**

`Attribute` 应只通过 [GameplayEffect](#concepts-ge) 修改，以便 `ASC` 能够进行 [预测(Predict)](#concepts-p) 改变。

`Attribute` 一般由 `AttributeSet` 定义并存储，`AttributeSet` 会同步标记为 replication 的 `Attribute`。具体的定义方式可参阅 [AttributeSet](#concepts-as) 部分。

**Tip**：如果不希望某个 `Attribute` 在编辑器的 Attribute 列表中显示，可添加 `Meta = (HideInDetailsView)` 属性宏。



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



<a name="concepts-a-changes"></a>
#### 响应Attribute变化

要监听 `Attribute` 的变化，以便更新 UI 或处理其他游戏逻辑，可以使用：

`UAbilitySystemComponent::GetGameplayAttributeValueChangeDelegate()`

该函数返回一个委托（Delegate），可绑定在 `Attribute` 变化时自动调用的函数。委托回调提供一个 `FOnAttributeChangeData` 参数，其中包含：

- `NewValue`：变化后的值
  
- `OldValue`：变化前的值
  
- `FGameplayEffectModCallbackData`：相关 GameplayEffect 回调数据（**注意**：此字段仅能在服务端使用）
  

也可以通过能力任务类`AbilityTask_WaitAttributeChange`(蓝图中为WaitAttributeChange)在能力激活时进行监听


<a name="concepts-a-derived"></a>
#### 自动更新Attribute值

如果希望一个属性自动基于其他属性更新,可以给ASC应用一个永久GE,修改器使用**基于 Attribute 或 [MMC](#concepts-ge-mmc) **。
如图是MaxMana(最大法力值)等于智力的两倍
![[attachments/Pasted image 20250818183930.png]]

> **注意**：在 PIE 中打开多个窗口时，需要在编辑器首选项中禁用 `Run Under One Process`，否则除了第一个窗口外，基于属性计算的属性的更新将无效




### GameplayEffect
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





<a name="concepts-ge-ga"></a>
#### GameplayEffect组件
>编辑器有详细的提示



<a name="concepts-ge-spec"></a>
##### 授予Ability

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



<a name="concepts-ge-tags"></a>
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


#### 自定义计算
##### ModifierMagnitudeCalculation

> 修改器幅度计算,简称MMC

ModifierMagnitudeCalculation是一种可以在 `GameplayEffect` 的 Modifier中使用的计算类。它的功能类似于 GameplayEffectExecutionCalculation，但更**简单、可预测**：  
其唯一职责就是在 `CalculateBaseMagnitude_Implementation()` 中返回一个浮点数。该函数可以通过 **C++** 或 **蓝图** 继承并重写。

###### 适用范围

MMC 可用于所有类型的 `GameplayEffect`：

- 即刻 (Instant)
  
- 持续 (Duration)
  
- 无限 (Infinite)
  
- 周期性 (Periodic)
  

###### 特点与优势

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

###### 结果修正

MMC 返回的浮点值，最终还会受到 `GameplayEffect` Modifier 中的 **系数与前后修正因子** 影响。

---

###### 示例：基于目标魔法值的中毒效果

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


####### 使用要点

- 若尝试捕获 Attribute 却未在构造函数中加入 `RelevantAttributesToCapture`，会报缺失错误。
  
- 如果 MMC 只依赖 Tag 或 SetByCaller 值（无需捕获 Attribute），则无需填充 `RelevantAttributesToCapture`。



<a name="concepts-ge-ec"></a>
##### GameplayEffectExecutionCalculation (ExecCalc)
>消息效果执行计算,也就是执行


GameplayEffectExecutionCalculation（简称 **ExecutionCalculation**、**Execution** 或 **ExecCalc**）是 `GameplayEffect` 修改 `属性` 最强大的方式。

与 ModifierMagnitudeCalculation (MMC)类似，它也可以捕获 `Attribute` 并选择性地创建 **快照 (Snapshot)**。但不同的是：

- **MMC** 只能返回一个浮点值，通常用于修改单一 `Attribute`。
  
- **ExecCalc** 可以一次性修改 **多个 Attribute**，并能够执行任意复杂逻辑。
  

正因为这种强大与灵活，**ExecCalc 不具备可预测性 (Non-Predictable)**，并且只能在 **C++** 中实现。

---

###### 使用范围

ExecCalc 只能由以下类型的 `GameplayEffect` 使用：

- 即刻 (Instant)
  
- 周期性 (Periodic)
  

插件代码中凡是涉及 “Execute” 的，一般都指这两种 GE 类型。

---

###### Attribute 捕获机制

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

###### Attribute 捕获设置方式
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

###### 网络调用规则

对于以下几种 **GameplayAbility**：

- Local Predicted
  
- Server Only
  
- Server Initiated
  

`ExecCalc` **只会在服务端调用**。

---

###### 应用场景

ExecCalc 最常见的用途是实现一个复杂的数值公式：

- 从 **Source** 和 **Target** 捕获多个 Attribute（如攻击力、防御、暴击率等）。
  
- 从 **Spec 的 SetByCaller** 中读取额外参数（如技能配置的伤害）。
  
- 综合计算最终的结果，再修改目标的多个 Attribute。
  

例如 ActionRPG 示例项目中的 **GDDamageExecCalculation**：

- 从 `Spec.SetByCaller` 中读取伤害值。
  
- 根据目标的护盾 Attribute 进行伤害削减。



<a name="concepts-ge-ec-senddata"></a>
###### 发送数据到Execution Calculation

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




<a name="concepts-ge-car"></a>
#### 自定义应用需求组件

`CustomApplicationRequirement(CAR)`类为设计师提供对于`GameplayEffect`是否可以应用的高阶控制, 而不是对`GameplayEffect`进行简单的`GameplayTag`检查. 这可以通过在蓝图中重写`CanApplyGameplayEffect()`和在C++中重写`CanApplyGameplayEffect_Implementation()`实现.  

`CAR`的应用场景:  

* 目标需要有一定数量的`Attribute`.
* 目标需要有一定数量的`GameplayEffect`堆栈.

`CAR`还有很多高阶功能, 像检查`GameplayEffect`实例是否已经位于目标上, 修改当前实例的[持续时间](#concepts-ge-duration)而不是应用一个新实例(对于`CanApplyGameplayEffect()`返回false).



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



<a name="concepts-ge-duration"></a>
#### 修改已激活GameplayEffect的持续时间
要修改 `Cooldown GE` 或其他 **持续型 (Duration)** `GameplayEffect` 的剩余时间，需要在服务端操作 `GameplayEffectSpec`：

- 更新 `Duration`、`StartServerWorldTime`、`CachedStartServerWorldTime`、`StartWorldTime`
    
- 调用 `CheckDuration()` 重新计算持续时间
    
- 将对应的 `FActiveGameplayEffect` 标记为 _dirty_，以便修改同步到客户端
    

> ⚠️ 注意：这通常需要使用 `const_cast`，可能并非 Epic 期望的官方做法，但目前验证可行且稳定。

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



<a name="concepts-ge-dynamic"></a>
#### 运行时创建动态`GameplayEffect`

- **限制**  
    只有 **即时型 (Instant)** `GameplayEffect` 可以在运行时通过 C++ 创建。  
    **持续型 (Duration)** 和 **无限型 (Infinite)** `GameplayEffect` 不能动态创建，因为它们在同步时需要引用一个实际存在的 `GameplayEffect` 类定义。
    
- **推荐做法**  
    为了支持运行时的定制，应先创建一个 **原型 GameplayEffect 类**（就像在编辑器中常规设置的一样），然后在运行时基于该原型生成并调整 `GameplayEffectSpec`。
    
- **客户端预测**  
    运行时创建的 **即时型 (Instant) GameplayEffect** 可以安全地在客户端预测的 `GameplayAbility` 中使用。但目前尚不确定这种动态创建是否会带来潜在副作用。
    
- **示例**  
    在样例项目中，当角色的 `AttributeSet` 检测到致命一击时，会在运行时创建一个 `GameplayEffect`，用于把金币和经验奖励返还给击杀者。

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



<a name="concepts-ge-containers"></a>
#### GameplayEffect Containers

Epic 的 [Action RPG](https://www.unrealengine.com/marketplace/en-US/slug/action-rpg) 示例项目实现了一个 **`FGameplayEffectContainer`** 结构体。它不是 GAS 原生的一部分，但在组织 `GameplayEffect` 和 [TargetData](https://chatgpt.com/c/68a5699f-ced4-8327-b6ae-2917b992061a#concepts-targeting-data) 时非常实用：

- 能简化流程，例如：
    
    - 从 `GameplayEffect` 自动生成 `GameplayEffectSpec`
        
    - 在 `GameplayEffectContext` 中自动设置默认值
        
- 在 `GameplayAbility` 中使用它非常直观，可以轻松地把 `GameplayEffectContainer` 传递给生成的投射物。
    

不过在示例文档中我没有采用该结构体，因为想展示 **原生 GAS 的工作方式**。但我仍然强烈建议你研究并在项目中引入它。

- **访问 GESpec**  
    如果需要修改（如添加 `SetByCaller`），可以通过 `FGameplayEffectContainer` 内部的 `GESpec` 数组索引来获取对应的 `GESpec` 引用。缺点是你必须事先知道要访问的索引。
    
- **获取目标支持**  
    `GameplayEffectContainer` 还包含一个可选的高效 **Targeting** 机制，用于定位目标。




<a name="concepts-ga"></a>
### GameplayAbility

#### GameplayAbility定义
>简称GA

`GameplayAbility` 代表 Actor 在游戏中可以触发的一切行为或技能。

- 可以同时激活多个，例如 **奔跑** 与 **射击**。
    
- 可通过 **蓝图** 或 **C++** 实现。
    

**示例：**

- 跳跃
    
- 奔跑
    
- 射击
    
- 每 X 秒被动格挡一次攻击
    
- 使用药剂
    
- 开门
    
- 收集资源
    
- 建造
    

**不建议使用 `GameplayAbility` 的情况：**

- 基础移动输入（如 WASD 移动）
    
- 与 UI 的纯交互（例如商店购买）
    

> ⚠️ 这不是硬性规定，而是常见的设计建议。

---

#### 内置功能

- 能根据 **等级** 自动调整 Attribute 的变化量。
    >意思是GE的等级默认等于GA的等级
- 内置对 **Cost** 和 **Cooldown GameplayEffect** 的支持。
    
- 支持通过 [AbilityTask](#concepts-at) 处理 **随时间推移** 的逻辑：
    
    - 等待事件
        
    - 等待 Attribute 变化
        
    - 等待目标选择
        
    - 使用 Root Motion Source 控制角色移动
        

---

#### 网络执行与同步

`GameplayAbility` 的执行位置取决于 **网络执行策略 (Net Execution Policy)**，而不是是否为 Simulated Proxy：

- 决定 Ability 是否可被客户端 **预测 (Predicted)**
    
- 定义了 **Cost / Cooldown GE** 的默认行为
    

> **重要：** Simulated Client 不会直接运行 `GameplayAbility`。

- 服务端执行 Ability 后，相关表现会同步给模拟代理：
    
    - 动画蒙太奇 → **Replicate**
        
    - 事件逻辑 → **AbilityTask RPC**
        
    - 声音/粒子等装饰效果 → **GameplayCue**

技能激活时的逻辑写在`ActivateAbility()`函数, 结束时的逻辑写在`EndAbility()`

一个简单的`GameplayAbility`流程图: ![[attachments/10a9b92a92f597843a5ecaeb47b13c25_MD5.png]]  

一个更复杂`GameplayAbility`流程图: ![[attachments/7c6a74d820c4d0c583493314e655a78d_MD5.png]]  

复杂的Ability可以使用多个相互交互(激活, 取消等等)的`GameplayAbility`实现.  



<a name="concepts-ga-definition-reppolicy"></a>
##### Replication Policy
>复制策略

不要使用这个选项。它的命名具有误导性，而且你并不需要它。

`GameplayAbilitySpec` 默认会从 **服务端** 同步到 **所属 (Owning) 客户端**。正如前文所述，**`GameplayAbility` 不会在 Simulated Proxy 上运行**。针对 Simulated Proxy 的视觉同步，会通过以下方式实现：

- **AbilityTask / RPC** → 同步事件或逻辑
    
- **GameplayCue** → 播放声音、粒子等装饰效果
    

此外，Epic 的 Dave Ratti 已明确表示，未来将移除该选项。



<a name="concepts-ga-definition-remotecancel"></a>
##### Server Respects Remote Ability Cancellation
>服务器考虑远程能力取消

- **作用**  
    当客户端的 `GameplayAbility` 因玩家取消或自然完成时，该选项会强制结束服务端的同一 Ability，不论服务端是否真的执行完毕。
    
- **问题**  
    这会带来严重隐患，尤其是在 **高延迟** 场景下：
    
    - 客户端预测的 Ability 可能提前结束
        
    - 服务端逻辑被强制中断，导致状态不同步或效果异常
        
- **建议**  
    一般情况下应禁用该选项，以避免不必要的同步问题。



<a name="concepts-ga-definition-repinputdirectly"></a>
##### Replicate Input Directly
>直接复制输入

⚠️ 不建议使用该选项

启用此选项后，客户端会 **持续向服务端同步输入事件**（按下 Press / 抬起 Release）。

- **问题**  
    这种做法会导致不必要的网络流量和潜在的同步问题。
    
- **推荐替代方案**  
    Epic 建议不要依赖该选项，而是使用输入相关的 [AbilityTask](#concepts-at)，并通过 **`Generic Replicated Event`** 机制处理输入（前提是你的输入已绑定在 ASC 上）。



<a name="concepts-ga-input"></a>
#### 绑定输入到ASC
> 这个用法没试过,感觉缺少灵活性,更喜欢lyra中的方法

`ASC` 支持直接绑定输入事件，并在 **授予 `GameplayAbility` 时** 将这些输入分配给 Ability。

- 当 **输入的 GameplayTag 符合要求** 时，按下按键即可激活对应的 `GameplayAbility`。
    
- 分配的输入事件必须使用 **内建的、支持输入的 AbilityTask** 来处理。
    

除了普通输入，还可以绑定两个特殊输入：

- **Confirm** → 例如用于确认 Target Actor
    
- **Cancel** → 例如用于取消 Target Actor  
    这些输入事件会被 `AbilityTask` 用于交互逻辑。
    

---

**输入绑定步骤**

1. **定义枚举**
    
    - 需要一个 `UENUM` 来将输入事件映射为 `uint8`，枚举项的名称必须与 **项目设置 (Project Settings)** 中的输入事件名称完全一致（`DisplayName` 无所谓）。
        
    
    示例：
    
    ```c++
    UENUM(BlueprintType)
    enum class EGDAbilityInputID : uint8
    {
        None    UMETA(DisplayName = "None"),
        Confirm UMETA(DisplayName = "Confirm"),
        Cancel  UMETA(DisplayName = "Cancel"),
        Ability1 UMETA(DisplayName = "Ability1"), // LMB
        Ability2 UMETA(DisplayName = "Ability2"), // RMB
        Ability3 UMETA(DisplayName = "Ability3"), // Q
        Ability4 UMETA(DisplayName = "Ability4"), // E
        Ability5 UMETA(DisplayName = "Ability5"), // R
        Sprint   UMETA(DisplayName = "Sprint"),
        Jump     UMETA(DisplayName = "Jump")
    };
    ```
    
    > **Note:** 样例项目中的 `Confirm` / `Cancel` 特殊情况，它们没有和输入设置一一对应（分别对应 `ConfirmTarget`、`CancelTarget`），但在绑定时通过 `BindAbilityActivationToInputComponent()` 进行了映射，因此允许不一致。其他输入必须严格匹配。
    
2. **绑定到 ASC**
    
    - 如果 `ASC` 在 `Character` 上，可以在 `SetupPlayerInputComponent()` 绑定：
        
    ```c++
    AbilitySystemComponent->BindAbilityActivationToInputComponent(
        PlayerInputComponent,
        FGameplayAbilityInputBinds(
            FString("ConfirmTarget"),
            FString("CancelTarget"),
            FString("EGDAbilityInputID"),
            static_cast<int32>(EGDAbilityInputID::Confirm),
            static_cast<int32>(EGDAbilityInputID::Cancel))
    );
    ```
    
    - 如果 `ASC` 在 `PlayerState` 上，则存在潜在的同步竞争：
        
        - `SetupPlayerInputComponent()` 可能执行时 `PlayerState` 尚未同步到客户端。
            
        - 仅依赖 `OnRep_PlayerState()` 也不够，因为 `InputComponent` 可能为 `NULL`。
            
        - **解决方案：** 在 `SetupPlayerInputComponent()` 和 `OnRep_PlayerState()` 中都尝试绑定，并用布尔值控制，确保只绑定一次。
            

---

**建议的设计习惯**

- 对于 **固定输入槽位** 的 Ability（如 MOBA 技能栏），可以在 `UGameplayAbility` 子类中添加一个变量定义它们的输入槽位。
    
- 在授予 Ability 时，从该 Ability 的 **Class Default Object (CDO)** 读取该变量，自动完成输入绑定。
    



<a name="concepts-ga-input-noactivate"></a>
##### 绑定输入时不激活Ability

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



<a name="concepts-ga-granting"></a>
#### 授予Ability


向 `ASC` 授予 `GameplayAbility` 会将其加入 **ActivatableAbilities** 列表，从而允许在满足GameplayTag 条件时激活该 Ability。

**网络同步**

- **服务端授予**：Ability 会自动同步对应的 **GameplayAbilitySpec** 到所属 (Owning) 客户端。
    
- **其他客户端 / Simulated Proxy**：不会接收到该 AbilitySpec。
    

---

**样例项目做法**

在游戏开始时，角色类保存了一个 `TArray<TSubclassOf<UGDGameplayAbility>>`，用于授予角色的初始 Ability：

```c++
void AGDCharacterBase::AddCharacterAbilities()
{
    // 仅在服务端授予
    if (Role != ROLE_Authority || !AbilitySystemComponent.IsValid() || AbilitySystemComponent->CharacterAbilitiesGiven)
    {
        return;
    }

    for (TSubclassOf<UGDGameplayAbility>& StartupAbility : CharacterAbilities)
    {
        AbilitySystemComponent->GiveAbility(
            FGameplayAbilitySpec(
                StartupAbility,
                GetAbilityLevel(StartupAbility.GetDefaultObject()->AbilityID),
                static_cast<int32>(StartupAbility.GetDefaultObject()->AbilityInputID),
                this
            )
        );
    }

    AbilitySystemComponent->CharacterAbilitiesGiven = true;
}
```

**说明**

- 授予 Ability 时，创建 **GameplayAbilitySpec** 需要的信息包括：
    
    - `UGameplayAbility` 类
        
    - Ability 等级
        
    - 输入绑定
        
    - `SourceObject` 或 ASC 的源对象
        
- 这些信息会被封装进 **GameplayAbilitySpec** 并添加到 ASC，从而使 Ability 可以被激活。
    



<a name="concepts-ga-activating"></a>
我帮你把这段文字整理优化，分层清晰，保留所有关键细节，同时便于快速查阅：

---

#### 激活 GameplayAbility

如果某个 `GameplayAbility` 已分配给输入事件，当按键按下且其 `GameplayTag` 条件满足时，它会自动激活。不过，这种方式并非总是你希望的激活方式。

`ASC` 提供另外四种激活 GameplayAbility 的方法：

1. **通过 GameplayTag**
    
2. **通过 GameplayAbility 类**
    
3. **通过 GameplayAbilitySpecHandle**
    
4. **通过 Event**（允许传递数据负载 Payload）
    

Blueprint ****/ C++ 函数接口示例

```c++
UFUNCTION(BlueprintCallable, Category = "Abilities")
bool TryActivateAbilitiesByTag(const FGameplayTagContainer& GameplayTagContainer, bool bAllowRemoteActivation = true);

UFUNCTION(BlueprintCallable, Category = "Abilities")
bool TryActivateAbilityByClass(TSubclassOf<UGameplayAbility> InAbilityToActivate, bool bAllowRemoteActivation = true);

bool TryActivateAbility(FGameplayAbilitySpecHandle AbilityToActivate, bool bAllowRemoteActivation = true);

bool TriggerAbilityFromGameplayEvent(FGameplayAbilitySpecHandle AbilityToTrigger, FGameplayAbilityActorInfo* ActorInfo, FGameplayTag Tag, const FGameplayEventData* Payload, UAbilitySystemComponent& Component);

FGameplayAbilitySpecHandle GiveAbilityAndActivateOnce(const FGameplayAbilitySpec& AbilitySpec);
```

**通过 Event 激活 Ability**

- `GameplayAbility` 必须设置 **Trigger**，分配一个 `GameplayTag` 并选择 Event 选项。
    
- 发送 Event 使用函数：
    
    ```c++
    UAbilitySystemBlueprintLibrary::SendGameplayEventToActor(AActor* Actor, FGameplayTag EventTag, FGameplayEventData Payload)
    ```
    
- Event 激活允许传递 **数据负载 (Payload)**。
    
- Trigger 也可用于在 `GameplayTag` 添加或移除时自动激活 Ability。
    

> **注意 (Blueprint)**：
> 
> - 使用 Event 激活时必须使用 `ActivateAbilityFromEvent` 节点。
>     
> - 标准的 `ActivateAbility` 节点不能与之共存，否则会一直调用标准节点而忽略 Event 节点。
>     

> **注意**：除非 Ability 是被动技能，否则在 `GameplayAbility` 终止时应调用 `EndAbility()`。

---

##### 客户端预测能力激活序列

**所属客户端 (Owning Client)：**

1. 调用 `TryActivateAbility()`
    
2. 调用 `InternalTryActivateAbility()`
    
3. 调用 `CanActivateAbility()` → 检查：
    
    - GameplayTag 条件
        
    - ASC 是否满足技能花费
        
    - Ability 是否未在冷却期
        
    - 当前实例是否允许激活
        
4. 调用 `CallServerTryActivateAbility()` 并传入 **Prediction Key**
    
5. 调用 `CallActivateAbility()`
    
6. 调用 `PreActivate()`（Epic 称为 "boilerplate init stuff"）
    
7. 调用 `ActivateAbility()` → 最终激活 Ability
    

**服务端 (Server)：**

1. 调用 `ServerTryActivateAbility()`
    
2. 调用 `InternalServerTryActivateAbility()`
    
3. 调用 `InternalTryActivateAbility()`
    
4. 调用 `CanActivateAbility()`（同客户端检查）
    
5. 如果成功：
    
    - 调用 `ClientActivateAbilitySucceed()` → 更新客户端 ActivationInfo 并广播 `OnConfirmDelegate`（与输入确认不同）
        
6. 调用 `CallActivateAbility()`
    
7. 调用 `PreActivate()`
    
8. 调用 `ActivateAbility()` → 最终激活 Ability
    

> **失败处理**：
> 
> - 如果服务端在任意步骤激活失败，会调用 `ClientActivateAbilityFailed()`
>     
> - 立即终止客户端 Ability 并撤销所有预测修改
>     

---

我可以帮你画一张 **客户端预测 + 服务端激活流程图**，把输入、预测 Key、成功/失败反馈都可视化，这样理解会更直观。你想让我画吗？  



<a name="concepts-ga-activating-passive"></a>
我帮你把这段内容优化成更清晰、分层的文档风格：

---

##### 被动 Ability

被动 `GameplayAbility` 的特点是 **自动激活** 并 **持续运行**，无需玩家手动触发。

**实现方式**

- 重写 `UGameplayAbility::OnAvatarSet()`：
    
    - 当 Ability 被授予、`AvatarActor`改变时会调用，可在这里决定是否调用 `TryActivateAbility()` 进行激活。
        
    - Epic 建议将此函数作为初始化被动 Ability 的位置，可执行类似 `BeginPlay` 的逻辑。
        
- 建议在自定义 `UGameplayAbility` 中添加一个 **布尔值**（例如 `ActivateAbilityOnGranted`），用于标记该 Ability 在授予时是否应自动激活。
    
    - 样例项目中的被动护甲叠层 Ability 就是这样实现的。
        

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

- 被动 Ability 通常使用 **仅服务器 (Server Only)** 的网络执行策略 (Net Execution Policy)




<a name="concepts-ga-cancelabilities"></a>
#### 取消Ability

在 GAS 中，可以通过**内部**或**外部**方式取消一个 `GameplayAbility`。

**1. 内部取消**  
Ability 自身可以调用：

```c++
CancelAbility();
```

- 会触发 `EndAbility()`
    
- `WasCancelled` 参数会被设置为 `true`
    
- 常用于 Ability 自己因某些条件提前结束的场景
    

**2. 外部取消（通过 ASC）**  
`AbilitySystemComponent` 提供一系列函数，从外部取消 Ability：

```c++
/** 取消指定的 Ability CDO（Class Default Object） */
void CancelAbility(UGameplayAbility* Ability);	

/** 通过 AbilitySpecHandle 取消 Ability，如果 handle 不存在，则不做任何操作 */
void CancelAbilityHandle(const FGameplayAbilitySpecHandle& AbilityHandle);

/** 取消所有拥有指定 Tags 的 Ability（可排除某个实例） */
void CancelAbilities(
    const FGameplayTagContainer* WithTags = nullptr,
    const FGameplayTagContainer* WithoutTags = nullptr,
    UGameplayAbility* Ignore = nullptr
);

/** 取消所有 Ability（可排除某个实例） */
void CancelAllAbilities(UGameplayAbility* Ignore = nullptr);

/** 取消所有 Ability 并清理残留实例化 Ability 状态 */
virtual void DestroyActiveState();
```

**注意事项：**

1. **非实例化 Ability（Non-Instanced）**
    
    - `CancelAllAbilities()` 可能无法正确处理非实例化 Ability，遇到它时可能直接返回，不会继续取消其他 Ability。
        
    - 对非实例化 Ability，推荐使用 `CancelAbility(UGameplayAbility* Ability)`，更可靠。
        
    - 样例项目中，跳跃技能就是非实例化 Ability，使用 `CancelAbility()` 可以正确取消。
        
2. **批量取消**
    
    - 如果需要取消大部分 Ability，可以使用 `CancelAbilities()` 或 `CancelAllAbilities()`。
        
    - 如果希望彻底清理所有 Ability 的实例状态（比如重置 ASC），使用 `DestroyActiveState()`。
        



#### 获取激活的 Ability

初学者常常问：“我如何获取当前激活的 Ability？”  
需要注意的是：

- **同一时间可能有多个 `GameplayAbility` 激活**，因此并不存在“唯一激活的 Ability”。
    
- 想要获取特定 Ability，需要在 `ASC` 的已授予 Ability 列表中（`ActivatableAbilities`）进行搜索，并匹配对应的 **资源** 或 **GameplayTag**。
    

**1. 遍历所有已授予 Ability**

```c++
TArray<FGameplayAbilitySpec> Abilities = AbilitySystemComponent->GetActivatableAbilities();
for (FGameplayAbilitySpec& Spec : Abilities)
{
    if (Spec.Ability->IsActive())
    {
        // 找到激活的 Ability
    }
}
```

**2. 使用标签过滤**

`ASC` 提供了更方便的函数，可以通过 `GameplayTagContainer` 搜索 Ability，而无需手动遍历列表：

```c++
UAbilitySystemComponent::GetActivatableGameplayAbilitySpecsByAllMatchingTags(
    const FGameplayTagContainer& GameplayTagContainer, 
    TArray<FGameplayAbilitySpec*>& MatchingGameplayAbilities, 
    bool bOnlyAbilitiesThatSatisfyTagRequirements = true
);
```

- `GameplayTagContainer`：你想匹配的标签
    
- `MatchingGameplayAbilities`：返回匹配的 Ability Spec 指针数组
    
- `bOnlyAbilitiesThatSatisfyTagRequirements`：如果为 `true`，只返回那些 **满足标签需求且可以立即激活** 的 Ability
    
    - 例如，你可能有两个攻击 Ability：一个使用武器，一个使用拳头。哪一个可以激活取决于是否装备了武器并满足标签要求。
        

**3. 判断 Ability 是否激活**

获取到 `FGameplayAbilitySpec` 后，可以通过：

```c++
Spec->Ability->IsActive()
```

来判断该 Ability 是否当前处于激活状态。




<a name="concepts-ga-instancing"></a>
#### 实例化策略

`GameplayAbility` 的实例化策略决定了在激活时是否创建新实例以及如何管理实例状态。

|                策略                 |                         说明                         |                                             适用场景                                             |
| :-------------------------------: | :------------------------------------------------: | :------------------------------------------------------------------------------------------: |
| 按 Actor 实例化 (Instanced Per Actor) | 每个 `ASC` 对应一个持续存在的 `GameplayAbility` 实例，可在多次激活间复用。 |                          最常用策略。适合需要在激活间保持状态的 Ability，可在激活之间手动重置变量。                           |
| 按操作实例化 (Instanced Per Execution)  |         每次激活都会创建一个新的 `GameplayAbility` 实例。         |                     适合每次激活都需要独立状态的 Ability。性能低于 Actor 实例化，因为每次激活都生成新实例。                      |
|       非实例化 (Non-Instanced)        |          直接使用 `ClassDefaultObject`，不创建实例。          | 性能最高，但受限。不能存储状态或绑定 `AbilityTask`。适合简单、频繁使用的 Ability，如 MOBA/RTS 小兵基础攻击。样例项目中跳跃 Ability 使用此策略。 |


<a name="concepts-ga-net"></a>
#### 网络执行策略 (Net Execution Policy)

`GameplayAbility` 的网络执行策略决定了该 Ability 在客户端和服务端的运行位置与顺序。

|        策略        |                            说明                            |
| :--------------: | :------------------------------------------------------: |
|    Local Only    |  只在所属客户端运行，通常用于只需视觉效果的 Ability。单人游戏可用 `Server Only` 替代。  |
| Local Predicted  | 首先在客户端预测激活，然后在服务端正式执行。服务端会纠正客户端预测的错误。适用于需要即时响应的 Ability。 |
|   Server Only    |               只在服务端运行，适合被动 Ability 或单人游戏。                |
| Server Initiated |                由服务端触发激活，然后同步到所属客户端。较少使用。                 |


<a name="concepts-ga-tags"></a>
#### Ability 标签

`GameplayAbility` 自带一个内建的 `GameplayTagContainer`，用于管理 Ability 的激活逻辑。这些标签不会自动同步到客户端。

|           标签类型            |                                说明                                 |
| :-----------------------: | :---------------------------------------------------------------: |
|       Ability Tags        |                  Ability 本身拥有的标签，仅用于描述该 Ability。                  |
| Cancel Abilities with Tag |                激活该 Ability 时，会取消其他拥有这些标签的 Ability。                |
| Block Abilities with Tag  |              激活该 Ability 时，会阻止其他拥有这些标签的 Ability 被激活。              |
|   Activation Owned Tags   |                激活该 Ability 时，这些标签会赋予 Ability 的拥有者。                |
| Activation Required Tags  |                  只有拥有者具备所有这些标签时，该 Ability 才能激活。                   |
|  Activation Blocked Tags  |                   拥有者具备任意这些标签时，该 Ability 不能激活。                    |
|   Source Required Tags    | 只有 Ability 的 Source 拥有所有这些标签时，该 Ability 才能激活。仅在 Ability 由事件触发时有效。 |
|    Source Blocked Tags    |       Source 拥有任意这些标签时，该 Ability 不能激活。仅在 Ability 由事件触发时有效。        |
|   Target Required Tags    |      只有 Target 拥有所有这些标签时，该 Ability 才能激活。仅在 Ability 由事件触发时有效。      |
|    Target Blocked Tags    |       Target 拥有任意这些标签时，该 Ability 不能激活。仅在 Ability 由事件触发时有效。        |


<a name="concepts-ga-spec"></a>
#### Gameplay Ability Spec

`GameplayAbilitySpec` 是 `ASC` 在授予 `GameplayAbility` 后保存的结构，用于定义可激活的 Ability。它包含以下信息：Ability 类、等级、输入绑定，以及必须独立于 Ability 类保存的运行时状态。

当 Ability 在服务端授予时，服务端会将 `GameplayAbilitySpec` 同步到所属客户端，使客户端能够激活该 Ability。

激活 `GameplayAbilitySpec` 时，会根据其实例化策略（Instancing Policy）创建对应的 `GameplayAbility` 实例（非实例化 Ability 除外）。


<a name="concepts-ga-data"></a>

#### 传递数据到Ability

`GameplayAbility` 的典型流程是 `Activate → Generate Data → Apply → End`。有时需要将外部数据传入 Ability，GAS 提供了几种常用方式：

|                 方法                 | 描述                                                                                                                                                                                                                                                                                                                                                             |
| :--------------------------------: | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|        通过 Event 激活 Ability         | 使用带有数据负载（Payload）的 Event 激活 Ability。对于客户端预测的 Ability，Payload 会由客户端同步到服务端。对于不适合存入现有变量的数据，可以使用两个可选对象（Optional Object）或 TargetData变量。缺点是无法通过输入绑定激活 Ability。要使用此方法，Ability 必须设置 Trigger，分配一个 GameplayTag 并选择 GameplayEvent 选项。发送事件可用 `UAbilitySystemBlueprintLibrary::SendGameplayEventToActor(AActor* Actor, FGameplayTag EventTag, FGameplayEventData Payload)`。 |
| 使用 `WaitGameplayEvent AbilityTask` | Ability 激活后，可使用此 AbilityTask 监听带 Payload 的事件。Payload 及发送过程与通过 Event 激活相同。缺点是 Event 不能通过 AbilityTask 同步，只能用于 `Local Only` 或 `Server Only` Ability。可自定义 AbilityTask 支持同步 Event Payload。                                                                                                                                                                          |
|           使用 TargetData            | 自定义 TargetData 结构体，可在客户端和服务端之间传递任意数据，适合复杂数据。                                                                                                                                                                                                                                                                                                                   |
|   保存数据在 OwnerActor 或 AvatarActor   | 将数据存储在 Ability 的 OwnerActor、AvatarActor 或其他可访问对象中。适用于输入绑定激活的 Ability，灵活性高。但无法保证数据同步顺序，需提前处理可能的丢包或延迟问题。                                                                                                                                                                                                                                                         |


<a name="concepts-ga-commit"></a>
#### Ability 花费 (Cost) 与冷却 (Cooldown)

`GameplayAbility` 内置可选的 Cost 和 Cooldown 机制。

- **Cost**：由 `ASC` 使用即刻（Instant）`GameplayEffect`（Cost GE）来实现。激活 Ability 前，必须满足预定义的 Attribute 条件。
    
- **Cooldown**：用于阻止 Ability 在冷却完成前再次激活，由持续（Duration）`GameplayEffect`（Cooldown GE）实现。
    

激活流程：

1. **CanActivateAbility**：在调用 `Activate()` 前，系统会先调用 `UGameplayAbility::CanActivateAbility()`，检查：
    
    - 所属 `ASC` 是否满足 Cost (`UGameplayAbility::CheckCost()`)
        
    - Ability 是否处于冷却状态 (`UGameplayAbility::CheckCooldown()`)
        
2. **CommitAbility**：在调用 `Activate()` 后，可选择调用 `UGameplayAbility::CommitAbility()` 提交 Cost 和 Cooldown。
    
    - `CommitAbility()` 内部会调用 `CommitCost()` 和 `CommitCooldown()`
        
    - 可根据需要单独提交 Cost 或 Cooldown
        
    - 提交时会再次调用 `CheckCost()` 和 `CheckCooldown()`，以确保当前 Attribute 状态仍能满足 Cost
        
    - 如果预测 Key 有效，提交过程可以支持客户端预测。

详见CostGE和CooldownGE.



<a name="concepts-ga-leveling"></a>
#### 升级 Ability

升级 Ability 常用的两种方法：

|方法|说明|
|:-:|:--|
|取消授予后重新授予|在服务端先从 `ASC` 移除（取消授予）旧的 `GameplayAbility`，再以新等级重新授予。如果 Ability 当时处于激活状态，会被强制终止。|
|直接增加 `GameplayAbilitySpec` 等级|在服务端找到对应的 `GameplayAbilitySpec`，提升其等级并标记为 Dirty，以同步到所属客户端。此方法不会终止已激活的 Ability。|

区别在于升级时是否希望终止已激活的 Ability。实际使用中，可根据需要选择方法，也可以在 `UGameplayAbility` 子类中增加布尔值来控制升级策略。



<a name="concepts-ga-sets"></a>
#### Ability 集合

`GameplayAbilitySet` 是一个便捷的 `UDataAsset` 类，用于保存输入绑定和初始 `GameplayAbility` 列表，这些 Ability 会在角色拥有授予能力逻辑时被授予。子类可以扩展额外逻辑和属性。在 Paragon 中，每个英雄都有一个 `GameplayAbilitySet` 来管理其所有授予的 Ability。



<a name="concepts-ga-batching"></a>
#### Ability 批处理

通常，一个 `GameplayAbility` 的生命周期至少涉及 2 到 3 个从客户端到服务端的 RPC：

1. `CallServerTryActivateAbility()`
    
2. `ServerSetReplicatedTargetData()`（可选）
    
3. `ServerEndAbility()`
    

如果这些操作在同一帧的原子组(Atomic)内完成，就可以优化为 **批处理**：将原本的 2–3 个 RPC 合并为 1 个。GAS 称这种优化为 **Ability 批处理**。

**典型使用场景**

- **半自动枪**：在一帧内完成射线检测、发送 `TargetData` 到服务端并结束 Ability，这三个操作可以合并为一个 RPC。
    
- **全自动/爆炸枪**：第一发子弹可以将 `CallServerTryActivateAbility()` 和 `ServerSetReplicatedTargetData()` 批处理成一个 RPC，随后每发子弹单独发送 `ServerSetReplicatedTargetData()`，最后停止射击时发送 `ServerEndAbility()`。
    
    - 最坏情况：只优化了第一发子弹的 RPC。
        
    - 可通过 `GameplayEvent` 发送 Payload 实现类似效果，但 `TargetData` 必须在 Ability 外部生成，而批处理方法可在 Ability 内生成。
        

**启用 Ability 批处理**

1. 重写 `ShouldDoServerAbilityRPCBatch()` 返回 `true`：
    

```c++
virtual bool ShouldDoServerAbilityRPCBatch() const override { return true; }
```

2. 在激活要批处理的 Ability 前，创建一个 `FScopedServerAbilityRPCBatcher` 对象。它会在其作用域内拦截并打包所有 RPC，并在退出作用域时自动调用 `UAbilitySystemComponent::EndServerAbilityRPCBatch()` 将批处理 RPC 发送到服务端。
    

服务端通过：

```c++
UAbilitySystemComponent::ServerAbilityRPCBatch_Internal(FServerAbilityRPCBatch& BatchInfo)
```

接收批处理，`BatchInfo` 包含标志位，如 Ability 是否应结束、输入是否按下、是否包含 `TargetData`。可以通过设置断点或 `AbilitySystem.ServerRPCBatching.Log 1` 观察批处理情况。

**示例：批处理激活 Ability**

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

**注意事项**

- 批处理只能在 C++ 中完成。
    
- 激活 Ability 必须通过 `FGameplayAbilitySpecHandle`。
    
- `EndAbilityImmediately` 参数用于控制半自动与全自动枪支的不同批处理策略。
    

GASShooter 中的设计允许通过客户端控制 Ability 的批处理，同时保留单独管理全自动武器结束 Ability 的灵活性。



<a name="concepts-ga-netsecuritypolicy"></a>
#### 网络安全策略 (Net Security Policy)

`GameplayAbility` 的网络安全策略决定了 Ability 在网络中应在哪一端执行，同时避免客户端进行未授权操作。

|          策略           | 说明                                                  |
| :-------------------: | :-------------------------------------------------- |
|    ClientOrServer     | 没有安全限制。客户端或服务端都可以自由执行或结束该 Ability。                  |
|  ServerOnlyExecution  | 客户端尝试执行该 Ability 会被服务端忽略，但客户端仍可请求服务端取消或结束该 Ability。 |
| ServerOnlyTermination | 客户端尝试取消或结束该 Ability 会被服务端忽略，但客户端仍可请求执行该 Ability。    |
|      ServerOnly       | 服务端完全控制 Ability 的执行和结束，客户端的任何请求都会被忽略。               |
<a name="concepts-at"></a>
### AbilityTask

<a name="concepts-at-definition"></a>
#### AbilityTask 定义

`GameplayAbility` 激活逻辑在单帧内执行，灵活性有限。为了实现能够随时间触发或在延迟后响应的操作，需要使用 `AbilityTask`。

GAS 提供了多种常用的 `AbilityTask`：

- 使用 `RootMotionSource` 移动 Character
    
- 播放动画蒙太奇
    
- 响应 `Attribute` 变化
    
- 响应 `GameplayEffect` 变化
    
- 响应玩家输入
    
- 其他功能
    

`UAbilityTask` 的构造函数中默认允许同时运行的 Task 最多为 1000 个。在设计同时拥有数百个 Character 的游戏（如 RTS）的 `GameplayAbility` 时，需要注意这个限制。



<a name="concepts-at-definition"></a>
#### 自定义 AbilityTask

通常需要在 C++ 中创建自定义的 `AbilityTask`。样例项目中提供了两个示例：

1. **PlayMontageAndWaitForEvent**  
    结合了默认的 `PlayMontageAndWait` 和 `WaitGameplayEvent` Task，允许动画蒙太奇通过 `AnimNotify` 发送 `GameplayEvent` 回到启动它的 `GameplayAbility`。可在动画的特定时刻触发操作。
    
2. **WaitReceiveDamage**  
    监听 `OwnerActor` 接收伤害。当英雄收到伤害实例时，被动护甲层 `GameplayAbility` 会移除一层护甲。
    

`AbilityTask` 的基本组成：

- 静态函数，用于创建新的 `AbilityTask` 实例
    
- 输出委托（Delegate），在任务完成时触发
    
- `Activate()` 函数，执行主要逻辑并绑定外部委托
    
- `OnDestroy()` 函数，进行清理，包括解绑外部委托
    
- 所有绑定到外部委托的回调函数
    
- 成员变量及内部辅助函数
    

**注意事项**：

- `AbilityTask` 只能声明一种类型的输出委托。所有输出委托必须为该类型，未使用的参数会传递默认值。
    
- `AbilityTask` 默认在拥有其 `GameplayAbility` 的客户端或服务端运行。可以通过设置 `bSimulatedTask = true` 在 Simulated Client 上运行，并在构造函数中重写 `InitSimulatedTask()` 来同步必要的成员变量。这适用于例如 `RootMotionSource AbilityTask` 这样的移动任务，模拟整个任务，而而不是同步每次位置变化。
    
- 如果在构造函数中设置 `bTickingTask = true` 并重写 `TickTask(float DeltaTime)`，`AbilityTask` 就可以每帧执行逻辑，这对于需要平滑线性插值的场景非常有用。可参考 `AbilityTask_MoveToLocation.h/.cpp`。



<a name="concepts-at-using"></a>
#### 使用 AbilityTask

在 C++ 中创建并激活 `AbilityTask` 示例：

```c++
UGDAT_PlayMontageAndWaitForEvent* Task = UGDAT_PlayMontageAndWaitForEvent::PlayMontageAndWaitForEvent(
    this, NAME_None, MontageToPlay, FGameplayTagContainer(), 1.0f, NAME_None, false, 1.0f
);
Task->OnBlendOut.AddDynamic(this, &UGDGA_FireGun::OnCompleted);
Task->OnCompleted.AddDynamic(this, &UGDGA_FireGun::OnCompleted);
Task->OnInterrupted.AddDynamic(this, &UGDGA_FireGun::OnCancelled);
Task->OnCancelled.AddDynamic(this, &UGDGA_FireGun::OnCancelled);
Task->EventReceived.AddDynamic(this, &UGDGA_FireGun::EventReceived);
Task->ReadyForActivation();
```

**要点说明：**

- 在蓝图中，只需使用对应的 `AbilityTask` 节点即可，无需手动调用 `ReadyForActivation()`。
    
    - 蓝图节点通过 `K2Node_LatentGameplayTaskCall` 自动处理 `ReadyForActivation()`、`BeginSpawningActor()` 和 `FinishSpawningActor()`（如果 `AbilityTask` 类实现了这些函数，如 `AbilityTask_WaitTargetData`）。
        
- 在 C++ 中，这些操作必须手动调用：
    
    - `BeginSpawningActor()`
        
    - `FinishSpawningActor()`:AbilityTask_WaitTargetData需要调用
        
    - `ReadyForActivation()`:AbilityTask_WaitTargetData需要调用
        
- 若需要手动取消 `AbilityTask`，在蓝图（Async Task Proxy）或 C++ 中调用 `EndTask()` 即可。



<a name="concepts-at-rms"></a>
#### RootMotionAbilityTask

GAS 自带的 `AbilityTask` 可以利用挂载在 `CharacterMovementComponent` 上的 `Root Motion Source` 实现随时间移动 `Character`，常用于击退、复杂跳跃、吸引或冲锋等效果。

**注意事项：**

- `RootMotionSource AbilityTask` 的客户端预测在引擎版本 4.19 和 4.25+ 中支持良好。
    
- 在 4.20–4.24 版本中，预测存在已知 Bug，但 `AbilityTask` 仍可通过小幅网络修正实现多人游戏功能，并在单人游戏中正常运行。
    
- 如果需要，也可以将 4.25 对预测的修复移植到 4.20–4.24 引擎版本中。



<a name="concepts-gc"></a>
### Targeting(目标选择)

<a name="concepts-targeting-data"></a>
#### Target Data

`FGameplayAbilityTargetData`是用于通过网络传输定位数据的通用结构体。`TargetData`一般用于保存AActor/UObject引用、FHitResult和其他通用的Location/Direction/Origin信息。然而，本质上你可以继承它以增添想要的任何数据，其可以简单理解为在客户端和服务端的GameplayAbility中传递数据。

基础结构体`FGameplayAbilityTargetData`不能直接使用，而是要继承它。GAS的`GameplayAbilityTargetTypes.h`中有一些开箱即用的派生`FGameplayAbilityTargetData`结构体。

`TargetData`一般由Target Actor或者手动创建，供AbilityTask使用，或者GameplayEffect通过EffectContext使用。因为其位于`EffectContext`中，所以Execution、MMC、GameplayCue和AttributeSet的后端函数可以访问该`TargetData`。

我们一般不直接传递`FGameplayAbilityTargetData`而是使用`FGameplayAbilityTargetDataHandle`，其包含一个`FGameplayAbilityTargetData`指针类型的TArray，这个中间结构体可以为`TargetData`的多态性提供支持。



<a name="concepts-targeting-actors"></a>
#### Target Actor

`GameplayAbility`通过`WaitTargetData AbilityTask`生成TargetActor，用于在世界中可视化和获取定位信息。`TargetActor`可以选择使用GameplayAbilityWorldReticles显示当前目标。确认后，定位信息作为TargetData返回，可传递给`GameplayEffect`。

`TargetActor`继承自`AActor`，可使用任意可视组件表示其位置，如静态网格物(Static Mesh)或贴花(Decal)。静态网格物可用于显示角色将要建造的物体，贴花可显示地面效果区域。例如，样例项目中的AGameplayAbilityTargetActor_GroundTrace用于陨石技能的伤害区域效果。也可以不显示任何内容，如GASShooter中的枪击射线检测。

`TargetActor`通过射线或Overlap获取定位信息，根据实现将`FHitResult`或`AActor`数组转换为`TargetData`。`WaitTargetData AbilityTask`使用`TEnumAsByte<EGameplayTargetingConfirmation::Type> ConfirmationType`决定目标确认时机：

- **Instant**：立即生成并返回TargetData，无需Tick()。
    
- **UserConfirmed**：用户确认或调用`UAbilitySystemComponent::TargetConfirm()`触发，支持取消输入或`TargetCancel()`。
    
- **Custom / CustomMulti**：通过`UGameplayAbility::ConfirmTaskByInstanceName()`决定确认时机，支持多次确认(CustomMulti)。
    

并非所有TargetActor支持所有确认类型，例如`AGameplayAbilityTargetActor_GroundTrace`不支持Instant。

`WaitTargetData AbilityTask`每次激活都会生成`AGameplayAbilityTargetActor`实例，并在结束时销毁。`WaitTargetDataUsingActor`使用已有TargetActor，但仍会销毁它。两者在原型开发中方便，但在高频场景下效率低，实际发布版本可参考GASShooter的自定义子类和WaitTargetDataWithReusableActor AbilityTask实现复用。

TargetActor默认不可同步，但可设计为可同步，通过RPC与服务端通信。若`ShouldProduceTargetDataOnServer=false`，客户端通过`CallServerSetReplicatedTargetData()`发送TargetData到服务端；若为true，则客户端发送`EAbilityGenericReplicatedEvent::GenericConfirm`事件，由服务端生成TargetData。取消时发送`GenericCancel`事件。服务端可校验TargetData以防作弊，直接在服务端生成TargetData可避免作弊，但可能影响客户端预测。

常用TargetActor参数：

|参数|说明|
|:--|:--|
|Debug|非发行版本下绘制射线/Overlap调试信息。|
|Filter|可选，用于过滤射线/Overlap命中的Actor，典型案例是排除玩家Pawn。|
|Reticle Class|可选，生成的`AGameplayAbilityWorldReticle`子类。|
|Reticle Parameters|可选，配置Reticle参数。|
|Start Location|射线检测起点，一般为玩家视口、枪口或Pawn位置。|

默认TargetActor只在射线/Overlap命中时有效，如果Target离开或视线偏移，则无效。若希望记住最后有效Target，可自定义TargetActor实现“持久化Target”，持续存在直到确认或取消，例如GASShooter火箭筒二技能的制导火箭。



<a name="concepts-target-data-filters"></a>
#### TargetData过滤器

你可以通过`Make GameplayTargetDataFilter`和`Make Filter Handle`节点来过滤Target，例如排除玩家的`Pawn`或仅选择特定类。对于更复杂的过滤条件，可以继承`FGameplayTargetDataFilter`并重写`FilterPassesForActor`函数：

```c++
USTRUCT(BlueprintType)
struct GASDOCUMENTATION_API FGDNameTargetDataFilter : public FGameplayTargetDataFilter
{
	GENERATED_BODY()

	/** 返回true表示Actor通过过滤，可被选为Target */
	virtual bool FilterPassesForActor(const AActor* ActorToBeFiltered) const override;
};
```

注意，这个自定义过滤器不能直接用于`WaitTargetData`节点，因为它需要`FGameplayTargetDataFilterHandle`。需要创建一个自定义的`Make Filter Handle`函数，将子类包装为Filter Handle：

```c++
FGameplayTargetDataFilterHandle UGDTargetDataFilterBlueprintLibrary::MakeGDNameFilterHandle(
    FGDNameTargetDataFilter Filter, 
    AActor* FilterActor)
{
    FGameplayTargetDataFilter* NewFilter = new FGDNameTargetDataFilter(Filter);
    NewFilter->InitializeFilterContext(FilterActor);

    FGameplayTargetDataFilterHandle FilterHandle;
    FilterHandle.Filter = TSharedPtr<FGameplayTargetDataFilter>(NewFilter);
    return FilterHandle;
}
```

这样生成的`FGameplayTargetDataFilterHandle`即可在`WaitTargetData AbilityTask`或其他TargetData节点中使用，实现自定义过滤逻辑。



<a name="concepts-targeting-reticles"></a>
#### Gameplay Ability World Reticles

在使用非Instant TargetActor进行定位时，AGameplayAbilityWorldReticle可以可视化目标位置。`TargetActor`负责`Reticle`的生成和销毁。

`Reticle`继承自`AActor`，可使用任意可视组件表现。例如，GASShooter常用`WidgetComponent`在屏幕空间显示UMG Widget（始终面向玩家摄像机）。默认情况下，`Reticle`不知道其正在定位的Actor，但可以通过自定义`TargetActor`实现该功能。通常在每次`Tick()`中将`Reticle`位置更新为Target位置。

GASShooter中火箭筒二技能制导火箭锁定的敌人目标使用了`Reticle`。敌人身上的红色标识就是`Reticle`，白色图像是火箭筒准星。

`Reticle`提供一些供蓝图使用的事件：

```c++
/** 当 bIsTargetValid 改变时触发 */
UFUNCTION(BlueprintImplementableEvent, Category = Reticle)
void OnValidTargetChanged(bool bNewValue);

/** 当 bIsTargetAnActor 改变时触发 */
UFUNCTION(BlueprintImplementableEvent, Category = Reticle)
void OnTargetingAnActor(bool bNewValue);

/** 参数初始化完成时触发 */
UFUNCTION(BlueprintImplementableEvent, Category = Reticle)
void OnParametersInitialized();

/** 设置Reticle材质浮点参数 */
UFUNCTION(BlueprintImplementableEvent, Category = Reticle)
void SetReticleMaterialParamFloat(FName ParamName, float value);

/** 设置Reticle材质向量参数 */
UFUNCTION(BlueprintImplementableEvent, Category = Reticle)
void SetReticleMaterialParamVector(FName ParamName, FVector value);
```

`Reticle`可以使用`TargetActor`提供的FWorldReticleParameters进行配置。默认结构体仅提供`FVector AOEScale`变量。若创建自定义`TargetActor`，可以传入自定义Reticle参数结构体，并在生成`Reticle`时使用。

默认情况下，`Reticle`不可同步，但可根据需求设计为可同步，以便其他玩家看到本地玩家定位的目标。

`Reticle`只显示在当前有效Target上。例如使用`AGameplayAbilityTargetActor_SingleLineTrace`时，敌人只有处于射线路径上才显示Reticle；若视线偏离，则Reticle消失。若希望Reticle保留在最后一个有效Target上，需要自定义`TargetActor`实现“持久化Target”，直到确认或取消。GASShooter火箭筒二技能制导火箭即使用了这种持久化Target逻辑。



<a name="concepts-targeting-containers"></a>
#### Gameplay Effect Containers Targeting

`GameplayEffectContainer`提供了一种高效生成TargetData的方式。与TargetActor不同，它直接在定位对象的CDO（Class Default Object）上运行，因此无需生成或销毁Actor，效率更高。

特点与限制：

- **即时生成**：在客户端和服务端应用时立即产生TargetData，无需用户输入或确认。
    
- **不可取消**：一旦生成，无法撤销。
    
- **无法同步**：不能从客户端向服务端发送TargetData，但在两端都会生成数据。
    
- **适用场景**：适合即时射线检测或碰撞Overlap。
    

Epic的Action RPG Sample Project展示了两种使用Container进行定位的方式：

1. 定位Ability拥有者。
    
2. 从事件中拉取TargetData。
    

此外，它在蓝图中实现了在玩家某个偏移位置进行球形射线检测（Sphere Trace），偏移由蓝图子类设置。你也可以在C++或蓝图中继承`URPGTargetType`来实现自定义定位类型。


### GameplayCue

<a name="concepts-gc-definition"></a>
#### GameplayCue 定义

`GameplayCue (GCue)` 负责处理与游戏逻辑无关的表现效果，比如音效、粒子特效、镜头震动等。  
GC 默认是**可同步且可预测**的（除非显式在客户端调用 `Execute`、`Add` 或 `Remove` 事件）。

在 `ASC` 中，我们可以通过发送一个**必须以 "GameplayCue" 为前缀**的 `GameplayTag`，并指定 `GameplayCueManager` 的事件类型（Execute、Add、Remove），来触发对应的 GC。  
`GameplayCueNotify` 对象以及实现了 `IGameplayCueInterface` 的 `Actor`，都可以基于该 `GameplayCueTag` 订阅这些事件。

**注意：**  
GC 的 `GameplayTag` 必须以 **GameplayCue** 开头，例如：`GameplayCue.A.B.C`。

---

#### GameplayCue 类型

| 类别                       | 触发事件       | GameplayEffect 类型   | 描述                                                                                                                      |
| ------------------------ | ---------- | ------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| GameplayCueNotify_Static | Execute    | Instant 或 Periodic  | **静态 GC** 基于 `ClassDefaultObject`（无实例），只执行一次逻辑，适合瞬时效果（如击中伤害）。                                                           |
| GameplayCueNotify_Actor  | Add/Remove | Duration 或 Infinite | **Actor GC** 在 `Added` 时生成实例，可以在一段时间内持续执行逻辑，直到被 `Removed`。常用于循环播放的音效、粒子特效等。支持设置“同类效果的最大并发数”，避免相同效果被重复实例化。适合持续或无限时长的 GE。 |

虽然 GC 技术上可以响应任何事件，但一般我们按上述方式使用。

---

**关键注意点**

- 使用 `GameplayCueNotify_Actor` 时，请启用 **Auto Destroy on Remove**，否则相同的 GC 在后续调用 `Add` 时将无法再次生效。
    
- 在使用 **非 Full 同步模式** 的 `ASC` 时，`Add` 和 `Remove` 事件在服务端监听玩家（Listen Server）会触发两次：
    
    1. GE 应用时触发
        
    2. GE 的 `NetMulticast` 同步到客户端时再次触发  
        但 **WhileActive 事件只会触发一次**。  
        在客户端，所有事件都只触发一次。
        
- 示例项目中：
    
    - `GameplayCueNotify_Actor` 用于 **眩晕** 和 **奔跑特效**
        
    - `GameplayCueNotify_Static` 用于 **子弹命中伤害特效**
        
    - 这些 GC 也可以通过 **客户端直接触发** 来优化，而不是依赖 GE 同步
        



<a name="concepts-gc-trigger"></a>
#### 触发GameplayCue

##### GameplayEffect

在成功应用(未被Tag或Immunity阻塞)的`GameplayEffect`中填写所有应该触发的`GameplayCue`的`GameplayTag`.  

  [[attachments/9ca952a53d4b0920600243fff04ffb72_MD5.png|Open: Pasted image 20250820210014.png]]
![[attachments/9ca952a53d4b0920600243fff04ffb72_MD5.png]]

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



<a name="concepts-gc-local"></a>
可以这样改写成更精准、易理解的版本：

---

#### 客户端 GameplayCue

默认情况下，从 `GameplayAbility` 或 `ASC` 触发的 `GameplayCue` 都是**同步的**，即会通过多播（Multicast）RPC在所有客户端和服务端之间传播。  
这会带来两个问题：

1. **RPC开销大** —— 每个 `GameplayCue` 事件都是一次 RPC，同步频繁触发时网络压力很大。
    
2. **频率限制** —— GAS 强制在一次网络更新中，`GameplayCue`RPC 最多只能发送两次，多了就会被丢弃。

    >这个RPC可以通过控制台变量MaxRPCPerNetUpdate改,比如改成10

为了解决这些问题，我们可以使用**客户端 GameplayCue**：只在**本地客户端**执行 `Execute`、`Add` 或 `Remove`，不经过网络同步。

---

✅ 适合用客户端 `GameplayCue` 的场景：

- 抛射物命中时的伤害特效
    
- 近战武器的碰撞反馈（如火花、击打特效）
    
- 动画蒙太奇中触发的特效或音效
    

---

**实现方式**

在 `ASC` 子类中添加以下函数，用于仅在本地触发 `GameplayCue`：

```c++
UFUNCTION(BlueprintCallable, Category = "GameplayCue", Meta = (AutoCreateRefTerm = "GameplayCueParameters", GameplayTagFilter = "GameplayCue"))
void ExecuteGameplayCueLocal(const FGameplayTag GameplayCueTag, const FGameplayCueParameters& GameplayCueParameters);

UFUNCTION(BlueprintCallable, Category = "GameplayCue", Meta = (AutoCreateRefTerm = "GameplayCueParameters", GameplayTagFilter = "GameplayCue"))
void AddGameplayCueLocal(const FGameplayTag GameplayCueTag, const FGameplayCueParameters& GameplayCueParameters);

UFUNCTION(BlueprintCallable, Category = "GameplayCue", Meta = (AutoCreateRefTerm = "GameplayCueParameters", GameplayTagFilter = "GameplayCue"))
void RemoveGameplayCueLocal(const FGameplayTag GameplayCueTag, const FGameplayCueParameters& GameplayCueParameters);
```

对应的 C++ 实现：

```c++
void UPAAbilitySystemComponent::ExecuteGameplayCueLocal(
    const FGameplayTag GameplayCueTag,
    const FGameplayCueParameters& GameplayCueParameters)
{
    UAbilitySystemGlobals::Get().GetGameplayCueManager()->HandleGameplayCue(
        GetOwner(), GameplayCueTag, EGameplayCueEvent::Executed, GameplayCueParameters);
}

void UPAAbilitySystemComponent::AddGameplayCueLocal(
    const FGameplayTag GameplayCueTag,
    const FGameplayCueParameters& GameplayCueParameters)
{
    UAbilitySystemGlobals::Get().GetGameplayCueManager()->HandleGameplayCue(
        GetOwner(), GameplayCueTag, EGameplayCueEvent::OnActive, GameplayCueParameters);

    UAbilitySystemGlobals::Get().GetGameplayCueManager()->HandleGameplayCue(
        GetOwner(), GameplayCueTag, EGameplayCueEvent::WhileActive, GameplayCueParameters);
}

void UPAAbilitySystemComponent::RemoveGameplayCueLocal(
    const FGameplayTag GameplayCueTag,
    const FGameplayCueParameters& GameplayCueParameters)
{
    UAbilitySystemGlobals::Get().GetGameplayCueManager()->HandleGameplayCue(
        GetOwner(), GameplayCueTag, EGameplayCueEvent::Removed, GameplayCueParameters);
}
```

---

⚠️ 注意事项：

- **本地添加 → 本地移除**：如果一个 `GameplayCue` 是在本地添加的，那也必须由本地移除。
    
- **同步添加 → 同步移除**：如果一个 `GameplayCue` 是通过网络同步触发的，那也必须通过同步方式来移除。
    




<a name="concepts-gc-parameters"></a>
#### GameplayCue 参数

`GameplayCue` 使用 `FGameplayCueParameters` 结构体作为参数，用于携带额外信息。

- **自动填充情况**  
    当 `GameplayCue` 由 `GameplayEffect` 触发时，以下字段会被自动填充：
    
    - `AggregatedSourceTags`
        
    - `AggregatedTargetTags`
        
    - `GameplayEffectLevel`
        
    - `AbilityLevel`
        
    - EffectContext
        
    - `Magnitude`（当 `GameplayEffect` 在 `GameplayCue` 标签容器的下拉列表中指定了 `Magnitude Attribute` 且该属性受 `Modifier` 影响时）
        
- **手动触发情况**  
    当从 `GameplayAbility` 或 `ASC` 中手动触发时，需要显式填充 `GameplayCueParameters`。其中 `SourceObject` 可用于传递自定义数据到 `GameplayCue`。
    
- **数据扩展方式**
    
    - 一些参数（如 `Instigator`）已经存在于 `EffectContext` 中。
        
    - `EffectContext` 还能包含 `FHitResult`，用于确定 `GameplayCue` 的世界位置。
        
    - 通过继承 `EffectContext` 可传递更多自定义数据，尤其适合由 `GameplayEffect` 触发的 `GameplayCue`。
        
- **可扩展接口**  
    在UAbilitySystemGlobals中，有三个可重写的虚函数
    
    ```c++
    /** Initialize GameplayCue Parameters */
    virtual void InitGameplayCueParameters(FGameplayCueParameters& CueParameters, const FGameplayEffectSpecForRPC &Spec);
    virtual void InitGameplayCueParameters_GESpec(FGameplayCueParameters& CueParameters, const FGameplayEffectSpec &Spec);
    virtual void InitGameplayCueParameters(FGameplayCueParameters& CueParameters, const FGameplayEffectContextHandle& EffectContext);
    ```



<a name="concepts-gc-manager"></a>
#### Gameplay Cue Manager

默认情况下，游戏启动时 `GameplayCueManager` 会扫描项目目录并加载所有 `GameplayCueNotify`。可以在 `DefaultGame.ini` 中修改其扫描路径为你GameplayCue存放路径这样就不会全局扫描了：

```ini
[/Script/GameplayAbilities.AbilitySystemGlobals]
GameplayCueNotifyPaths="/Game/GASDocumentation/Characters"
```

虽然我们希望 `GameplayCueManager` 能扫描并找到所有的 `GameplayCueNotify`，但并不希望它异步加载每一个。因为这会导致所有 `GameplayCueNotify` 及其引用的音效、粒子等资源都被加载进内存，无论它们是否真正被使用。在大型游戏（如 Paragon）中，这可能占用数百兆内存，并造成卡顿或启动无响应。

一种优化方式是：只异步加载实际会触发的 `GameplayCue`。这样既能避免无用资源的内存占用，也能减少首次触发时的延迟。SSD 基本不会出现延迟，但在 HDD 或 UE 编辑器中，若需要编译粒子系统，首次加载时可能会有轻微卡顿。在打包后的构建版本中不存在这个问题，因为粒子系统已提前编译。

为实现这一点，需要继承 `UGameplayCueManager` 并在 `DefaultGame.ini` 中指定使用自定义子类：

```ini
[/Script/GameplayAbilities.AbilitySystemGlobals]
GlobalGameplayCueManagerClass="/Script/ParagonAssets.PBGameplayCueManager"
```

然后在自定义的 `UGameplayCueManager` 子类中重写：

```c++
virtual bool ShouldAsyncLoadRuntimeObjectLibraries() const override
{
    return false;
}
```




<a name="concepts-gc-prevention"></a>
#### 阻止 GameplayCue 响应

在某些情况下我们并不希望 `GameplayCue` 生效。比如，当一次攻击被成功阻止时，可能就不需要播放附加在伤害 `GameplayEffect` 上的击打特效或自定义效果。

实现方式有两种：

- 在 `GameplayEffectExecutionCalculations` 中调用 `OutExecutionOutput.MarkGameplayCuesHandledManually()`，这样可以阻止默认的 `GameplayCue` 触发，然后根据需要手动向 `Target` 或 `Source` 的 `ASC` 发送自定义的 `GameplayCue` 事件。
    
- 如果希望某个特定 `ASC` 中的所有 `GameplayCue` 永不触发，可以直接设置：
    

```c++
AbilitySystemComponent->bSuppressGameplayCues = true;
```

这样就能完全屏蔽该 `ASC` 的 `GameplayCue` 响应。



<a name="concepts-gc-batching"></a>
#### GameplayCue批处理

每次`GameplayCue`触发都是一次不可靠的多播(NetMulticast)RPC. 在同一时刻触发多个`GameplayCue`的情况下, 有一些优化方法来将它们压缩成一个RPC或者通过发送更少的数据来节省带宽.  



<a name="concepts-gc-batching-manualrpc"></a>
##### 手动 RPC

以一把能发射 8 枚弹丸的霰弹枪为例，每次开火都会触发 8 个轨迹和碰撞的 `GameplayCue`。在 GASShooter 中采用的是延迟（Lazy）合并的方法：将所有轨迹信息存入`EffectContext `作为 `TargetData`，再通过一个 RPC 发送。这种方式虽然将 RPC 数量从 8 个减少为 1 个，但单个 RPC 仍可能包含较大的数据量（约 500 bytes）。

进一步优化的思路是通过**自定义结构体 RPC** 来传输数据：

- 在 RPC 中高效编码命中位置（Hit Location），或只发送一个随机种子，在客户端重现/近似计算碰撞结果。
    
- 客户端收到自定义结构体后解包，并触发本地的GameplayCue。
    

##### 运行机制

1. **声明 `FScopedGameplayCueSendContext`**  
    阻止 `UGameplayCueManager::FlushPendingCues()` 的执行，直到该作用域结束。所有 `GameplayCue` 会先排队等待。
    
2. **重写 `UGameplayCueManager::FlushPendingCues()`**  
    将可批处理的 `GameplayCue`（通过特定 `GameplayTag` 区分）合并进自定义结构体，并通过 RPC 发送到客户端。
    
3. **客户端接收与解包**  
    客户端接收自定义结构体，解析其中的参数，并触发对应的本地 `GameplayCue`。
    

**适用场景**

这种方法不仅能降低网络带宽占用，还可以在 `GameplayCue` 需要**特殊参数**时使用，比如：

- 伤害数值
    
- 暴击标识
    
- 破盾标识
    
- 处决标识
    

这些参数往往不便放入 `GameplayCueParameter` 或 `EffectContext` 中，通过自定义 RPC 可以灵活处理。

👉 参考：[官方论坛讨论](https://forums.unrealengine.com/development-discussion/c-gameplay-programming/1711546-fscopedgameplaycuesendcontext-gameplaycuemanager)

---

要不要我帮你画一个**流程图**，展示 `FScopedGameplayCueSendContext` 的工作原理和数据流动？



<a name="concepts-gc-batching-gcsonge"></a>
#### GameplayEffect 中的多个 GameplayCue

在一个 `GameplayEffect` 中的所有 `GameplayCue` 默认会通过单个 RPC 发送。

- 默认情况下，`UGameplayCueManager::InvokeGameplayCueAddedAndWhileActive_FromSpec()` 会在不可靠的多播（NetMulticast）RPC 中发送整个 `GameplayEffectSpec`（除了转换为 `FGameplayEffectSpecForRPC`），不考虑 `ASC` 的同步模式。
    
- 根据 `GameplayEffectSpec` 的内容，这可能会消耗大量网络带宽。
    

**优化方式**：

通过在配置中设置：

```ini
AbilitySystem.AlwaysConvertGESpecToGCParams=1
```

- 这样 `GameplayEffectSpec` 会被转换为轻量级的 `FGameplayCueParameter` 结构体再进行 RPC，而不是发送整个 `FGameplayEffectSpecForRPC`。
    
- 优点：显著节省带宽。
    
- 缺点：传递的信息较少，具体可用信息取决于 `GESpec` 如何被转换为 `GameplayCueParameters`，以及你的 `GameplayCue` 实际需要哪些数据。
    

这种方法适合大多数只需基础参数的 `GameplayCue` 场景，同时避免网络拥堵。 



<a name="concepts-asg"></a>
### AbilitySystemGlobal

`AbilitySystemGlobals` 类用于保存 GAS（Gameplay Ability System）的全局信息。大多数变量都可以通过 `DefaultGame.ini` 配置。

- 一般情况下不需要直接与该类交互，但需要了解其存在。
    
- 如果想继承`GameplayCueManager` 或 `GameplayEffectContext`，就必须通过 `AbilitySystemGlobals` 来指定自定义类。
    

#### 配置自定义 AbilitySystemGlobals

在 `DefaultGame.ini` 中设置自定义类名：

```c++
[/Script/GameplayAbilities.AbilitySystemGlobals]
AbilitySystemGlobalsClassName="/Script/ParagonAssets.PAAbilitySystemGlobals"
```

这样 GAS 系统会使用你的自定义 `AbilitySystemGlobals` 类来管理全局配置和对象。



<a name="concepts-asg-initglobaldata"></a>
#### InitGlobalData()

从 UE 4.24 开始，必须调用 `UAbilitySystemGlobals::InitGlobalData()` 才能安全使用TargetData。

- **不调用的后果**：
    
    - 会出现与 `ScriptStructCache` 相关的错误
        
    - 客户端可能会因服务端断开而掉线
        
- **调用次数**：只需在项目中调用一次即可。
    

##### 调用位置参考

- **Fortnite**：在 `AssetManager` 的StartInitialLoading()中调用
    
- **Paragon**：在 `UEngine::Init()` 中调用
    
- **推荐位置**：在 `AssetManager` 的StartInitialLoading()中调用
    

##### 注意事项

- 如果在使用 `AbilitySystemGlobals::GlobalAttributeSetDefaultsTableNames` 时发生崩溃，可能需要将 `InitGlobalData()` 调用移动到 `AssetManager` 或 `GameInstance` 中，而不是放在 `UEngineSubsystem::Initialize()`。
    
- 这种崩溃通常由 `Subsystem` 的加载顺序引起：`GlobalAttributeDefaultsTables` 需要依赖 `EditorSubsystem` 来绑定 `UAbilitySystemGlobals::InitGlobalData()` 中的委托。
    

这样可以确保全局属性和 TargetData 正确初始化，避免运行时崩溃。



<a name="concepts-p"></a>
### Prediction（预测）

GAS 提供开箱即用的客户端预测功能，但不能预测所有内容。预测的核心意义是：客户端可以在无需等待服务端确认的情况下激活 `GameplayAbility` 和应用 `GameplayEffect`。

- 客户端预测后，服务端会验证预测是否正确。
    
- 如果预测错误，客户端会“回滚”错误的修改，以与服务端状态保持一致。
    

#### 预测源码参考

`GameplayPrediction.h` 是最直接的源码参考。

---

#### Epic 的预测理念

- 只预测“可以不被惩罚的操作”（“get away with”）。
    
- 不建议预测伤害值或死亡等关键状态。
    
- Paragon 和 Fortnite 使用 [ExecutionCalculations](https://chatgpt.com/c/68a5d131-5dcc-8329-b0df-b80ddc56c485#concepts-ge-ec) 来处理伤害，这些无法预测。
    

> “…we are also not all in on a 'predict everything: seamlessly and automatically' solution. We still feel player prediction is best kept to a minimum (meaning: predict the minimum amount of stuff you can get away with).”  
> — Epic, Dave Ratti

---

#### 可预测内容

- Ability 激活
    
- 触发事件
    
- GameplayEffect 应用
    
    - Attribute 修改（Execution 例外）
        
    - GameplayTag 修改
        
- GameplayCue 事件（包含 GameplayEffect 内的和自身事件）
    
- 蒙太奇
    
- 移动（UE4 内建 UCharacterMovement 支持）
    

#### 不可预测内容

- GameplayEffect 移除
    
- GameplayEffect 周期效果（DOT）
    

> 注意：即使可以预测 `GameplayEffect` 的应用，也无法预测移除。可选变通方法：预测性地应用相反效果（如降低 40% 移动速度的效果可以通过增加 40% 移动速度的 Buff 来抵消）。

---

#### 对 Cooldown 的影响

- 由于无法预测 `GameplayEffect` 移除，`GameplayAbility` 冷却无法完全预测。
    
- 高延迟玩家会比低延迟玩家触发能力更慢。
    
- Fortnite 通过自定义 bookkeeping 而非依赖 `Cooldown GE` 来规避该问题。
    

---

#### 关于预测伤害与死亡

- 不建议预测伤害值或死亡。
    
- 错误预测可能导致生命值跳动或布娃娃模拟异常，影响游戏体验。
    

---

#### 即刻 GameplayEffect 的预测

- 自己的即时（Instant）`GameplayEffect`（如 Cost GE）可无缝预测。
    
- 对其他角色的即时 `GameplayEffect` 预测可能产生短暂异常。
    
- 即刻 `GameplayEffect` 被视为无限（Infinite），预测错误可以回滚。
    

---

#### GAS 预测实现解决的问题

1. **能否执行（Can I do this?）**
    
    - 客户端预测的基本协议
        
2. **撤销（Undo）**
    
    - 预测错误时如何消除副作用
        
3. **重复作用（Redo）**
    
    - 如何避免已经在客户端预测但还是从服务端同步的重播副作用
        
4. **完整性（Completeness）**
    
    - 确保预测涵盖所有副作用
        
5. **依赖（Dependencies）**
    
    - 管理预测事件链和依赖
        
6. **重写（Override）**
    
    - 如何预测性地覆盖服务端同步状态
        

源自 `GameplayPrediction.h`



<a name="concepts-p-key"></a>
#### Prediction Key

GAS的预测机制基于`Prediction Key`，它是客户端激活`GameplayAbility`时生成的整型标识符。

- 客户端激活`GameplayAbility`时生成`Prediction Key`，称为`Activation Prediction Key`。
    
- 客户端通过`CallServerTryActivateAbility()`将该`Prediction Key`发送到服务端。
    
- 在`Prediction Key`有效期间，客户端会将其应用到所有相关的`GameplayEffect`。
    
- 当客户端的`Prediction Key`失效时，该`GameplayAbility`中的预测效果需要通过新的`Scoped Prediction Window`处理。
    
- 服务端接收客户端发送的`Prediction Key`。
    
- 服务端将该`Prediction Key`应用到所有相关的`GameplayEffect`。
    
- 服务端同步该`Prediction Key`回客户端。
    
- 客户端使用同步回来的`Prediction Key`应用服务端同步的`GameplayEffect`。如果服务端同步的`GameplayEffect`与客户端使用相同`Prediction Key`应用的`GameplayEffect`匹配，则说明预测正确。在此期间，目标上可能会出现两份`GameplayEffect`，直到客户端移除自己预测的那一份。
    
- 客户端接收回的同步`Prediction Key`会被标记为陈旧（Stale）。
    
- 客户端移除所有由陈旧`Prediction Key`创建的`GameplayEffect`，服务端同步的`GameplayEffect`会持续存在。任何客户端添加但未接收到匹配同步版本的`GameplayEffect`都视为错误预测。
    

在源于`Activation Prediction Key`激活的`GameplayAbility`的一个原子操作组（instruction "window"）期间，`Prediction Key`保证有效。可以理解为`Prediction Key`只在一帧期间有效。任何潜在的`AbilityTask`回调在此之后将不再拥有有效的`Prediction Key`，除非该`AbilityTask`内置了可生成新`Scoped Prediction Window`的同步点。



<a name="concepts-p-windows"></a>
#### 在Ability中创建新的预测窗口

为了在`AbilityTask`的回调函数中预测更多行为，需要使用新的`Scoped Prediction Key`创建`Scoped Prediction Window`。这有时被视为客户端和服务端之间的同步点（Sync Point）。

一些`AbilityTask`（例如所有输入相关的Task）内建了创建新`Scoped Prediction Window`的功能，这意味着回调函数中的原子（Atomic）操作有一个有效的`Scoped Prediction Key`可用。而像`WaitDelay`这样的Task没有内建新`Scoped Prediction Window`用于回调函数，如果需要在这类Task之后进行预测，就必须使用带`OnlyServerWait`选项的`WaitNetSync AbilityTask`手动实现。

当客户端触发带`OnlyServerWait`选项的`WaitNetSync`时，它会生成一个新的基于`GameplayAbility`的`Activation Prediction Key`的`Scoped Prediction Key`，并通过RPC发送到服务端，同时将其添加到所有新应用的`GameplayEffect`。服务端触发该选项时，会在继续执行前等待接收客户端的新`Scoped Prediction Key`，该Key会像`Activation Prediction Key`一样应用到`GameplayEffect`并同步回客户端标记为陈旧（Stale）。`Scoped Prediction Key`在出域前有效，也就表示`Scoped Prediction Window`已经关闭。因此，只有原子操作（Atomic），非延迟（Latent）操作可以使用新的`Scoped Prediction Key`。

可以根据需求创建任意数量的`Scoped Prediction Window`。

如果想为自定义`AbilityTask`添加同步点功能，可以参考输入相关`AbilityTask`如何在内部注入`WaitNetSync AbilityTask`的实现。

**注意:** 使用`WaitNetSync`会阻塞服务端`GameplayAbility`的继续执行，直到接收到客户端消息。这可能被恶意用户滥用以故意延迟发送新的`Scoped Prediction Key`。Epic很少使用`WaitNetSync`，如果担心安全问题，可以创建一个带延迟的新`AbilityTask`，它会自动继续运行而无需等待客户端消息。

样例项目在执行`GameplayAbility`时使用`WaitNetSync`为每次应用耐力花费创建新的`Scoped Prediction Window`，从而实现预测。在应用花费和冷却时，理想状态下需要一个有效的`Prediction Key`。

如果在所属客户端执行了两次预测`GameplayEffect`，则`Prediction Key`会变为陈旧（Stale），出现“redo”问题。通常可以在应用`GameplayEffect`之前，将带`OnlyServerWait`的`WaitNetSync AbilityTask`放在正确位置以创建新的`Scoped Prediction Key`来解决此问题。



<a name="concepts-p-spawn"></a>
#### 预测性地生成Actor

在客户端预测性地生成Actor是一项高级技术。GAS对此没有提供开箱即用的功能（`SpawnActor AbilityTask`只在服务端生成Actor）。其关键是在客户端和服务端都生成同步的Actor。

如果Actor只是用于场景装饰或者不服务于任何游戏逻辑，简单的解决方案就是重写Actor的`IsNetRelevantFor()`函数以限制服务端同步到所属客户端。所属客户端会拥有其本地生成的版本，而服务端和其他客户端会拥有服务端同步的版本。

```cpp
bool APAReplicatedActorExceptOwner::IsNetRelevantFor(const AActor * RealViewer, const AActor * ViewTarget, const FVector & SrcLocation) const
{
    return !IsOwnedBy(ViewTarget);
}
```

如果生成的Actor影响了游戏逻辑，像投掷物就需要预测伤害值，那么你需要本文档范围之外的高级知识。在Epic Games的Github上查看UnrealTournament是如何生成投掷物的，它有一个只在所属客户端生成且与服务端同步的投掷物。 



<a name="concepts-p-future"></a>
**GAS预测的未来发展**

`GameplayPrediction.h`说明了在未来可能会增加预测`GameplayEffect`移除和周期`GameplayEffect`的功能.  




**网络预测插件(Network Prediction plugin)**

Epic最近发起了一项倡议, 将使用新的网络预测插件替换`CharacterMovementComponent`, 该插件仍处于起步阶段, 但是在Unreal Engine Github上已经可以访问了, 现在说未来哪个引擎版本将首次搭载其试验版还为时尚早.  



<a name="concepts-targeting"></a>
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



<a name="concepts-gt-change"></a>
### 常用的Abilty和Effect

<a name="cae-stun"></a>
#### 按钮交互系统 (Button Interaction System)

在 **GASShooter** 中实现了一个按钮交互系统，允许玩家通过 **按下或按住 `E` 键** 与可交互对象进行交互。常见的交互包括：

- 复活其他玩家
    
- 打开武器箱
    
- 控制滑动门的开启或关闭
    

该系统为通用交互提供了基础框架，能够方便地扩展到更多类型的可交互对象。



<a name="debugging"></a>
#### 游戏暂停时生成 TargetData

如果需要在玩家通过 `WaitTargetData AbilityTask` 生成 TargetData 时让游戏“暂停”，推荐使用 **`slomo 0`**（将游戏速度调为 0）而不是直接使用暂停功能。这样可以避免与 GAS 内部的预测和网络同步机制产生冲突。



<a name="cae-onebuttoninteractionsystem"></a>
#### 非堆栈 GameplayEffect，但仅最高强度生效

在某些情况下（例如 Paragon 中的减速效果），`GameplayEffect` 本身是非堆栈的：

- **实例应用**：每个 `GameplayEffect` 实例都会正常应用并跟踪生命周期。
    
- **效果生效**：只有其中 **最高强度 (Greatest Magnitude)** 的效果才会真正影响角色。
    

GAS 针对这种场景提供了现成的支持 —— 可以使用 **`AggregatorEvaluateMetaData`** 来实现


<a name="cae-paused"></a>
#### 暴击 (Critical Hits)

暴击的处理逻辑一般放在伤害的 `ExecutionCalculation` 中：

- **判定条件**：`GameplayEffect` 带有类似 `Effect.CanCrit` 的 `GameplayTag`。在 `ExecutionCalculation` 中检查 `GameplayEffectSpec` 是否包含该标签。
    
- **暴击判定**：如果标签存在，就会根据来源角色捕获的“暴击率”属性生成随机数，并进行判定。
    
- **伤害加成**：若判定成功，则在计算伤害时增加来源角色捕获的“暴击伤害”属性。
    

由于 `ExecutionCalculation` 只在服务端执行，因此这里无需担心客户端与服务端随机数同步的问题。如果使用 `MMC` 在客户端进行预测性伤害计算，则必须从 `GameplayEffectSpec->GameplayEffectContext->GameplayAbilityInstance` 中获取随机数种子的引用来保持一致性。

另外，可以参考 **GASShooter** 对爆头的实现方式：其原理与暴击类似，但不是依赖随机数概率，而是通过检测 `FHitResult` 中的骨骼名称来直接判定是否命中头部。



<a name="cae-nonstackingge"></a>
#### 在客户端和服务端中生成随机数

在 `GameplayAbility` 中经常需要使用随机数来处理诸如枪械后坐力或子弹扩散等效果。为了保证客户端和服务端生成的随机数一致，必须在技能激活时使用相同的随机数种子。由于可能出现客户端错误预测或与服务端随机序列不同步的情况，因此需要在每次激活 `GameplayAbility` 时重新设置随机数种子。

|随机数种子设置方式|描述|
|:-:|:--|
|**使用 `Activation Prediction Key`**|`GameplayAbility` 的 `Activation Prediction Key` 是一个可同步的 int16 值，且在客户端和服务端的 `Activation()` 中都能获取。可以将其作为随机数种子使用。缺点是该 Key 在每场游戏开始时总是从 0 开始并不断递增，因此每场游戏会生成相同的随机数序列。这在部分需求下足够，但并非所有情况都适用。|
|**通过事件负载 (Event Payload) 发送种子**|在通过事件激活 `GameplayAbility` 时，由客户端生成随机种子并通过同步的事件负载传递到服务端，从而保证更高的随机性。但缺点是客户端有可能作弊，每次都发送相同的种子。同时，通过事件激活会使技能无法直接绑定用户输入。|

如果随机偏差较小，大多数玩家不会察觉到随机数序列的重复，此时使用 **`Activation Prediction Key`** 通常足够。  
如果需要更高的安全性，建议采用 **服务端主导的技能激活 (Server Initiated GameplayAbility)**，由服务端生成 `Prediction Key` 或随机数种子，并通过事件负载发送，以防止客户端作弊。



<a name="cae-crit"></a>
#### 生命偷取 (Lifesteal)

生命偷取的逻辑通常放在伤害的 `ExecutionCalculation` 中处理：

- **判定条件**：`GameplayEffect` 带有类似 `Effect.CanLifesteal` 的 `GameplayTag`。在 `ExecutionCalculation` 中检查 `GameplayEffectSpec` 是否包含该标签。
    
- **应用效果**：如果标签存在，则根据伤害值和偷取比例计算生命偷取量，创建一个动态的 `Instant GameplayEffect`，并将其应用回伤害来源的 `AbilitySystemComponent`。
    

示例代码：

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

    SourceAbilitySystemComponent->ApplyGameplayEffectToSelf(
        GELifesteal,
        1.0f,
        SourceAbilitySystemComponent->MakeEffectContext()
    );
}
```



<a name="cae-random"></a>
#### 瞄准 (Aim Down Sight)

样例项目对瞄准的处理方式与奔跑几乎相同，不同之处在于瞄准时会降低而非提升移动速度。

- **移动速度调整**：`CharacterMovementComponent` 会通过预测性处理降低移动速度，具体可参考 `GDCharacterMovementComponent.h/cpp`。
    
- **技能处理**：`GameplayAbility` 负责处理输入，控制瞄准的开始与结束，具体可参考 `GA_AimDownSight_BP`。与奔跑不同，瞄准不会消耗耐力。


<a name="cae-ls"></a>
#### 眩晕 (Stun)

在角色进入眩晕状态时，需要做以下处理：

- **取消所有已激活的技能**：在眩晕 `GameplayTag` 被添加时，调用 `AbilitySystemComponent->CancelAbilities()` 取消目标当前的所有 `GameplayAbility`。
    
- **阻止新的技能激活**：在 `GameplayAbility` 的 `Activation Blocked Tags GameplayTagContainer` 中添加眩晕 `GameplayTag`，从而阻止在眩晕状态下激活新的技能。
    
- **禁止移动**：在角色拥有眩晕 `GameplayTag` 时，重写 `CharacterMovementComponent` 的 `GetMaxSpeed()` 函数，使其返回 0，从而禁止角色移动。
>或者将移动模式改为None



<a name="cae-sprint"></a>
#### 奔跑 (Sprint)

样例项目提供了一个奔跑的实现示例 —— 按住左 Shift 键即可加快移动速度。

- **更快的移动**：由 `CharacterMovementComponent` 通过网络发送标志到服务端进行预测性处理，具体实现可参考 `GDCharacterMovementComponent.h/cpp`。
    
- **技能处理**：`GameplayAbility` 负责响应左 Shift 输入，通知 `CharacterMovementComponent` 开始或停止奔跑，并在按下左 Shift 时进行预测性的耐力消耗，具体可参考 `GA_Sprint_BP`。



<a name="cae-ads"></a>
### 调试 GAS

在调试 GAS 相关问题时，通常需要关心以下内容：

- **属性 (Attribute) 的当前值**
    
- **角色当前持有的 GameplayTag**
    
- **应用在角色身上的 GameplayEffect**
    
- **已授予的 Ability：哪些正在运行，哪些被阻止激活**
    

为此，GAS 提供了两种运行时调试手段：

- 使用 **showdebug abilitysystem**
    
- 在 GameplayDebugger 中挂接 (Hook)
    

**调试技巧**  
UE4 在构建时倾向于对 C++ 代码进行优化，这会使得某些函数在调试时难以追踪或监视变量。如果将 Visual Studio 的解决方案配置设置为 **`DebugGame Editor`** 后仍然无法单步调试，可以使用以下宏来临时关闭特定函数的优化：

```c++
PRAGMA_DISABLE_OPTIMIZATION_ACTUAL
void MyClass::MyFunction(int32 MyIntParameter)
{
    // 调试代码
}
PRAGMA_ENABLE_OPTIMIZATION_ACTUAL
```

注意事项：

- 这种方式对 **inline 函数** 是否生效，取决于其定义方式和位置。
    
- 插件代码中默认不可用，除非是从源码重新编译插件。
    
- 完成调试后务必移除这些宏，以避免影响性能。



<a name="debugging-sd"></a>
#### showdebug abilitysystem

在游戏中打开控制台输入 `showdebug abilitysystem` 即可使用该调试功能。该界面分为三页，每页都会显示当前角色拥有的 `GameplayTag`。可以通过控制台输入 `AbilitySystem.Debug.NextCategory` 来切换页面。

- **第一页**：显示所有 `Attribute` 的 `CurrentValue`。  
    ![[attachments/82bce806c03018ab6f1e6a8d4aa16f0a_MD5.png]]
    
- **第二页**：显示所有应用到角色的持续 (Duration) 和无限 (Infinite) `GameplayEffect`，包括堆栈数、使用的 `GameplayTag` 以及 Modifier 信息。  
    ![[attachments/abc8096d382fe7dafc3a7543689bdaab_MD5.png]]
    
- **第三页**：显示授予给角色的所有 `GameplayAbility`，无论其是否正在运行或被阻止激活，并显示当前正在运行的 `AbilityTask` 状态。  
    ![[attachments/2e7de398871c8533a6d950662579d19a_MD5.png]]
    

**导航功能**：

- 使用 `PageUp` 和 `PageDown` 切换 Target。
    
- 默认仅显示本地控制的角色 `ASC` 数据。
    
- 可通过 `AbilitySystem.Debug.NextTarget` 或 `AbilitySystem.Debug.PrevTarget` 查看其他 `ASC` 的数据，但此时：
    
    - 不会显示调试信息上半部分
        
    - 不会更新绿色目标框，无法知道当前定位的是哪个 `ASC`
        
    - 该问题已提交到 [UE-90437](https://issues.unrealengine.com/issue/UE-90437)
        

**注意事项**：

- `showdebug abilitysystem` 生效前，必须在 GameMode 中选择一个实际的 HUD 类，否则会返回 "Unknown Command"。



<a name="debugging-gd"></a>
#### Gameplay Debugger

GAS 在 **Gameplay Debugger** 中提供了额外功能，用于查看角色的 `GameplayTag`、`GameplayEffect` 和 `GameplayAbility`。

- **开启方法**：
    
    - 使用 **反引号 (`) 键** 打开 Gameplay Debugger。
        
    - 按 **小键盘 3 键** 启用 Ability 分类（具体分类可能因插件不同而有所差异）。
        
    - 如果没有小键盘（如笔记本键盘），可以在 **Project Settings** 中修改键盘绑定。
        
- **功能特点**：
    
    - 可用于查看其他 Character 的 GAS 数据。
        
    - 不能显示目标 Attribute 的 `CurrentValue`。
        
    - 默认监视屏幕中央的 Character，可通过：
        
        - 在 **World Outliner** 中选择目标
            
        - 或看向其他 Character 并重新按下反引号 (`) 键  
            来更改监视的目标。
            
    - 当前监视的 Character 上方会显示一个较大的红色圆圈标识。



<a name="debugging-log"></a>
### GAS 日志 (Logging)

GAS 提供了大量不同详细级别的日志语句，例如 `ABILITY_LOG()`。默认情况下，这些日志的详细级别为 `Display`，更高详细级别的日志不会显示在控制台中。

- **修改日志详细级别**：在控制台输入：
    

```c++
log [category] [verbosity]
```

- **示例**：开启 `ABILITY_LOG()` 的详细输出
    

```c++
log LogAbilitySystem VeryVerbose
```

- **恢复默认级别**
    

```c++
log LogAbilitySystem Display
```

- **显示所有日志分类**
    

```c++
log list
```

GAS相关的日志分类:  

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



<a name="optimizations"></a>
### 优化

<a name="optimizations-abilitybatching"></a>
#### ASC 懒加载

在 Fortnite 大逃杀中，世界中存在大量可破坏的 `AActor`（如树木、建筑物等），每个 `AActor` 都可能拥有一个 `ASC`，直接创建会显著增加内存消耗。

为优化内存，Fortnite 采用 **懒加载 ASC** 的策略：

- **按需创建**：只有当 `AActor` 第一次受到玩家伤害时，才会为其创建 `ASC`。
    
- **优化效果**：未被互动或伤害的 `AActor` 不会占用额外内存，从而降低整体内存消耗。
    

这种方式尤其适用于大型开放世界或拥有大量可破坏物体的游戏场景。



<a name="qol"></a>
#### Attribute 代理同步

在大型多人游戏中（如 Fortnite 大逃杀），大量玩家存在于常相关的 `PlayerState` 上，每个 `PlayerState` 包含一个 `ASC` 和多个 `AttributeSet`，同步这些数据会产生严重的网络瓶颈。

为了优化，Fortnite 采用了以下策略：

- **禁止完全同步到模拟玩家代理**：在 `PlayerState::ReplicateSubobjects()` 中，不再将 `ASC` 和其 `AttributeSet` 完全同步给模拟玩家控制的代理 (Simulated Player-Controlled Proxy)。
    
- **保留完全同步给其他代理**：Autonomous 代理和 AI 控制的 Pawn 仍根据其同步模式完全同步。
    
- **使用 Pawn 上的代理结构体**：
    
    - 玩家 Pawn 上维护可同步的代理结构体，而不是直接同步 PlayerState 上的 `ASC` 属性。
        
    - 当服务端 `ASC` 上的 `Attribute` 变化时，同时更新代理结构体。
        
    - 客户端通过该代理结构体接收同步的 `Attribute`，再将修改返回到本地 `ASC`。
        
    - 利用 Pawn 的 **相关性 (Relevancy)** 和 **NetUpdateFrequency** 控制同步频率。
        
    - 代理结构体使用 **位掩码 (Bitmask)** 同步小规模的 `GameplayTag` 白名单集合。
        

这种优化减少了网络数据量，同时充分利用 Pawn 的相关性进行同步。AI 控制的 Pawn 因其 ASC 已经使用 Pawn 相关性，所以不需要额外优化。

> Epic 的 Dave Ratti 在社区问题#3 中提到，这种方法可能随着其他服务端优化（如 Replication Graph）已经不再必要，并且不是最易维护的方案。  
> [来源](https://epicgames.ent.box.com/s/m1egifkxv3he3u3xezb9hzbgroxyhx89)



<a name="optimizations-asclazyloading"></a>
#### AbilitySystemComponent 同步模式 (Replication Mode)

默认情况下，`ASC` 处于 **Full Replication** 模式，会将所有 `GameplayEffect` 同步到每个客户端，这对单人游戏完全可行。

在多人游戏中，可以根据角色类型调整同步模式：

- **玩家角色**：设置为 **Mixed Replication**，只将应用到玩家角色的 `GE` 同步给该角色的拥有者。
    
- **AI 控制角色**：设置为 **Minimal Replication**，应用到 AI 角色的 `GE` 永远不会同步到客户端。
    

无论使用哪种同步模式：

- `GameplayTag` 仍会进行同步
    
- `GameplayCue` 仍会以不可靠的网络多播 (NetMulticast) 发送到所有客户端
    

这种机制可以在不需要所有客户端查看数据时，显著减少 `GE` 同步产生的网络开销。



<a name="optimizations-attributeproxyreplication"></a>
#### Ability 批处理

在一帧内，`GameplayAbility` 的激活、选择性向服务器发送的 `TargetData` 以及结束操作可以进行 **批处理**，将原本 2-3 个 RPC 压缩为一个 RPC。

这种批处理机制常用于 **枪械命中判定** 等需要高频网络同步的场景，以减少网络开销并提高性能。



<a name="optimizations-gameplaycuebatching"></a>
#### GameplayCue 批处理

当需要同时触发大量 GameplayCue 时，可以将它们 **批处理为单个 RPC**。

- 目标是 **减少 RPC 数量**，因为 `GameplayCue` 是不可靠的网络多播 (NetMulticast)。
    
- 同时尽可能以 **最小的数据量**进行发送，提高网络效率。



<a name="optimizations-ascreplicationmode"></a>
### 代码质量建议

<a name="qol-gameplayeffectcontainers"></a>
#### 使用Gameplay Effect Containers

`GameplayEffectContainer` 将多个元素整合为一个易用结构体，包括：

- `GameplayEffectSpec`
    
- `TargetData`
    
- 简单的定位信息
    
- 其他相关功能
    

这种封装特别适合将 `GameplayEffectSpec` 转移到由 Ability 生成的抛射物上，抛射物在命中碰撞体时再应用这些 `GameplayEffectSpec`，实现灵活的效果传播和伤害处理。



<a name="qol-asynctasksascdelegates"></a>
#### 将蓝图 AsyncTask 绑定到 ASC 委托

为了提升设计师在 UI 设计（如 UMG Widget）中的迭代效率，可以在 C++ 中创建蓝图 AsyncTask，将其直接绑定到 `ASC` 中的常用委托。

**注意事项**：

- 这些 AsyncTask **必须手动销毁**（例如在 Widget 销毁时），否则会一直驻留在内存中。
    
- 样例项目中提供了三个蓝图 AsyncTask 示例：
    

1. **监听 Attribute 修改**  
    ![[attachments/7615f2cbcb71b1ec17fe13f200aa0a7c_MD5.png]]
    
2. **监听冷却时间修改**  
    ![[attachments/e8aa31b6963a4142023590947b5ff46b_MD5.png]]
    
3. **监听 GameplayEffect 堆栈修改**  
    ![[attachments/d0b092d7380721504bc8c8851c258512_MD5.png]]
    

这种方式使得 UI 蓝图可以轻松响应游戏状态变化，无需在 C++ 中手动管理复杂委托逻辑。



<a name="troubleshooting"></a>
### 疑难解答

<a name="troubleshooting-notlocal"></a>
#### 复制的蓝图 Actor 会将 AttributeSet 设置为 nullptr

这是一个 [虚幻引擎的已知 bug](https://issues.unrealengine.com/issue/UE-81109)：  
当使用现有蓝图 Actor 类复制创建新类时，AttributeSet 指针可能被自动设置为 **nullptr**。

**解决方法**

- **不要在类中直接创建自定义 AttributeSet 指针**（头文件中不声明指针，也不在构造函数中调用 `CreateDefaultSubobject`）。
    
- **在 `PostInitializeComponents()` 中向 ASC 添加 AttributeSet**：
    

```c++
void AGDPlayerState::PostInitializeComponents()
{
    Super::PostInitializeComponents();

    if (AbilitySystemComponent)
    {
        AbilitySystemComponent->AddSet<UGDAttributeSetBase>();
        // ... 添加其他 AttributeSet
    }
}
```

- 复制的 AttributeSet 会存在于 ASC 的 `SpawnedAttributes` 数组中。
    

**访问或修改属性值**

由于不直接持有指针，需要通过 **ASC 实例函数** 来获取或修改属性值，而不是使用 AttributeSet 中定义的宏。

```c++
/** 返回当前(最终)属性值 */
float GetNumericAttribute(const FGameplayAttribute &Attribute) const;

/** 设置属性基础值，当前激活的 Modifier 不会被清除 */
void SetNumericAttributeBase(const FGameplayAttribute &Attribute, float NewBaseValue);
```

- **获取生命值示例**
    

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

- **设置(初始化)生命值示例**
    

```c++
const float NewHealth = 100.0f;
if (AbilitySystemComponent)
{
    AbilitySystemComponent->SetNumericAttributeBase(
        UGDAttributeSetBase::GetHealthAttribute(), 
        NewHealth
    );
}
```

**注意**：每个注册到 ASC 的 AttributeSet 类最多只有一个实例对象。

**[⬆ Back to Top](#table-of-contents)**

<a name="acronyms"></a>
#### LogAbilitySystem: Warning: Can't activate LocalOnly or LocalPredicted ability %s when not local!

你需要在客户端初始化ASC.



<a name="troubleshooting-scriptstructcache"></a>
#### 动画蒙太奇不能同步到客户端

确保在GameplayAbility中正在使用的是`PlayMontageAndWait`节点而不是`PlayMontage`, 该AbilityTask可以通过ASC自动同步蒙太奇而PlayMontage节点不可以.



<a name="troubleshooting-duplicatingblueprintactors"></a>
#### `ScriptStructCache`错误

你需要调用`UAbilitySystemGlobals::InitGlobalData()`


### 其他资源

[官方文档](https://docs.unrealengine.com/en-US/InteractiveExperiences/GameplayAbilitySystem/index.html)  

源代码! 特别是`GameplayPrediction.h`.  

[Epic的Action RPG样例项目](https://www.unrealengine.com/marketplace/en-US/slug/action-rpg)  

[来自Epic的Dave Ratti回复社区关于GAS的问题](https://epicgames.ent.box.com/s/m1egifkxv3he3u3xezb9hzbgroxyhx89)  

[Unreal Slackers Discord](https://unrealslackers.org)有一个专注于GAS`#gameplay-abilities-plugin`的文字频道  

[Dan 'Pan'的Github库](https://github.com/Pantong51/GASContent)  

[SabreDartStudios的YouTube视频](https://www.youtube.com/channel/UCCFUhQ6xQyjXDZ_d6X_H_-A)



<a name="changelog"></a>
### GAS更新日志

这是从Unreal Engine官方升级日志和我遇到的未记录升级整理的一份值得参考的更新列表。如果你发现遗漏的内容，可以提交 issue 或 PR。

**[⬆ 返回目录](https://chatgpt.com/c/68a716d4-f59c-8333-9005-8563740f83f7#table-of-contents)**

#### 4.24

- 修复蓝图节点 Attribute 变量在编译时重置为 None 的问题。
    
- 使用 TargetData 前需调用 `UAbilitySystemGlobals::InitGlobalData()`，否则会出现 ScriptStructCache 错误并导致客户端断开。
    
- 修复复制 GameplayTag setter 到未定义变量的蓝图导致崩溃。
    
- UGameplayAbility::MontageStop() 现在正确使用 OverrideBlendOutTime 参数。
    
- 修复组件上 GameplayTag 查询变量在编辑时未修改的问题。
    
- GameplayEffectExecutionCalculation 支持临时变量的 scoped modifier，无需属性捕获即可进行操作。
    
- 优化受限标签操作，添加多个标签时不重置源，提升使用便利性。
    
- APawn::PossessedBy() 现在将 Pawn 的 Owner 设置为新 Controller，适用于混合复制模式。
    
- 修复 FAttributeSetInitterDiscreteLevels 的 POD 问题。
#### 4.25

- 修复 RootMotionSource AbilityTasks 的预测问题。
    
- `GAMEPLAYATTRIBUTE_REPNOTIFY()` 现在可获取旧属性值，需将其作为 OnRep 函数的可选参数传入，避免旧值在复制过程中丢失。
    
- 新增 UGameplayAbility 的 NetSecurityPolicy。
    
- 修复添加 gameplay tag 时未选择有效源导致崩溃。
    
- 修复 Ability System 中攻击者可能导致服务器崩溃的方式。
    
- 修复检查 tag 条件前未确认 GameplayEffect 定义导致的问题。
    
- 修复蓝图函数终止节点中 Gameplay Tag 分类未应用的问题。
    
- 修复 GameplayEffect 标签在多视口中未正确同步的问题。
    
- 修复 InternalTryActivateAbility 循环触发能力时，GameplayAbilitySpec 被无效化的问题。
    
- 修复删除标签时父标签更新延迟导致委托状态不一致的问题。
    
- 修复在确认目标时，遍历 SpawnedTargetActors 前未复制数组导致回调修改数组的问题。
    
- 修复堆叠 SetByCaller Duration 的 GE 持续时间只在首个实例正确应用的问题，并添加自动化测试。
    
- 修复处理 Gameplay Event Delegate 时可能修改 delegate 列表导致的问题。
    
- 修复 GiveAbilityAndActivateOnce 行为不一致的问题。
    
- 调整 FGameplayEffectSpec::Initialize 内部操作顺序，解决潜在依赖问题。
    
- 新增 UGameplayAbility 的 OnRemoveAbility 函数，遵循 OnGiveAbility 模式，仅在主要实例或 CDO 调用。
    
- 调试文本显示被阻止标签时，将显示总数。
    
- 重命名 `UAbilitySystemComponent::InternalServerTryActiveAbility` 为 `InternalServerTryActivateAbility`。
    
- 新增 AbilitySystemComponent 查询函数 `GetActiveEffectsWithAllTags`，可通过代码或蓝图使用。
    
- 根运动相关 AbilityTask 结束时会恢复移动模式。
    
- SpawnedAttributes 设置为 transient，避免保存过时数据，增加空检查防止数据错误传播。
    
- API 变更：`AddDefaultSubobjectSet` 弃用，改用 `AddAttributeSetSubobject`。
    
- Gameplay Abilities 可指定播放蒙太奇的 Anim Instance。
    

#### 4.25.1

- 修复 UE-92787：带 Get Float Attribute 节点且属性引脚为 inline 时保存蓝图崩溃。
    
- 修复 UE-92810：生成 actor 时，如果实例可编辑的 gameplay tag 属性被 inline 修改，会导致崩溃。
    

#### 4.26

- GAS 插件不再标记为测试版。
    
- 修复崩溃：添加 gameplay tag 时若未选择有效 tag 源会导致崩溃。
    
- 修复崩溃：在 `UGameplayCueManager::VerifyNotifyAssetIsInValidPath` 中添加路径字符串参数，防止崩溃。
    
- 修复崩溃：使用 AbilitySystemComponent_Abilities 时未检查指针导致访问冲突。
    
- 修复错误：修复了堆叠 GameplayEffect 时未重置持续时间的问题。
    
- 修复错误：修复 CancelAllAbilities 只能取消非实例化能力的问题。
    
- 新增：GameplayAbility 的提交函数支持可选 tag 参数。
    
- 新增：`PlayMontageAndWait` AbilityTask 增加 StartTimeSeconds 并优化注释。
    
- 新增：FGameplayAbilitySpec 增加 `DynamicAbilityTags` 标签容器，可选标签可随 spec 复制，并作为源标签被应用的 GameplayEffect 捕获。
    
- 新增：GameplayAbility 的 `IsLocallyControlled` 和 `HasAuthority` 函数可在蓝图中调用。
    
- 新增：Visual logger 只在记录数据时收集瞬时 GE 信息。
    
- 新增：蓝图节点的 GameplayAttribute 引脚支持重定向。
    
- 新增：当根运动相关的 AbilityTask 结束时，会将 Movement Component 的移动模式恢复到任务开始前的状态。
    

**更新时间：2021/03/26(完结)**  
翻译此文时间: 2020.12.21  
原文地址: [tranek/GASDocumentation](https://github.com/BillEliot/GASDocumentation)  
翻译地址: [BillEliot/GASDocumentation_Chinese](https://github.com/BillEliot/GASDocumentation_Chinese)  
优化版:[uuz7/GAS-ChineseDocumentationOptimizedVersion: 使用AI对GAS中文文档的优化](https://github.com/uuz7/GAS-ChineseDocumentationOptimizedVersion)


> 最好的文档永远是该插件的代码.  

