
# GAS网络同步与预测机制初探

GAS在GameplayPrediction.h中对其实现的预测系统进行了叙述

在能力本身内部的逻辑中，对预测是无感的，不需要为预测实现任何逻辑。

<details>
<summary>点击展开/折叠 原文</summary>

> At the GameplayAbility level (implementing an ability) prediction is transparent.

</details>

<details>
<summary>点击展开/折叠 译文</summary>

> 在GameplayAbility层面（实现一个能力时），预测是透明的

</details>

预测系统的交互范围是需要服务器参与权威认证的客户端应预测的游戏状态。

<details>
<summary>点击展开/折叠 原文</summary>

> When we say "client side prediction" we really mean client predicting game simulation state. Things can still be 'completely client side' without having to work within a prediction system.

</details>

<details>
<summary>点击展开/折叠 译文</summary>

> 当我们说"客户端预测"时，我们实际上是指客户端预测游戏模拟状态。有些事物可以完全在客户端进行，而无需在预测系统内工作。

</details>

该系统目前实现的范围为

<details>
<summary>点击展开/折叠 原文</summary>

> What do we currently predict?
>
> - Ability activation
> - Triggered Events
> - GameplayEffect application:
> - - Attribute modification (EXCEPTIONS: Executions do not currently predict, only attribute modifiers)
> - - GameplayTag modification
> - Gameplay Cue events (both from within predictive gameplay effect and on their own)
> - Montages
> - Movement (built into UE UCharacterMovement)
>
> Some things we don't predict (most of these we potentially could, but currently dont):
>
> - GameplayEffect removal
> - GameplayEffect periodic effects (dots ticking)

</details>

<details open>
<summary>点击展开/折叠 译文</summary>

> 我们目前预测什么？
>
> - 能力激活
> - 触发事件
> - GameplayEffect应用：
> - - 属性修改（例外：执行（Executions）目前不可预测，仅属性修饰器可预测）
> - - GameplayTag修改
> - GameplayCue事件（包括来自预测性GameplayEffect内部的和独立触发的）
> - 蒙太奇动画
> - 移动（内置在UE的UCharacterMovement中）
>
> 一些我们目前不预测的内容（其中大部分我们未来可能支持，但目前尚未实现）：
>
> - GameplayEffect移除
> - GameplayEffect周期性效果（如持续伤害的 ticking）

</details>

## 1. 预测系统的实现

### 1.1. 总览

GAS的预测基于一个整形标识符预测键（PredictionKey），客户端在激活GameplayAbility时生成，并通过CallServerTryActivateAbility()一起发送到服务器，在预测键有效并且在作用域内时，期间应用的GameplayEffect会被添加该预测键；服务器接收到该预测键时，会将该预测键添加到其应用的GameplayEffect中，同时该预测键会被复制回客户端；如果客户端预测的GameplayEffect与从服务器接收的GameplayEffect具有相同的预测键，则它们是正确预测的；客户端从服务器接收预测键后，该预测键便会被标记为过时；客户端会移除所有添加了过时预测键的GameplayEffect，而由服务器复制的GameplayEffect会被保留；如果被移除的GameplayEffect无匹配的服务器复制版本，则它们是错误预测的。

客户端生成的激活预测键（Activation Prediction Key）只在技能激活瞬间有效，它只负责"激活"这个行为。在技能激活后，来自延迟操作AbilityTasks的任何回调将不再具有有效的预测键，如果需要应用新的GE，应在AbilityTask内置同步点，以新建ScopedPredictionWindow生成新的预测键。

一些AbilityTasks，如所有与输入相关的任务，都内置了创建新的作用域预测窗口的功能。如果需要在没有内置创建作用域预测窗口功能的AbilityTask（如WaitDelay）之后预测动作，应使用带有OnlyServerWait选项的WaitNetSync AbilityTask手动执行此操作。当客户端遇到带有OnlyServerWait的WaitNetSync时，它会基于GameplayAbility的激活预测键生成一个新的作用域预测键。当服务器遇到带有OnlyServerWait的WaitNetSync时，它会等待收到来自客户端的新作用域预测键。此作用域预测键执行与激活预测键相同的操作。

使用WaitNetSync时，会阻止服务器的GameplayAbility继续执行，直到它收到客户端的消息。针对该问题，可以构建一个带有延迟的AbilityTask新版本，如果客户端没有响应，它会自动继续。

### 1.2. 预测键

预测键是预测系统的一个基本概念，它是在客户端生成的一个唯一ID，用于识别在客户端上执行的预测性操作和伴生效果。客户端会将其发送到服务器，并且关联到预测性操作和伴生效果上。服务器会接收或拒绝该预测键，如果接受了，服务器产生的伴生效果也会与其关联。

在预测键从服务器复制回客户端时，它们仅复制给最初将该预测键发送给服务器的那个客户端。

<details>
<summary>点击展开/折叠 原文</summary>

> A fundamental concept in this system is the Prediction Key (FPredictionKey). A prediction key on its own is simply a unique ID that is generated in a central place on the client. The client will send its prediction key to the server, and associate predictive actions and side effects with this key. The server may respond with an accept/reject for the prediction key, and will also associate the server-side created side effects with this prediction key.
>
> (IMPORTANT) FPredictionKey always replicate client -> server, but when replicating server -> clients they *only* replicate to the client that sent the prediction key to the server in the first place.
>
> This happens in FPredictionKey::NetSerialize. All other clients will receive an invalid (0) prediction key when a prediction key sent from a client is replicated back down through a replicated property.

</details>

<details>
<summary>点击展开/折叠 译文</summary>

> 本系统的一个基本概念是预测键（FPredictionKey）。预测键本身只是一个在客户端中心位置生成的唯一ID。客户端会将其预测键发送给服务器，并将预测性操作和副作用与此键关联。服务器可能会接受或拒绝该预测键，并也会将服务器端产生的副作用与此预测键关联。
>
> （重要）FPredictionKey 总是从客户端复制到服务器，但当从服务器复制回客户端时，它们仅复制给最初将该预测键发送给服务器的那个客户端。
>
> 这发生在 FPredictionKey::NetSerialize 中。所有其他客户端在通过复制属性接收到从客户端发送的预测键时，将收到一个无效（0）的预测键。

</details>

### 1.3. 能力激活

当客户端预测性地激活一个能力时，它会明确地向服务器请求（ServerTryActivateAbility），服务器也会明确地回应（ClientActivateAbility(Failed/Succeed)）。在能力被预测性地激活时，会为客户端创建一个预测窗口，在此窗口内发生的预测性伴生效果不会被明确"询问"，自动与生成的PredictionKey关联。

能力预测性激活的客户端和服务器间通信由一个往返构成。

客户端调用 TryActivateAbility，生成一个新的 FPredictionKey 并调用ServerTryActivateAbility。随后客户端在收到服务器回复之前继续执行，并调用ActivateAbility，将生成的 PredictionKey 与能力的 ActivationInfo 关联。在ActivateAbility 调用完成之前发生的任何副作用，都与生成的 FPredictionKey关联。

服务器在 ServerTryActivateAbility 中决定该能力是否真的发生，调用ClientActivateAbility(Failed/Succeed) 并将UAbilitySystemComponent::ReplicatedPredictionKey设置为发送来的生成键。

如果客户端收到ClientAbilityFailed，它会立即终止该能力并回滚与预测键关联的副作用。如果被接受，并且客户端的ReplicatedPredictionKey在属性复制完成后达到先前步骤中使用的键值，客户端就可以撤销其预测性副作用。

<details>
<summary>点击展开/折叠 原文</summary>

> Ability Activation is a first class predictive action. Whenever a client predictively activates an ability, it explicitly asks the server and the server explicitly responds. Once an ability has been predictively activated, the client has a valid 'prediction window' where predictive side effects can happen which are not explicitly 'asked about'. (E.g., we do not explicitly ask 'Can I decrement mana, Can I put this ability on cooldown.' Those actions are considered logically atomic with activating an ability).
>
> AbilitySystemComponent provides a set of functions for communicating ability activation between clients and server: TryActivateAbility -> ServerTryActivateAbility -> ClientActivateAbility(Failed/Succeed).
>
> 1. Client calls TryActivateAbility which generates a new FPredictionKey and calls ServerTryActivateAbility.
> 2. Client continues (before hearing back from server) and calls ActivateAbility with the generated PredictionKey associated with the Ability's ActivationInfo.
> 3. Any side effects that happen /before the call to ActivateAbility finish/ have the generated FPredictionKey associated with them.
> 4. Server decides if the ability really happened in ServerTryActivateAbility, calls ClientActivateAbility(Failed/Succeed) and sets UAbilitySystemComponent::ReplicatedPredictionKey to the generated key that was sent.
> 5. If client receives ClientAbilityFailed, it immediately kills the ability and rolls back side effects that were associated with the prediction key.
>
> - - 'Rolling back' is accomplished via FPredictionKeyDelegates and FPredictionKey::NewRejectedDelegate/NewCaughtUpDelegate/NewRejectOrCaughtUpDelegate.
>    Registering the callback in TryActivateAbility:
>    // If this PredictionKey is rejected, we will call OnClientActivateAbilityFailed.
>    ThisPredictionKey.NewRejectedDelegate().BindUObject(this, &UAbilitySystemComponent::OnClientActivateAbilityFailed, Handle, ThisPredictionKey.Current);
>    Invoking the callback in ClientActivateAbilityFailed_Implementation:
>    FPredictionKeyDelegates::BroadcastRejectedDelegate(PredictionKey);
>
> 6. If accepted, client must wait until property replication catches up (the Succeed RPC will be sent immediately, property replication will happen on its own). Once the ReplicatedPredictionKey catches up to the key used previous steps, the client can undo its predictive side effects. See UAbilitySystemComponent::OnRep_PredictionKey.

</details>

<details>
<summary>点击展开/折叠 译文</summary>

> 能力激活是一类优先级的预测动作。每当客户端预测性地激活一个能力时，它会明确地向服务器请求，服务器也会明确地回应。一旦一个能力被预测性地激活，客户端就拥有了一个有效的"预测窗口"，在此窗口内发生的预测性副作用不会被明确"询问"。（例如，我们不会明确地问"我能减少法力值吗？我能将此能力设置为冷却吗？"这些操作被视为与激活能力在逻辑上原子级的。）
>
> AbilitySystemComponent 提供了一组函数用于在客户端和服务器之间通信能力激活：TryActivateAbility -> ServerTryActivateAbility -> ClientActivateAbility(Failed/Succeed)。
>
> 1. 客户端调用 TryActivateAbility，生成一个新的 FPredictionKey 并调用 ServerTryActivateAbility。
> 2. 客户端在收到服务器回复之前继续执行，并调用 ActivateAbility，将生成的 PredictionKey 与能力的 ActivationInfo 关联。
> 3. 在 ActivateAbility 调用完成之前发生的任何副作用，都与生成的 FPredictionKey 关联。
> 4. 服务器在 ServerTryActivateAbility 中决定该能力是否真的发生，调用 ClientActivateAbility(Failed/Succeed) 并将 UAbilitySystemComponent::ReplicatedPredictionKey 设置为发送来的生成键。
> 5. 如果客户端收到 ClientAbilityFailed，它会立即终止该能力并回滚与预测键关联的副作用。
>
> - - "回滚"通过 FPredictionKeyDelegates 和 FPredictionKey::NewRejectedDelegate/NewCaughtUpDelegate/NewRejectOrCaughtUpDelegate 实现。
>    在 TryActivateAbility 中注册回调：
>    // 如果此 PredictionKey 被拒绝，我们将调用 OnClientActivateAbilityFailed。
>    ThisPredictionKey.NewRejectedDelegate().BindUObject(this, &UAbilitySystemComponent::OnClientActivateAbilityFailed, Handle, ThisPredictionKey.Current);
>    在 ClientActivateAbilityFailed_Implementation 中调用回调：
>    FPredictionKeyDelegates::BroadcastRejectedDelegate(PredictionKey);
>
> 6. 如果被接受，客户端必须等待属性复制赶上（Succeed RPC 会立即发送，属性复制会在其自己的时机发生）。一旦 ReplicatedPredictionKey 达到先前步骤中使用的键值，客户端就可以撤销其预测性副作用。参见 UAbilitySystemComponent::OnRep_PredictionKey。

</details>

### 1.4. GameplayEffect预测

GameplayEffect被视为预测的副作用，不会被明确询问。只有在存在有效预测键的情况下，GameplayEffect才会在客户端上应用。并且如果GameplayEffect是可预测的，则关联的属性、GameplayCue和GameplayTag都会被预测。

当客户端的FActiveGameplayEffect创建时，它会存储预测键。在服务器上，相同的预测键也会被设置在服务器将要复制下来的FActiveGameplayEffect上。

客户端在收到一个带有有效预测键的复制FActiveGameplayEffect时，会检查是否拥有预测键相同的ActiveGameplayEffect。如果匹配，将不执行"应用时"类型的逻辑，同时ActiveGameplayEffects容器中会暂时有两个"相同"的GameplayEffect。在ReplicatedPredictionKey同步到客户端时，匹配的预测性的效果将被移除，ActiveGameplayEffects容器中就只剩下服务器复制的GameplayEffect。

<details>
<summary>点击展开/折叠 原文</summary>

> GameplayEffects are considered side effects of prediction and are not explicitly asked about.
>
> 1. GameplayEffects are only applied on clients if there is a valid prediction key. (If no prediction key, it simply skips the application on client).
> 2. Attributes, GameplayCues, and GameplayTags are all predicted if the GameplayEffect is predicted.
> 3. When the FActiveGameplayEffect is created, it stores the prediction key (FActiveGameplayEffect::PredictionKey)
>    3a. Instant effects are explained below in "Attribute Prediction".
> 4. On the server, the same prediction key is also set on the server's FActiveGameplayEffect that will be replicated down.
> 5. As a client, if you get a replicated FActiveGameplayEffect with a valid prediction key on it, you check to see if you have an ActiveGameplayEffect with that same key, if there is match, we do not apply the 'on applied' type of logic, e.g., GameplayCues. The solves the "Redo" problem. However we will have 2 of the 'same' GameplayEffects in our ActiveGameplayEffects container, temporarily.
> 6. At the same time, UAbilitySystemComponent::ReplicatedPredictionKey will catch up and the predictive effects will be removed. When they are removed in this case, we again check PredictionKey and decide if we should not do the 'On Remove' logic / GameplayCue.

</details>

<details>
<summary>点击展开/折叠 译文</summary>

> GameplayEffect 被视为预测的副作用，不会被明确询问。
>
> 1. 只有在存在有效预测键的情况下，GameplayEffect 才会在客户端上应用。（如果没有预测键，则在客户端上跳过应用）。
> 2. 如果 GameplayEffect 是可预测的，则属性、GameplayCue 和 GameplayTag 都会被预测。
> 3. 当 FActiveGameplayEffect 创建时，它会存储预测键（FActiveGameplayEffect::PredictionKey）。
>    3a. 即时效果在下面的"属性预测"中解释。
> 4. 在服务器上，相同的预测键也会被设置在服务器将要复制下来的 FActiveGameplayEffect 上。
> 5. 作为客户端，如果您收到一个带有有效预测键的复制 FActiveGameplayEffect，您会检查是否拥有相同键的 ActiveGameplayEffect。如果匹配，我们不执行"应用时"类型的逻辑（例如 GameplayCue）。这解决了"重做"问题。但是，我们的 ActiveGameplayEffects 容器中会暂时有两个"相同"的 GameplayEffect。
> 6. 同时，UAbilitySystemComponent::ReplicatedPredictionKey 会赶上，预测性的效果将被移除。当它们在这种情况下被移除时，我们再次检查 PredictionKey 并决定是否不应执行"移除时"逻辑/GameplayCue。

</details>

### 1.5. 属性预测

属性预测的基本策略是将其视为增量预测，而非绝对值预测。

将预测性的即时GameplayEffect视为无限持续时间的GameplayEffect，并且始终接收属性的RepNotify调用（不仅仅是在与上次本地值有变化时）。在属性RepNotify中，调用AbilitySystemComponent::ActiveGameplayEffects根据新的服务器复制值，来更新的施加预测修改后的最终值。当复制预测键被同步时，匹配的预测性的GameplayEffect 被移除，客户端的值恢复到服务器给定的值。

<details>
<summary>点击展开/折叠 原文</summary>

> Since attributes are replicated as standard uproperties, predicting modification to them can be tricky ("Override" problem). Instantaneous modification can be even harder since these are non stateful by nature. (E.g., rolling back an attribute mod is difficult if there is no book keeping past the modification). This makes the "Undo" and "Redo" problem also hard in this case.
>
> The basic plan of attack is to treat attribute prediction as delta prediction rather than absolute value prediction. We do not predict that we have 90 mana, we predict that we have -10 mana from the server value, until the server confirms our prediction key. Basically, treat instant modifications as /infinite duration modifications/ to attributes while they are done predictively. The solves "Undo" and "Redo".
>
> For the "override" problem, we can handle this in the properties OnRep by treating the replicated (server) value as the 'base value' instead of 'final value' of the attribute, and to reaggregate our 'final value' after a replication happens.
>
> 1. We treat predictive instant gameplay effects as infinite duration gameplay effects. See UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf.
> 2. We have to *always* receive RepNotify calls on our attributes (not just when there is a change from last local value, since we will predict the change ahead of time). Done with REPNOTIFY_Always.
> 3. In the attribute RepNotify, we call into the AbilitySystemComponent::ActiveGameplayEffects to update our 'final value' give the new 'base value'. the GAMEPLAYATTRIBUTE_REPNOTIFY can do this.
> 4. Everything else will work like above (GameplayEffect prediction) : when the prediction key is caught up, the predictive GameplayEffect is removed and we will return to the server given value.

</details>

<details>
<summary>点击展开/折叠 译文</summary>

> 由于属性作为标准的 UProperty 进行复制，预测对它们的修改可能比较棘手（"覆盖"问题）。即时修改可能更难处理，因为它们在本质上是无状态的。（例如，如果修改之后没有保留记录，回滚属性修改是很困难的）。这使得"撤销"和"重做"问题在这种情况下也变得很困难。
>
> 基本的策略是将属性预测视为增量预测，而非绝对值预测。我们不是预测我们拥有90点法力值，而是预测我们比服务器值少10点法力值，直到服务器确认我们的预测键。基本上，在预测性修改期间，将即时修改视为对属性的/无限持续时间的修改/。这解决了"撤销"和"重做"问题。
>
> 对于"覆盖"问题，我们可以在属性的 OnRep 中处理，将复制得到的（服务器）值视为属性的"基础值"而非"最终值"，并在复制发生后重新聚合我们的"最终值"。
>
> 1. 我们将预测性的即时 GameplayEffect 视为无限持续时间的 GameplayEffect。参见 UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf。
> 2. 我们必须始终接收属性的 RepNotify 调用（不仅仅是在与上次本地值有变化时，因为我们会提前预测变化）。使用 REPNOTIFY_Always 实现。
> 3. 在属性 RepNotify 中，我们调用 AbilitySystemComponent::ActiveGameplayEffects 来根据新的"基础值"更新我们的"最终值"。GAMEPLAYATTRIBUTE_REPNOTIFY 可以完成此操作。
> 4. 其他所有内容都将像上面（GameplayEffect 预测）一样工作：当预测键被赶上时，预测性的 GameplayEffect 被移除，我们将恢复到服务器给定的值。

</details>

### 1.6. GameplayCue事件

除了通过GameplayEffect关联以外，GameplayCue也可以独立激活。

相关函数会考虑网络角色和预测键，表现如下

<details>
<summary>点击展开/折叠 原文</summary>

> 1. In UAbilitySystemComponent::ExecuteGameplayCue, if authority then do the multicast event (with replication key). If non authority but w/ a valid prediction key, predict the GameplayCue.
> 2. On the receiving end (NetMulticast_InvokeGameplayCueExecuted etc), if there is a replication key, then don't do the event (assume you predicted it).

</details>

<details open>
<summary>点击展开/折叠 译文</summary>

> 1. 在 UAbilitySystemComponent::ExecuteGameplayCue 中，如果是权威端，则执行多播事件（带有复制键）。如果是非权威端但具有有效的预测键，则预测 GameplayCue。
> 2. 在接收端（NetMulticast_InvokeGameplayCueExecuted 等），如果存在复制键，则不执行事件（假设您已经预测了它）。

</details>

### 1.7. 触发数据预测

触发数据当前用于激活能力，其预测与ActivateAbility的代码路径相同。

能力是由游戏代码驱动的事件激活，因此客户端能够预测性地执行这些事件，从而预测性地激活能力。

服务器也会运行触发事件的代码，并保留通过预测性能力激活的触发能力的列表。当从触发能力接收到TryActivate时，服务器会查看是否已经运行了该能力，并用该信息进行响应。

<details>
<summary>点击展开/折叠 原文</summary>

> Triggered Data is currently used to activate abilities. Essentially this all goes through the same code path as ActivateAbility. Rather than the ability being activated from input press, it is activated from another game code driven event. Clients are able to predictively execute these events which predictively activate abilities.
>
> There are some nuances to however, since the server will also run the code that triggers events. The server won't just wait to hear from the client. The server will keep a list of triggered abilities that have been activated from a predictive ability. When receiving a TryActivate from a triggered ability, the server will look to see if /it/ has already run this ability, and respond with that information.

</details>

<details>
<summary>点击展开/折叠 译文</summary>

> 触发数据当前用于激活能力。基本上，这都经过与 ActivateAbility 相同的代码路径。能力不是由输入按键激活，而是由另一个游戏代码驱动的事件激活。客户端能够预测性地执行这些事件，从而预测性地激活能力。
>
> 然而，这有一些细微差别，因为服务器也会运行触发事件的代码。服务器不会只是等待客户端的消息。服务器会保留已从预测性能力激活的触发能力的列表。当从触发能力接收到 TryActivate 时，服务器会查看它是否已经运行了该能力，并用该信息进行响应。

</details>

## 2. 高级主题

### 2.1. 依赖性

GAS通过基础预测键（Base PredictionKey）处理能力连锁触发的依赖链，它是FPredictionKey的一个成员。

链头能力的预测键作为后续连锁能力预测键的基础键，并且后续能力的依赖关系在客户端处理，通过FPredictionKeyDelegates::AddDependancy实现。如果存在能力被拒绝/接受，会添加委托以拒绝/跟进后续能力。

为了简化状态，客户端发送到服务器的预测键只有链头能力的预测键，链条后续由服务器自己基于自身的权威状态重新执行，用最初的基础键来标记整个预测窗口。

<details>
<summary>点击展开/折叠 原文</summary>

> We can have situations such as "Ability X activates and immediately triggers an event which activates Ability Y which triggers another Ability Z". The dependency chain is X->Y->Z.
>
> Each of those abilities could be rejected by the server. If Y is rejected, then Z also never happened, but the server never tries to run Z, so the server doesn't explicitly decide 'no Z can't run'.
>
> To handle this, we have a concept of a Base PredictionKey, which is a member of FPredictionKey. When calling TryActivateAbility, we pass in the current PredictionKey (if applicable). That prediction key is used as the base for any new prediction keys generated. We build a chain of keys this way, and can then invalidate Z if Y is rejected.
>
> This is slightly more nuanced though. In the X->Y->Z case, the server will only receive the PredictionKey for X before trying to run the chain itself. E.g., it will TryActivate Y and Z with the original prediction key sent to it from the client, whereas the client will generate a new PredictionKey each time it calls TryActivateAbility. The client *has* to generate a new PredictionKey for each ability activate, since each activate is not logically atomic. Each side effect produced in the chain of events has to have a unique PredictionKey. We cannot have GameplayEffects produced in X have the same PredictionKey produced in Z.
>
> To get around this, The prediction key of X is considered the Base key for Y and Z. The dependency from Y to Z is kept completely client side, which is done in by FPredictionKeyDelegates::AddDependancy. We add delegates to reject/catchup Z if Y rejected/confirmed.
>
> This dependency system allows us to have multiple predictive actions that are not logically atomic within a single prediction window/scope.

</details>

<details>
<summary>点击展开/折叠 译文</summary>

> 我们可能会遇到这样的情况："能力 X 激活并立即触发一个事件，该事件激活能力 Y，Y 又触发另一个能力 Z"。依赖链是 X->Y->Z。
>
> 这些能力中的每一个都可能被服务器拒绝。如果 Y 被拒绝，那么 Z 也从未发生，但服务器从未尝试运行 Z，因此服务器没有明确决定"不，Z 不能运行"。
>
> 为了处理这种情况，我们有一个基础预测键（Base PredictionKey）的概念，它是 FPredictionKey 的一个成员。在调用 TryActivateAbility 时，我们传入当前的 PredictionKey（如果适用）。该预测键被用作任何新生成的预测键的基础。我们通过这种方式构建一个键链，并且如果 Y 被拒绝，可以使 Z 无效。
>
> 不过，这稍微有些微妙。在 X->Y->Z 的情况下，服务器在尝试运行链本身之前只会收到 X 的 PredictionKey。例如，它将使用从客户端发送给它的原始预测键来 TryActivate Y 和 Z，而客户端每次调用 TryActivateAbility 时都会生成一个新的 PredictionKey。客户端必须为每次能力激活生成一个新的 PredictionKey，因为每次激活在逻辑上不是原子的。在事件链中产生的每个副作用都必须有一个唯一的 PredictionKey。我们不能让在 X 中产生的 GameplayEffect 与在 Z 中产生的 GameplayEffect 具有相同的 PredictionKey。
>
> 为了解决这个问题，X 的预测键被视为 Y 和 Z 的基础键。从 Y 到 Z 的依赖关系完全保持在客户端，这通过 FPredictionKeyDelegates::AddDependancy 完成。如果 Y 被拒绝/确认，我们添加委托来拒绝/赶上 Z。
>
> 这种依赖系统允许我们在一个单一的预测窗口/范围内拥有多个在逻辑上并非原子的预测性操作。

</details>

### 2.2. 能力内的额外预测窗口

FScopedPredictionWindow提供了一种方法，可以向服务器发送一个新的预测键，并让服务器在相同的逻辑范围内接收并使用该键。

例子UAbilityTask_WaitInputRelease::OnReleaseCallback的事件流程为

<details>
<summary>点击展开/折叠 原文</summary>

> 1. Client enters UAbilityTask_WaitInputRelease::OnReleaseCallback and starts a new FScopedPredictionWindow. This creates a new prediction key for this scope (FScopedPredictionWindow::ScopedPredictionKey).
> 2. Client calls AbilitySystemComponent->ServerInputRelease which passes ScopedPrediction.ScopedPredictionKey as a parameter.
> 3. Server runs ServerInputRelease_Implementation which takes the passed in PredictionKey and sets it as UAbilitySystemComponent::ScopedPredictionKey with an FScopedPredictionWindow.
> 4. Server runs UAbilityTask_WaitInputRelease::OnReleaseCallback /within the same scope/
> 5. When the server hits the FScopedPredictionWindow in ::OnReleaseCallback, it gets the prediction key from UAbilitySystemComponent::ScopedPredictionKey. That is now used for all side effects within this logical scope.
> 6. Once the server ends this scoped prediction window, the prediction key used is finished and set to ReplicatedPredictionKey.
> 7. All side effects created in this scope now share a key between client and server.
>
> The key to this working is that ::OnReleaseCallback calls ::ServerInputRelease which calls ::OnReleaseCallback on the server. There is no room for anything else to happen and use the given prediction key.
>
> While there is no "Try/Failed/Succeed" calls in this example, all side effects are procedurally grouped/atomic. This solves the "Undo" and "Redo" problems for any arbitrary function calls that run on the server and client.

</details>

<details open>
<summary>点击展开/折叠 译文</summary>

> 1. 客户端进入 UAbilityTask_WaitInputRelease::OnReleaseCallback 并启动一个新的 FScopedPredictionWindow。这会为此范围创建一个新的预测键（FScopedPredictionWindow::ScopedPredictionKey）。
> 2. 客户端调用 AbilitySystemComponent->ServerInputRelease，并将 ScopedPrediction.ScopedPredictionKey 作为参数传递。
> 3. 服务器运行 ServerInputRelease_Implementation，它接收传入的 PredictionKey 并使用 FScopedPredictionWindow 将其设置为 UAbilitySystemComponent::ScopedPredictionKey。
> 4. 服务器 /在同一范围内/ 运行 UAbilityTask_WaitInputRelease::OnReleaseCallback。
> 5. 当服务器在 ::OnReleaseCallback 中遇到 FScopedPredictionWindow 时，它会从 UAbilitySystemComponent::ScopedPredictionKey 获取预测键。该键现在用于此逻辑范围内的所有副作用。
> 6. 一旦服务器结束此范围预测窗口，所使用的预测键即告完成，并被设置为 ReplicatedPredictionKey。
> 7. 在此范围内创建的所有副作用现在在客户端和服务器之间共享一个键。
>
> 此功能工作的关键在于 ::OnReleaseCallback 调用 ::ServerInputRelease，而后者又在服务器上调用 ::OnReleaseCallback。没有其他任何东西可以介入并使用给定的预测键。
>
> 虽然在此示例中没有"尝试/失败/成功"调用，但所有副作用在程序上都是分组的/原子级的。这解决了在服务器和客户端上运行的任何任意函数调用的"撤销"和"重做"问题。

</details>

## 3. 不支持/问题/待办事项

触发事件不会显式复制。如果触发事件仅在服务器上运行，客户端将永远不会知道它。

<details>
<summary>点击展开/折叠 原文</summary>

> Triggered events do not explicitly replicate. E.g., if a triggered event only runs on the server, the client will never hear about it. This also prevents us from doing cross player/AI etc events. Support for this should eventually be added and it should follow the same pattern that GameplayEffect and GameplayCues follow (predict triggered event with a prediction key, ignore the RPC event if it has a prediction key).

</details>

<details>
<summary>点击展开/折叠 译文</summary>

> 触发事件不会显式复制。例如，如果触发事件仅在服务器上运行，客户端将永远不会知道它。这也阻止了我们进行跨玩家/AI 等事件。最终应该添加对此的支持，并且它应该遵循 GameplayEffect 和 GameplayCue 相同的模式（使用预测键预测触发事件，如果 RPC 事件带有预测键则忽略它）。

</details>

### 3.1. 预测"元"属性（如伤害/治疗）与"真实"属性（如生命值）

无法预测性地应用元属性，元属性仅对即时效果有效。

<details>
<summary>点击展开/折叠 原文</summary>

> We are unable to apply meta attributes predictively. Meta attributes only work on instant effects, in the back end of GameplayEffect (Pre/Post Modify Attribute on the UAttributeSet). These events are not called when applying duration-based gameplay effects. E.g., a GameplayEffect that modifies damage for 5 seconds doesn't make sense.
>
> In order to support this, we would probably add some limited support for duration based meta attributes, and move the transform of the instant gameplay effect from the front end (UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf) to the backend (UAttributeSet::PostModifyAttribute).

</details>

<details>
<summary>点击展开/折叠 译文</summary>

> 我们无法预测性地应用元属性。元属性仅对即时效果有效，在 GameplayEffect 的后端（UAttributeSet 上的 Pre/Post Modify Attribute）中起作用。这些事件在应用基于持续时间的 GameplayEffect 时不会被调用。例如，一个修改伤害持续 5 秒的 GameplayEffect 是没有意义的。
>
> 为了支持这一点，我们可能会添加对基于持续时间的元属性的有限支持，并将即时 GameplayEffect 的转换从前端（UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf）移动到后端（UAttributeSet::PostModifyAttribute）。

</details>

### 3.2. 预测持续性的乘法型 GameplayEffect

在预测基于百分比的 GameplayEffect 时也存在限制。

服务器复制的是属性的"最终值"，而不是完整的聚合器链（所有修改它的内容），客户端无法准确预测新GameplayEffect 的情况。这将需要通过复制属性的聚合器链来解决。

<details>
<summary>点击展开/折叠 原文</summary>

> There are also limitations when predicting % based gameplay effects. Since the server replicates down the 'final value' of an attribute, but not the entire aggregator chain of what is modifying it, we may run into cases where the client cannot accurately predict new gameplay effects.
>
> For example:
>
> - Client has a perm +10% movement speed buff with base movement speed of 500 -> 550 is the final movement speed for this client.
>
> - Client has an ability which grants an additional 10% movement speed buff. It is expected to *sum* the % based multipliers for a final 20% bonus to 500 -> 600 movement speed.
>
> - However on the client, we just apply a 10% buff to 550 -> 605.
>
> This will need to be fixed by replicating down the aggregator chain for attributes. We already replicate some of this data, but not the full modifier list. We will need to look into supporting this eventually.

</details>

<details>
<summary>点击展开/折叠 译文</summary>

> 在预测基于百分比的 GameplayEffect 时也存在限制。由于服务器复制的是属性的"最终值"，而不是完整的聚合器链（所有修改它的内容），我们可能会遇到客户端无法准确预测新 GameplayEffect 的情况。
>
> 例如：
>
> - 客户端有一个永久 +10% 移动速度增益，基础移动速度为 500 -> 该客户端的最终移动速度为 550。
>
> - 客户端有一个能力，可以额外提供 10% 的移动速度增益。预期是将百分比乘数相加，得到最终 20% 的加成，即 500 -> 600 移动速度。
>
> - 然而，在客户端，我们只是将 10% 的增益应用于 550 -> 605。
>
> 这将需要通过复制属性的聚合器链来解决。我们已经复制了其中的一些数据，但不是完整的修饰符列表。我们需要考虑在未来支持这一点。

</details>

### 3.3. "弱预测"

为了解决不完全适合此系统的情况，GAS设想了弱预测模式。当没有可以准确关联副作用的新的预测键时，某些能力（所有能力？）会回退到该模式。在弱预测模式下，只有某些操作可以被预测。

<details>
<summary>点击展开/折叠 原文</summary>

> We will probably still have cases that do not fit well into this system. Some situations will exist where a prediction key exchange is not feasible. For example, an ability where any one that player collides with/touches receives a GameplayEffect that slows them and their material blue. Since we can't send Server RPCs every time this happens (and the server couldn't necessarily handle the message at its point in the simulation), there is no way to correlate the gameplay effect side effects between client and server.
>
> One approach here may be to think about a weaker form of prediction. One where there is not a fresh prediction key used and instead the server assumes the client will predict all side effects from an entire ability. This would at least solve the "redo" problem but would not solve the "completeness" problem. If the client side prediction could be made as minimal as possible - for example only predicting an initial particle effect rather than predicting the state and attribute change - then the problems get less severe.
>
> I can envision a weak prediction mode which is what (certain abilities? All abilities?) fall back to when there is no fresh prediction key that can accurately correlate side effects. When in weak prediction mode, perhaps only certain actions can be predicted - for example GameplayCue execute events, but not OnAdded/OnRemove events.

</details>

<details>
<summary>点击展开/折叠 译文</summary>

> 我们可能仍然会遇到不完全适合此系统的情况。在某些情况下，预测键交换可能不可行。例如，一个能力，任何与该玩家碰撞/接触的玩家都会收到一个减慢其速度并改变其材质的 GameplayEffect。由于我们无法在每次发生时都发送 Server RPC（并且服务器在其模拟时间点也不一定能够处理该消息），因此无法关联客户端和服务器之间的 GameplayEffect 副作用。
>
> 这里的一种方法可能是考虑一种较弱形式的预测。这种预测不使用全新的预测键，而是服务器假定客户端将预测整个能力的所有副作用。这至少可以解决"重做"问题，但不能解决"完整性"问题。如果客户端的预测可以做得尽可能少——例如仅预测一个初始粒子效果，而不是预测状态和属性变化——那么问题就不会那么严重。
>
> 我可以设想一种弱预测模式，当没有可以准确关联副作用的新的预测键时，某些能力（所有能力？）会回退到该模式。在弱预测模式下，也许只有某些操作可以被预测——例如 GameplayCue 执行事件，但不包括 OnAdded/OnRemove 事件。

</details>

## 4. 代码细节

GAS预测系统主要由GameplayPrediction和UAbilitySystemComponent两部分构成。前者实现了预测的基础设施，包括FPredictionKey（预测键的生成、序列化与有效性判断）、FPredictionKeyDelegates（全局委托管理与依赖链）、FScopedPredictionWindow（RAII作用域，将后续操作绑定到特定预测键）以及FReplicatedPredictionKeyMap（基于FastArray的预测键同步机制）。它定义了"预测如何工作"但不涉及具体业务。

后者作为GAS的中枢，将这套机制注入到所有实际游戏逻辑中。能力激活的预测握手（TryActivateAbility/ServerTryActivateAbility）、GameplayEffect应用时的预测键关联与回滚（ApplyGameplayEffectSpecToSelf）、蒙太奇播放时的预测绑定（PlayMontage）、属性同步时的增量预测校正（OnRep与AttributeAggregator），以及预测键复制到达后的"确认/回滚"触发（OnRep_PredictionKey）。可以概括为GameplayPrediction提供工具箱，UAbilitySystemComponent决定何时何地使用这些工具。

### 4.1. GameplayPrediction

#### 4.1.1. FPredictionKey

预测键在UE5.0后废弃了原先使用的公开的PredictiveConnection，转为通过使用私有的PredictiveConnectionKey存储预测键的客户端来源，并且提供公开的GetPredictiveConnectionKey用以访问PredictiveConnectionKey。因为旧方式直接持有对象引用（容易导致悬空指针或生命周期问题），新方式改用 Key（键/标识符）来间接引用，更安全。

<details>
<summary>点击展开/折叠 代码：FPredictionKey 成员变量（UE5.0 后）</summary>

```cpp
/** On the server, what network connection this was serialized on. */
UE_DEPRECATED(5.0, "No longer used in favor of GetPredictiveConnectionKey, to avoid holding a direct object reference.")
UPROPERTY(NotReplicated)
TObjectPtr<UPackageMap> PredictiveConnection = nullptr;
UPTRINT GetPredictiveConnectionKey() const { return PredictiveConnectionKey; }
/** On the server, uniquely identifies network connection this was serialized on/from.  See NetSerialize for additional information. */
UPTRINT PredictiveConnectionKey = 0;
```

</details>

因此，在UE5.0后，预测键结构体内的成员变量为公开的Current、Base、bIsStale、bIsServerInitiated和私有的PredictiveConnectionKey，其中bIsStale是不复制的（NotReplicated），仅在客户端标记预测键的有效性。

<details>
<summary>点击展开/折叠 代码：FPredictionKey 结构体成员</summary>

```cpp
/** The unique ID of this prediction key */
UPROPERTY()
int16	Current;
/** If non 0, the prediction key this was created from */
UPROPERTY()
int16	Base;
/** If stale, this key cannot be used for more prediction */
UPROPERTY(NotReplicated)
bool bIsStale;
/** True if this was created as a server initiated activation key, used to identify server activations but cannot be used for prediction */
UPROPERTY()
bool bIsServerInitiated;
/** On the server, uniquely identifies network connection this was serialized on/from.  See NetSerialize for additional information. */
UPTRINT PredictiveConnectionKey = 0;
```

</details>

工具成员函数主要有3种类型

1.创建预测键的静态工厂函数。预测键结构体提供了公开的CreateNewPredictionKey和CreateNewServerInitiatedKey以创建预测键，在工厂函数内部调用GenerateNewPredictionKey，通过静态变量Gkey的自增为Current赋值，并将bIsStale和bIsServerInitiated标记为与工厂函数对应的值。

<details>
<summary>点击展开/折叠 代码：创建预测键的工厂函数</summary>

```cpp
/** Construct a new prediction key with no dependencies */
FPredictionKey FPredictionKey::CreateNewPredictionKey(UAbilitySystemComponent* OwningComponent)
{
	FPredictionKey NewKey;
	// We should never generate prediction keys on the authority
	if(OwningComponent->GetOwnerRole() != ROLE_Authority)
	{
		NewKey.GenerateNewPredictionKey();
	}
	return NewKey;
}
/** Construct a new server initiation key, for abilities activated on the server */
FPredictionKey FPredictionKey::CreateNewServerInitiatedKey(UAbilitySystemComponent* OwningComponent)
{
	FPredictionKey NewKey;

	// Only valid on the server
	if (OwningComponent->GetOwnerRole() == ROLE_Authority)
	{
		NewKey.GenerateNewPredictionKey();
		NewKey.bIsServerInitiated = true;
	}
	return NewKey;
}
void FPredictionKey::GenerateNewPredictionKey()
{
	static KeyType GKey = 1;
	Current = GKey++;
	if (GKey < 0)
	{
		GKey = 1;
	}
	bIsStale = false;
}
```

</details>

2.预测键的依赖关系处理。预测键结构体提供了公开的GenerateDependentPredictionKey、NewRejectedDelegate、NewCaughtUpDelegate和NewRejectOrCaughtUpDelegate以调用预测键委托结构体（FPredictionKeyDelegates）的相关静态函数，处理预测键间的依赖关系。

<details>
<summary>点击展开/折叠 代码：依赖关系处理函数</summary>

```cpp
void FPredictionKey::GenerateDependentPredictionKey()
{
	if (bIsServerInitiated)
	{
		// Can't have dependent keys on server keys, use same key
		return;
	}
	KeyType Previous = 0;
	if (Base == 0)
	{
		Base = Current;
	}
	else
	{
		Previous = Current;
	}
	GenerateNewPredictionKey();
	if (Previous > 0)
	{
		FPredictionKeyDelegates::AddDependency(Current, Previous);
	}
}
FPredictionKeyEvent& FPredictionKey::NewRejectedDelegate()
{
	return FPredictionKeyDelegates::NewRejectedDelegate(Current);
}
FPredictionKeyEvent& FPredictionKey::NewCaughtUpDelegate()
{
	return FPredictionKeyDelegates::NewCaughtUpDelegate(Current);
}
void FPredictionKey::NewRejectOrCaughtUpDelegate(FPredictionKeyEvent Event)
{
	FPredictionKeyDelegates::NewRejectOrCaughtUpDelegate(Current, Event);
}
```

</details>

3.条件判断函数。预测键结构体提供了许多用于快速进行条件判断的内联函数。

<details>
<summary>点击展开/折叠 代码：条件判断函数</summary>

```cpp
/** A key is valid if it's non-zero, prediction keys for other clients will serialize down as 0 and be invalid */
bool IsValidKey() const
{
	return Current > 0;
}
/** A key was generated by the local client if it's valid and not a server key, prediction keys for other clients will serialize down as 0 and be invalid */
bool IsLocalClientKey() const
{
	return Current > 0 && !bIsServerInitiated;
}
/** True if this was created as a server initiated activation key, used to identify server activations but cannot be used for prediction */
bool IsServerInitiatedKey() const
{
	return bIsServerInitiated;
}
/** Can this key be used for more predictive actions, or has it already been sent off to the server? */
bool IsValidForMorePrediction() const
{
	return Current > 0 && bIsStale == false && bIsServerInitiated == false;
}
/** Was this PredictionKey received from a NetSerialize or created locally? */
bool WasReceived() const
{
	return PredictiveConnectionKey != 0;
}
bool WasLocallyGenerated() const
{
	return (Current > 0) && (PredictiveConnectionKey == 0);
}
bool DependsOn(KeyType Key)
{
	return (Current == Key || Base == Key);
}
```

</details>

在预测键结构体内，仅公开了一个无参构造函数供UE的反射系统使用，为"正确创建逻辑"服务的两个带参构造函数均为私有，有效预测键的创建只能通过提供的静态工厂函数。确保了预测键的生命周期/语义正确。

<details>
<summary>点击展开/折叠 代码：FPredictionKey 构造函数</summary>

```cpp
FPredictionKey()
: Current(0), Base(0), bIsStale(false), bIsServerInitiated(false)
{
}
FPredictionKey(int32 Key)
	: Current(Key), Base(0), bIsStale(false), bIsServerInitiated(false)
{
}
FPredictionKey(int16 InKey, int16 PreviousKey)
	: Current(InKey), Base(PreviousKey), bIsStale(false), bIsServerInitiated(false)
{
}
```

</details>

为了使得预测键仅复制回发送它的客户端，GAS通过特化TStructOpsTypeTraits模板，开启了网络序列化器，并且在预测键的网络序列化函数（NetSerialize）中实现了预测密钥的归属权追踪。

此外，在特化模板时，还开启了WithIdenticalViaEquality，让引擎在判断这个类型的相等性时直接调用它的==运算符，而不是默认的比较整个对象占用的所有内存字节。从而避免因内存填充字节或无关成员导致的相等性误判，确保网络复制脏标记的准确性。

<details>
<summary>点击展开/折叠 代码：TStructOpsTypeTraits 特化与相等运算符</summary>

```cpp
template<>
struct TStructOpsTypeTraits<FPredictionKey> : public TStructOpsTypeTraitsBase2<FPredictionKey>
{
	enum
	{
		WithNetSerializer = true,
		WithIdenticalViaEquality = true
	};
};
bool operator==(const FPredictionKey& Other) const
{
	return Current == Other.Current && Base == Other.Base && bIsServerInitiated == Other.bIsServerInitiated;
}
```

</details>

预测键的网络序列化函数分为5步流程

1.判定该预测键是否有效。当该预测键满足以下任一条件（没有绑定特定连接（PredictiveConnectionKey == 0）、当前连接就是当初提供者（(UPTRINT)Map == PredictiveConnectionKey）、或是服务器主动发起（bIsServerInitiated）），并且ID有效时（Current > 0），预测键有效。发送方会将该判定结果赋值给ValidKeyForConnection并写入序列（有效为1，无效为0）；接收方则是读取序列值赋给ValidKeyForConnection，以供后续流程使用。

2.判定预测键是否存在基础键。预测键有效时进入该流程，当预测键的基础键ID有效（Base > 0）时，存在基础键。发送方会将该判定结果赋值给HasBaseKey并写入序列（存在为1，不存在为0）；接收方则是读取序列值赋给HasBaseKey，以供后续流程使用。

3.判定预测键是否为服务器创建。发送方将是否服务器主动发起的预测键（bIsServerInitiated）写入序列；接收方从序列读取是否服务器主动发起的预测键赋值给bIsServerInitiated。

4.处理预测键值。预测键有效时进入该流程，发送方将预测键ID写入序列，接收方读取序列值赋给预测键ID。当其基础键存在时（HasBaseKey），发送方将其基础键ID（Base）写入序列，接收方读取序列值赋给基础键ID。

5.接收时将发送者网络信息（class UPackageMap* Map）保存。接收方读取序列，如果不是服务器主动发起的预测键（!bIsServerInitiated），将网络信息赋值给PredictiveConnectionKey，记录该键的来源连接。

<details>
<summary>点击展开/折叠 代码：FPredictionKey::NetSerialize</summary>

```cpp
/** The key to understanding this function is that when a key is received by the server, we note which connection gave it to us. We only serialize the key back to that client.  */
bool FPredictionKey::NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess)
{
	// First bit for valid key for this connection or not. (most keys are not valid)
	uint8 ValidKeyForConnection = 0;
	if (Ar.IsSaving())
	{
		/**
		 *	Only serialize the payload if we have no owning connection (Client sending to server)
		 *	or if the owning connection is this connection (Server only sends the prediction key to the client who gave it to us)
		 *  or if this is a server initiated key (valid on all connections)
		 */		
		ValidKeyForConnection = (PredictiveConnectionKey == 0 || ((UPTRINT)Map == PredictiveConnectionKey) || bIsServerInitiated) && (Current > 0);
	}
	Ar.SerializeBits(&ValidKeyForConnection, 1);
	// Second bit for base key (only if valid connection)
	uint8 HasBaseKey = 0;
	if (ValidKeyForConnection)
	{
		if (Ar.IsSaving())
		{
			HasBaseKey = Base > 0;
		}
		Ar.SerializeBits(&HasBaseKey, 1);
	}
	// Third bit for server initiated
	uint8 ServerInitiatedByte = bIsServerInitiated;
	Ar.SerializeBits(&ServerInitiatedByte, 1);
	bIsServerInitiated = ServerInitiatedByte & 1;
	// Conditionally Serialize the Current and Base keys
	if (ValidKeyForConnection)
	{
		Ar << Current;
		if (HasBaseKey)
		{
			Ar << Base;
		}
	}	
	if (Ar.IsLoading())
	{
		// We are reading this key: the connection that gave us this key is the predictive connection, and we will only serialize this key back to it.
		if (!bIsServerInitiated)
		{
			PredictiveConnectionKey = (UPTRINT)Map;
		}
	}
	bOutSuccess = true;
	return true;
}
```

</details>

#### 4.1.2. FPredictionKeyDelegates

预测键委托通过静态Get函数作为全局访问点实现了单例模式，在静态实例中维护了全局的预测键委托映射（DelegateMap）。

<details>
<summary>点击展开/折叠 代码：FPredictionKeyDelegates 单例与委托映射</summary>

```cpp
FPredictionKeyDelegates& FPredictionKeyDelegates::Get()
{
	static FPredictionKeyDelegates StaticMap;
	return StaticMap;
}
struct FDelegates
{
public:
	/** This delegate is called if the prediction key is associated with an action that is explicitly rejected by the server. */
	TArray<FPredictionKeyEvent>	RejectedDelegates;
	/** This delegate is called when replicated state has caught up with the prediction key. Doesnt imply rejection or acceptance. */
	TArray<FPredictionKeyEvent>	CaughtUpDelegates;
};
TMap<FPredictionKey::KeyType, FDelegates>	DelegateMap;
static FPredictionKeyDelegates& Get();
```

</details>

预测键委托提供了NewRejectedDelegate与NewCaughtUpDelegate为指定的预测键创建对应委托列表，并返回临时生成的预测键事件。并且可使用NewRejectOrCaughtUpDelegate为指定预测键的委托增添新事件。

<details>
<summary>点击展开/折叠 代码：委托创建函数</summary>

```cpp
FPredictionKeyEvent& FPredictionKeyDelegates::NewRejectedDelegate(FPredictionKey::KeyType Key)
{
	TArray<FPredictionKeyEvent>& DelegateList = Get().DelegateMap.FindOrAdd(Key).RejectedDelegates;
	DelegateList.Add(FPredictionKeyEvent());
	return DelegateList.Top();
}
FPredictionKeyEvent& FPredictionKeyDelegates::NewCaughtUpDelegate(FPredictionKey::KeyType Key)
{
	TArray<FPredictionKeyEvent>& DelegateList = Get().DelegateMap.FindOrAdd(Key).CaughtUpDelegates;
	DelegateList.Add(FPredictionKeyEvent());
	return DelegateList.Top();
}
void FPredictionKeyDelegates::NewRejectOrCaughtUpDelegate(FPredictionKey::KeyType Key, FPredictionKeyEvent NewEvent)
{
	FDelegates& Delegates = Get().DelegateMap.FindOrAdd(Key);
	Delegates.CaughtUpDelegates.Add(NewEvent);
	Delegates.RejectedDelegates.Add(NewEvent);
}
```

</details>

为了处理不同的生命周期语义，预测键委托实现了两套执行函数。BroadcastRejectedDelegate和BroadcastCaughtUpDelegate用于状态同步通知，只触发回调，不删除 Key。为了避免遍历时因委托回调修改容器导致迭代器失效，采用拷贝快照的方式遍历；同时利用static局部变量复用内存，减少动态分配开销。Reject 和 CatchUpTo 则用于生命周期终结，在触发回调后立即删除 Key，执行一次性清理。

<details>
<summary>点击展开/折叠 代码：委托广播与清理函数</summary>

```cpp
void FPredictionKeyDelegates::BroadcastRejectedDelegate(FPredictionKey::KeyType Key)
{
	// Intentionally making a copy of the delegate list since it may change when firing one of the delegates
	static TArray<FPredictionKeyEvent> DelegateList;
	DelegateList.Reset();
	DelegateList = Get().DelegateMap.FindOrAdd(Key).RejectedDelegates;
	for (auto& Delegate : DelegateList)
	{
		Delegate.ExecuteIfBound();
	}
}
void FPredictionKeyDelegates::BroadcastCaughtUpDelegate(FPredictionKey::KeyType Key)
{
	// Intentionally making a copy of the delegate list since it may change when firing one of the delegates
	static TArray<FPredictionKeyEvent> DelegateList;
	DelegateList.Reset();
	DelegateList = Get().DelegateMap.FindOrAdd(Key).CaughtUpDelegates;
	for (auto& Delegate : DelegateList)
	{
		Delegate.ExecuteIfBound();
	}
}
void FPredictionKeyDelegates::Reject(FPredictionKey::KeyType Key)
{
	FDelegates* DelPtr = Get().DelegateMap.Find(Key);
	if (DelPtr)
	{
		for (auto& Delegate : DelPtr->RejectedDelegates)
		{
			Delegate.ExecuteIfBound();
		}
		Get().DelegateMap.Remove(Key);
	}
}
void FPredictionKeyDelegates::CatchUpTo(FPredictionKey::KeyType Key)
{
	FDelegates* DelPtr = Get().DelegateMap.Find(Key);
	if (DelPtr)
	{
		for (auto& Delegate : DelPtr->CaughtUpDelegates)
		{
			Delegate.ExecuteIfBound();
		}
		Get().DelegateMap.Remove(Key);
	}
}
```

</details>

在预测键存在依赖关系时，便可通过AddDependency函数，将预测键的Reject和CatchUpTo绑定到基础键的委托列表上。当基础键的委托广播时，回调触发预测键的Reject/CatchUpTo。

<details>
<summary>点击展开/折叠 代码：AddDependency</summary>

```cpp
void FPredictionKeyDelegates::AddDependency(FPredictionKey::KeyType ThisKey, FPredictionKey::KeyType DependsOn)
{
	NewRejectedDelegate(DependsOn).BindStatic(&FPredictionKeyDelegates::Reject, ThisKey);
	NewCaughtUpDelegate(DependsOn).BindStatic(&FPredictionKeyDelegates::CatchUpTo, ThisKey);
}
```

</details>

#### 4.1.3. FScopedPredictionWindow

范围预测窗口内仅有构造函数、析构函数和记录状态的变量，通过RAII（资源获取即初始化）发挥作用。当在代码中创建这个对象时，会调用构造函数，记录当前状态，并更新ASC的范围预测键（ScopedPredictionKey），进入该预测键作用域，即后续通过ASC产生的效果绑定的都是该预测键；当该对象生命周期结束时，会调用析构函数，会回退状态，离开该预测键作用域。

Owner用于持有ASC的弱引用，ClearScopedPredictionKey控制析构时是否恢复旧键，SetReplicatedPredictionKey控制析构时是否将当前键提交到复制映射表，RestoreKey则是记录范围预测窗口构造前ASC的范围预测键。

<details>
<summary>点击展开/折叠 代码：FScopedPredictionWindow 结构体</summary>

```cpp
struct GAMEPLAYABILITIES_API FScopedPredictionWindow
{
	/** To be called on server when a new prediction key is received from the client (In an RPC). 
	 *	InSetReplicatedPredictionKey should be set to false in cases where we want a scoped prediction key but have already repped the prediction key.
	 *	(For example, cached target data will restore the prediction key that the TD was sent with, but this key was already repped down as confirmed when received)
	 **/
	FScopedPredictionWindow(UAbilitySystemComponent* AbilitySystemComponent, FPredictionKey InPredictionKey, bool InSetReplicatedPredictionKey = true);
	/** To be called in the callsite where the predictive code will take place. This generates a new PredictionKey and acts as a synchonization point between client and server for that key.  */
	FScopedPredictionWindow(UAbilitySystemComponent* AbilitySystemComponent, bool CanGenerateNewKey=true);
	~FScopedPredictionWindow();
private:
	TWeakObjectPtr<UAbilitySystemComponent> Owner;
	bool ClearScopedPredictionKey;
	bool SetReplicatedPredictionKey;
	FPredictionKey RestoreKey;
};
```

</details>

范围预测窗口提供了两种构造函数。第一种构造函数用于将一个已生成的预测键设置为当前作用域的预测键，应在服务器端调用，用于那些特定键有效的逻辑作用域。第二种构造函数用于生成一个新的的预测键设置为当前作用域的预测键，应在客户端调用，如果在服务器端则不会生成新键，直接返回（因为服务器是权威端，不需要预测键）。

<details>
<summary>点击展开/折叠 代码：FScopedPredictionWindow 构造函数</summary>

```cpp
FScopedPredictionWindow::FScopedPredictionWindow(UAbilitySystemComponent* AbilitySystemComponent, FPredictionKey InPredictionKey, bool InSetReplicatedPredictionKey /*=true*/)
{
	if (AbilitySystemComponent == nullptr)
	{
		return;
	}
	// This is used to set an already generated prediction key as the current scoped prediction key.
	// Should be called on the server for logical scopes where a given key is valid. E.g, "client gave me this key, we both are going to run Foo()".
	if (AbilitySystemComponent->IsNetSimulating() == false)
	{
		Owner = AbilitySystemComponent;
		check(Owner.IsValid());
		RestoreKey = AbilitySystemComponent->ScopedPredictionKey;
		AbilitySystemComponent->ScopedPredictionKey = InPredictionKey;
		ClearScopedPredictionKey = true;
		SetReplicatedPredictionKey = InSetReplicatedPredictionKey;
	}
}
FScopedPredictionWindow::FScopedPredictionWindow(UAbilitySystemComponent* InAbilitySystemComponent, bool bCanGenerateNewKey)
{
	// On the server, this will do nothing since it is authoritative and doesn't need a prediction key for anything.
	// On the client, this will generate a new prediction key if bCanGenerateNewKey is true, and we have a invalid prediction key.
	ClearScopedPredictionKey = false;
	SetReplicatedPredictionKey = false;
	// Owners that are mid destruction will not be valid and will trigger the ensure below (ie. when they stop their anim montages)
	// Original ensure has been left in to catch other cases of invalid Owner ASCs
	if ((!InAbilitySystemComponent) || (InAbilitySystemComponent->IsBeingDestroyed()) || (!IsValidChecked(InAbilitySystemComponent) || InAbilitySystemComponent->IsUnreachable()))
	{
		ABILITY_LOG(Verbose, TEXT("FScopedPredictionWindow() aborting due to Owner (ASC) being null, destroyed or pending kill / unreachable"));
		return;
	}
	Owner = InAbilitySystemComponent;
	if (!ensure(Owner.IsValid()) || InAbilitySystemComponent->IsNetSimulating() == false)
	{
		return;
	}
	// InAbilitySystemComponent->GetPredictionKey().IsValidForMorePrediction() == false && 
	if (bCanGenerateNewKey)
	{
		check(InAbilitySystemComponent != NULL); // Should have bailed above with ensure(Owner.IsValid())
		ClearScopedPredictionKey = true;
		RestoreKey = InAbilitySystemComponent->ScopedPredictionKey;
		InAbilitySystemComponent->ScopedPredictionKey.GenerateDependentPredictionKey();		
	}
}
```

</details>

范围预测窗口的析构函数会先判断是否需要将当前预测键提交到复制映射表（ReplicatedPredictionKeyMap）以便网络同步（服务器端且预测键有效），然后再将 ASC 的范围预测键回退到构造前的值。

<details>
<summary>点击展开/折叠 代码：FScopedPredictionWindow 析构函数</summary>

```cpp
FScopedPredictionWindow::~FScopedPredictionWindow()
{
	if (UAbilitySystemComponent* OwnerPtr = Owner.Get())
	{
		if (SetReplicatedPredictionKey)
		{
			// It is important to not set the ReplicatedPredictionKey unless it is valid (>0).
			// If we werent given a new prediction key for this scope from the client, then setting the
			// replicated prediction key back to 0 could cause OnReps to be missed on the client during high PL.
			// (for example, predict w/ key 100 -> prediction key replication dropped -> predict w/ invalid key -> next rep of prediction key is 0).
			if (OwnerPtr->ScopedPredictionKey.IsValidKey())
			{
				OwnerPtr->ReplicatedPredictionKeyMap.ReplicatePredictionKey(OwnerPtr->ScopedPredictionKey);
			}
		}
		if (ClearScopedPredictionKey)
		{
			OwnerPtr->ScopedPredictionKey = RestoreKey;
		}
	}
}
```

</details>

#### 4.1.4. FReplicatedPredictionKeyMap

复制预测键映射基于UE快速数组序列化（FastArraySerialization）实现。因此复制预测键映射的数组元素复制预测键项目（FReplicatedPredictionKeyItem）继承于FFastArraySerializerItem，以使用其提供的必要的成员变量和虚函数接口。如果不继承，编译会失败，因为：FFastArraySerializer 的 NetDeltaSerialize 依赖数组元素是 FFastArraySerializerItem 类型，且宏展开和模板约束会进行类型检查。

此外，复制预测键映射还需特化模板，实现NetDeltaSerialize，以使用增量序列化功能。

<details>
<summary>点击展开/折叠 代码：FReplicatedPredictionKeyMap NetDeltaSerialize 特化</summary>

```cpp
template<>
struct TStructOpsTypeTraits< FReplicatedPredictionKeyMap > : public TStructOpsTypeTraitsBase2< FReplicatedPredictionKeyMap >
{
	enum
	{
		WithNetDeltaSerializer = true, 
	};
};
bool FReplicatedPredictionKeyMap::NetDeltaSerialize(FNetDeltaSerializeInfo& DeltaParms)
{
	return FastArrayDeltaSerialize<FReplicatedPredictionKeyItem>(PredictionKeys, DeltaParms, *this);
}
```

</details>

复制预测键项目额外存储了预测键，重载了构造函数和赋值运算符。并且实现了PostReplicatedAdd和PostReplicatedChange，以在服务器该项目变化复制到客户端时回调OnRep处理。

<details>
<summary>点击展开/折叠 代码：FReplicatedPredictionKeyItem</summary>

```cpp
USTRUCT()
struct FReplicatedPredictionKeyItem : public FFastArraySerializerItem
{
	GENERATED_USTRUCT_BODY()
	// As we know that FReplicatedPredictionKeyItem is well behaved and does not leak outside of FReplicatedPredictionKeyMap we allow ReplicationID and ReplicationKey to be copied around to avoid issues when instantiating from archetype or CDO
	FReplicatedPredictionKeyItem();
	FReplicatedPredictionKeyItem(const FReplicatedPredictionKeyItem& Other);
	FReplicatedPredictionKeyItem(FReplicatedPredictionKeyItem&& Other);
	FReplicatedPredictionKeyItem& operator=(FReplicatedPredictionKeyItem&& other);
	FReplicatedPredictionKeyItem& operator=(const FReplicatedPredictionKeyItem& other);
	UPROPERTY()
	FPredictionKey PredictionKey;
	void PostReplicatedAdd(const struct FReplicatedPredictionKeyMap &InArray) { OnRep(); }
	void PostReplicatedChange(const struct FReplicatedPredictionKeyMap &InArray) { OnRep(); }
	FString GetDebugString() { return PredictionKey.ToString(); }

private:
	void OnRep();
};
```

</details>

在复制预测键项目的OnRep中，会执行该项目的预测键的CatchUpTo，以同步到服务器的伴生效果。并且处理在环形缓冲区中服务器确认（MapIt.Key() <= PredictionKey.Current）但由于被该项目覆盖（MapIt.Key() % FReplicatedPredictionKeyMap::KeyRingBufferSize == Index）的未CatchUpTo的预测键，执行其CaughtUpDelegates后将其从DelegateMap中移出。

<details>
<summary>点击展开/折叠 代码：FReplicatedPredictionKeyItem::OnRep</summary>

```cpp
void FReplicatedPredictionKeyItem::OnRep()
{
	ABILITY_LOG(Verbose, TEXT("FReplicatedPredictionKeyItem::OnRep %s"), *PredictionKey.ToString());
	// Every predictive action we've done up to and including the current value of ReplicatedPredictionKey needs to be wiped
	FPredictionKeyDelegates::CatchUpTo(PredictionKey.Current);
	// Sanity checking
	int32 Index = PredictionKey.Current % FReplicatedPredictionKeyMap::KeyRingBufferSize;
	for (auto MapIt = FPredictionKeyDelegates::Get().DelegateMap.CreateIterator(); MapIt; ++MapIt)
	{
		// If older key
		if (MapIt.Key() <= PredictionKey.Current)
		{	
			// Older key that would have gone in this slot
			if (MapIt.Key() % FReplicatedPredictionKeyMap::KeyRingBufferSize == Index)
			{
				// Message the log, this can happen during normal gameplay due to replication order, but can also indicate an ability-specific issue
				ABILITY_LOG(Log, TEXT("Passed PredictionKey %d in Delegate map while OnRep'ing %s"), MapIt.Key(), *PredictionKey.ToString());

				// Execute CaughtUp delegates
				for (auto& Delegate : MapIt.Value().CaughtUpDelegates)
				{
					Delegate.ExecuteIfBound();
				}

				// Cleanup
				MapIt.RemoveCurrent();
			}
		}
	}
}
```

</details>

复制预测键映射采用固定大小的环形缓冲区（默认32槽位），避免动态扩容开销，并且提供ReplicatePredictionKey以将预测键存储到对应的槽位（Index = (Key.Current % KeyRingBufferSize)）。

<details>
<summary>点击展开/折叠 代码：FReplicatedPredictionKeyMap 环形缓冲区</summary>

```cpp
const int32 FReplicatedPredictionKeyMap::KeyRingBufferSize = 32;
void FReplicatedPredictionKeyMap::ReplicatePredictionKey(FPredictionKey Key)
{	
	int32 Index = (Key.Current % KeyRingBufferSize);
	PredictionKeys[Index].PredictionKey = Key;
	MarkItemDirty(PredictionKeys[Index]);
}
```

</details>

复制预测键映射在构造函数中将复制预测键项目数组的大小设置为环形缓冲区的大小，并且将所有PredictionKey项标记为脏（Dirty），以确保新连接的客户端能够获得完整的服务器状态。

<details>
<summary>点击展开/折叠 代码：FReplicatedPredictionKeyMap 构造函数</summary>

```cpp
FReplicatedPredictionKeyMap::FReplicatedPredictionKeyMap()
{
	PredictionKeys.SetNum(KeyRingBufferSize);
	for (FReplicatedPredictionKeyItem& Item : PredictionKeys)
	{
		MarkItemDirty(Item);
	}
}
```

</details>

### 4.2. AbilitySystemComponent

#### 4.2.1. 预测键管理

范围预测键（ScopedPredictionKey）是当前作用域下的预测键。在创建范围预测键窗口（FScopedPredictionWindow）时会在其构造函数中被更新，在析构函数中还原。所有在范围预测键窗口作用域内执行的预测操作（如激活能力、应用 GE）都会使用这个键，它是预测操作的"身份凭证"。

<details>
<summary>点击展开/折叠 代码：ScopedPredictionKey</summary>

```cpp
/** Current prediction key, set with FScopedPredictionWindow */
FPredictionKey	ScopedPredictionKey;
```

</details>

复制预测键映射（ReplicatedPredictionKeyMap）会在服务器端的范围预测键窗口的析构函数触发时，存储该作用域的有效预测键的复制预测键项目（ReplicatedPredictionKeyItem）。通过复制，客户端会触发同步过来的复制预测键项目的回调函数，以激活该预测键的CatchUpTo函数，告知客户端服务器已实现了该预测键的伴生效果。必须放在最后复制以确保回调顺序。

<details>
<summary>点击展开/折叠 代码：ReplicatedPredictionKeyMap</summary>

```cpp
/** PredictionKeys, see more info in GameplayPrediction.h. This has to come *last* in all replicated properties on the AbilitySystemComponent to ensure OnRep/callback order. */
UPROPERTY(Replicated, Transient)
FReplicatedPredictionKeyMap ReplicatedPredictionKeyMap;
```

</details>

ASC提供了GetPredictionKeyForNewAction()和CanPredict()这两个内联函数，获取当前预测键以供新动作使用，或判断当前预测键是否有效。

<details>
<summary>点击展开/折叠 代码：GetPredictionKeyForNewAction 与 CanPredict</summary>

```cpp
/** Returns the prediction key that should be used for any actions */
FPredictionKey GetPredictionKeyForNewAction() const
{
	return ScopedPredictionKey.IsValidForMorePrediction() ? ScopedPredictionKey : FPredictionKey();
}
/** Do we have a valid prediction key to do more predictive actions with */
bool CanPredict() const
{
	return ScopedPredictionKey.IsValidForMorePrediction();
}
```

</details>

#### 4.2.2. 预测权限判断

ASC提供网络权限的相关判断函数。HasAuthorityOrPredictionKey会在服务器端并且预测键有效时返回真，IsOwnerActorAuthoritative()则会在组件持有者权威时返回真。并且提供了HasNetworkAuthorityToActivateTriggeredAbility以判断指定能力是否可以在当前网络模式下通过事件触发激活。

<details>
<summary>点击展开/折叠 代码：预测权限判断函数</summary>

```cpp
/** Returns true if this is running on the server or has a valid prediciton key */
bool UAbilitySystemComponent::HasAuthorityOrPredictionKey(const FGameplayAbilityActivationInfo* ActivationInfo) const
{
	return ((ActivationInfo->ActivationMode == EGameplayAbilityActivationMode::Authority) || CanPredict());
}
/** Returns true if this component's actor has authority */
bool UAbilitySystemComponent::IsOwnerActorAuthoritative() const
{
	return !bCachedIsNetSimulated;
}
/** Returns true if the specified ability should be activated from an event in this network mode */
bool UAbilitySystemComponent::HasNetworkAuthorityToActivateTriggeredAbility(const FGameplayAbilitySpec &Spec) const
{
	bool bIsAuthority = IsOwnerActorAuthoritative();
	bool bIsLocal = AbilityActorInfo->IsLocallyControlled();
	switch (Spec.Ability->GetNetExecutionPolicy())
	{
	case EGameplayAbilityNetExecutionPolicy::LocalOnly:
	case EGameplayAbilityNetExecutionPolicy::LocalPredicted:
		return bIsLocal;
	case EGameplayAbilityNetExecutionPolicy::ServerOnly:
	case EGameplayAbilityNetExecutionPolicy::ServerInitiated:
		return bIsAuthority;
	}
	return false;
}
```

</details>

此外，ASC还会通过CacheIsNetSimulated函数将网络角色缓存到bCachedIsNetSimulated。

<details>
<summary>点击展开/折叠 代码：CacheIsNetSimulated</summary>

```cpp
/** Cached value of rather this is a simulated actor */
UPROPERTY()
bool bCachedIsNetSimulated;
/** Caches the flags that indicate whether this component has network authority. */
void UAbilitySystemComponent::CacheIsNetSimulated()
{
	bCachedIsNetSimulated = IsNetSimulating();
	ActiveGameplayEffects.OwnerIsNetAuthority = IsOwnerActorAuthoritative();
	UpdateActiveGameplayEffectsReplicationCondition();
}
```

</details>

#### 4.2.3. 预测复制模式

ASC 定义了三种不同的复制模式Full、Mixed 和 Minimal，用于复制 GameplayEffects、GameplayTags 和 GameplayCues。Attributes 由其 AttributeSet 复制。Full模式将每个GE复制给所有客户端，适用于单机游戏；Mixed模式仅将GE复制给拥有该ASC的客户端，其他客户端只收到Tags和Cues，适用于玩家控制的角色；Minimal模式完全不复制GE，所有客户端仅同步Tags和Cues，适用于AI角色。

通过SetReplicationMode设置GE的复制模式，并存储在ReplicationMode中。

<details>
<summary>点击展开/折叠 代码：SetReplicationMode</summary>

```cpp
/** When true, we will not replicate active gameplay effects for this ability system component, so attributes and tags */
void UAbilitySystemComponent::SetReplicationMode(EGameplayEffectReplicationMode NewReplicationMode)
{
	ReplicationMode = NewReplicationMode;
	// The changing of replication mode can affect replication conditions for ActiveGameplayEffects and MinimalReplicationGameplayCues.
	// It's ok to call these before the component is replicated, GetReplicatedCustomConditionState will make sure the conditions are up-to-date.
	UpdateActiveGameplayEffectsReplicationCondition();
	UpdateMinimalReplicationGameplayCuesCondition();
}
/** How gameplay effects are replicated */
EGameplayEffectReplicationMode ReplicationMode;
```

</details>

并且提供了UpdateActiveGameplayEffectsReplicationCondition与UpdateMinimalReplicationGameplayCuesCondition以更新对应内容的复制条件，控制ActiveGameplayEffects和GameplayCues的网络复制状态。

<details>
<summary>点击展开/折叠 代码：更新复制条件</summary>

```cpp
void UAbilitySystemComponent::UpdateActiveGameplayEffectsReplicationCondition()
{
	if (ActiveGameplayEffects.IsUsingReplicationCondition())
	{
		DOREPDYNAMICCONDITION_SETCONDITION_FAST(ThisClass, ActiveGameplayEffects, ActiveGameplayEffects.GetReplicationCondition());
	}
}
void UAbilitySystemComponent::UpdateMinimalReplicationGameplayCuesCondition()
{
	DOREPCUSTOMCONDITION_SETACTIVE_FAST(ThisClass, MinimalReplicationGameplayCues, MinimalReplicationGameplayCues.ShouldReplicate());
}
```

</details>

#### 4.2.4. 预测能力（GameplayAbility）

ASC中与能力激活预测有关的逻辑主要在InternalTryActivateAbility函数中。这个函数会处理能力的实例化、网络和预测，处理成功后便会调用CallActivateAbility激活技能。

在网络和预测部分（即该段文字仅梳理网络相关，实例化相关不讲解），该函数首先会获取AvatarActor的网络角色（LocalRole）和本地控制权状态（IsLocallyControlled）。

如果不由本地玩家控制（!bIsLocal），并且能力的网络执行策略为只在本地或者本地预测但是预测键无效（Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::LocalOnly || (Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::LocalPredicted && !InPredictionKey.IsValidKey())）时，能力激活失败，并且广播失败信息。

如果网络角色不为权威（NetMode != ROLE_Authority），并且网络执行策略为只在服务器或者服务器初始化（Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::ServerOnly || Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::ServerInitiated）时，能力激活失败，并且广播失败信息。

如果在服务器端（NetMode == ROLE_Authority）或者网络执行策略为只在本地（Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::LocalOnly），进入该分支。在服务器端，如果传入的预测键无效，或者能力的网络执行策略为只在服务器或者服务器初始化（NetMode == ROLE_Authority && (!InPredictionKey.IsValidKey() || (Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::ServerInitiated ||  Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::ServerOnly))）时，新建预测键并作为该技能实例（GameplayAbilitySpec）的当前激活的预测键（PredictionKeyWhenActivated）；否则直接用传入该函数的有效预测键（InPredictionKey.IsValidKey()）。随后用该预测键创建范围预测窗口（ScopedPredictionWindow）。如果网络执行策略不为只在服务器（Ability->GetNetExecutionPolicy() != EGameplayAbilityNetExecutionPolicy::ServerOnly），并且不由本地玩家控制（!bIsLocal），则通过远程过程调用（RPC）通知客户端技能激活成功。而后便是激活技能（CallActivateAbility）。

如果未进入上段分支（即不在服务器端并且执行策略不是只在本地），并且网络执行策略为本地预测（Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::LocalPredicted），进入该分支。如果在客户端（!ActorInfo->IsNetAuthority()），为了防止触发动画根运动或影响移动的能力导致网络纠正，会先刷新在此能力激活之前发生的服务器移动数据，以确保服务器按正确顺序接收RPC调用。随后创建范围预测窗口，并将其生成的范围预测键（ScopedPredictionKey）作为该能力的预测键。随后立即让服务器以该预测键尝试激活能力（CallServerTryActivateAbility），并且将OnClientActivateAbilityCaughtUp绑定到该预测键的CatchUp委托上，以通过Log的形式观察预测情况。如果能力实例策略不为每次执行实例（EGameplayAbilityInstancingPolicy::InstancedPerExecution），激活技能。如果能力实例策略为每次执行实例并且能力复制策略为不复制（EGameplayAbilityReplicationPolicy::ReplicateNo），激活技能；反之（能力实例策略为每次执行实例并且能力复制策略为复制），则通过Log报错，说明该配置不支持本地预测。

<details>
<summary>点击展开/折叠 代码：InternalTryActivateAbility</summary>

```cpp
/**
 * Attempts to activate the ability.
 *	-This function calls CanActivateAbility
 *	-This function handles instancing
 *	-This function handles networking and prediction
 *	-If all goes well, CallActivateAbility is called next.
 */
bool UAbilitySystemComponent::InternalTryActivateAbility(FGameplayAbilitySpecHandle Handle, FPredictionKey InPredictionKey, UGameplayAbility** OutInstancedAbility, FOnGameplayAbilityEnded::FDelegate* OnGameplayAbilityEndedDelegate, const FGameplayEventData* TriggerEventData)
{
	const FGameplayTag& NetworkFailTag = UAbilitySystemGlobals::Get().ActivateFailNetworkingTag;
	
	InternalTryActivateAbilityFailureTags.Reset();
	if (Handle.IsValid() == false)
	{
		ABILITY_LOG(Warning, TEXT("InternalTryActivateAbility called with invalid Handle! ASC: %s. AvatarActor: %s"), *GetPathName(), *GetNameSafe(GetAvatarActor_Direct()));
		return false;
	}
	FGameplayAbilitySpec* Spec = FindAbilitySpecFromHandle(Handle);
	if (!Spec)
	{
		ABILITY_LOG(Warning, TEXT("InternalTryActivateAbility called with a valid handle but no matching ability was found. Handle: %s ASC: %s. AvatarActor: %s"), *Handle.ToString(), *GetPathName(), *GetNameSafe(GetAvatarActor_Direct()));
		return false;
	}
	// Lock ability list so our Spec doesn't get destroyed while activating
	ABILITYLIST_SCOPE_LOCK();
	const FGameplayAbilityActorInfo* ActorInfo = AbilityActorInfo.Get();
	// make sure the ActorInfo and then Actor on that FGameplayAbilityActorInfo are valid, if not bail out.
	if (ActorInfo == nullptr || !ActorInfo->OwnerActor.IsValid() || !ActorInfo->AvatarActor.IsValid())
	{
		return false;
	}
	// This should only come from button presses/local instigation (AI, etc)
	ENetRole NetMode = ROLE_SimulatedProxy;
	// Use PC netmode if its there
	if (APlayerController* PC = ActorInfo->PlayerController.Get())
	{
		NetMode = PC->GetLocalRole();
	}
	// Fallback to avataractor otherwise. Edge case: avatar "dies" and becomes torn off and ROLE_Authority. We don't want to use this case (use PC role instead).
	else if (AActor* LocalAvatarActor = GetAvatarActor_Direct())
	{
		NetMode = LocalAvatarActor->GetLocalRole();
	}
	if (NetMode == ROLE_SimulatedProxy)
	{
		return false;
	}
	bool bIsLocal = AbilityActorInfo->IsLocallyControlled();
	UGameplayAbility* Ability = Spec->Ability;
	if (!Ability)
	{
		ABILITY_LOG(Warning, TEXT("InternalTryActivateAbility called with invalid Ability"));
		return false;
	}
	// Check to see if this a local only or server only ability, if so don't execute
	if (!bIsLocal)
	{
		if (Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::LocalOnly || (Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::LocalPredicted && !InPredictionKey.IsValidKey()))
		{
			// If we have a valid prediction key, the ability was started on the local client so it's okay
			ABILITY_LOG(Warning, TEXT("Can't activate LocalOnly or LocalPredicted ability %s when not local! Net Execution Policy is %d."), *Ability->GetName(), (int32)Ability->GetNetExecutionPolicy());
			if (NetworkFailTag.IsValid())
			{
				InternalTryActivateAbilityFailureTags.AddTag(NetworkFailTag);
				NotifyAbilityFailed(Handle, Ability, InternalTryActivateAbilityFailureTags);
			}
			return false;
		}		
	}
	if (NetMode != ROLE_Authority && (Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::ServerOnly || Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::ServerInitiated))
	{
		ABILITY_LOG(Warning, TEXT("Can't activate ServerOnly or ServerInitiated ability %s when not the server! Net Execution Policy is %d."), *Ability->GetName(), (int32)Ability->GetNetExecutionPolicy());
		if (NetworkFailTag.IsValid())
		{
			InternalTryActivateAbilityFailureTags.AddTag(NetworkFailTag);
			NotifyAbilityFailed(Handle, Ability, InternalTryActivateAbilityFailureTags);
		}
		return false;
	}
	// If it's an instanced one, the instanced ability will be set, otherwise it will be null
	UGameplayAbility* InstancedAbility = Spec->GetPrimaryInstance();
	if (TriggerEventData)
	{
		UGameplayAbility* AbilitySource = InstancedAbility ? InstancedAbility : Ability;
		if (!AbilitySource->ShouldAbilityRespondToEvent(ActorInfo, TriggerEventData))
		{
			NotifyAbilityFailed(Handle, AbilitySource, InternalTryActivateAbilityFailureTags);
			return false;
		}
	}
	{
		const FGameplayTagContainer* SourceTags = TriggerEventData ? &TriggerEventData->InstigatorTags : nullptr;
		const FGameplayTagContainer* TargetTags = TriggerEventData ? &TriggerEventData->TargetTags : nullptr;
		// If we have an instanced ability, call CanActivateAbility on it.
		// Otherwise we always do a non instanced CanActivateAbility check using the CDO of the Ability.
		UGameplayAbility* const CanActivateAbilitySource = InstancedAbility ? InstancedAbility : Ability;
		FScopedCanActivateAbilityLogEnabler LogEnabler;
		if (!CanActivateAbilitySource->CanActivateAbility(Handle, ActorInfo, SourceTags, TargetTags, &InternalTryActivateAbilityFailureTags))
		{
			NotifyAbilityFailed(Handle, CanActivateAbilitySource, InternalTryActivateAbilityFailureTags);
			return false;
		}
	}
	// If we're instance per actor and we're already active, don't let us activate again as this breaks the graph
	if (Ability->GetInstancingPolicy() == EGameplayAbilityInstancingPolicy::InstancedPerActor)
	{
		if (Spec->IsActive())
		{
			if (Ability->bRetriggerInstancedAbility && InstancedAbility)
			{
				bool bReplicateEndAbility = true;
				bool bWasCancelled = false;
				InstancedAbility->EndAbility(Handle, ActorInfo, Spec->ActivationInfo, bReplicateEndAbility, bWasCancelled);
			}
			else
			{
				ABILITY_LOG(Verbose, TEXT("Can't activate instanced per actor ability %s when their is already a currently active instance for this actor."), *Ability->GetName());
				return false;
			}
		}
	}
	// Make sure we have a primary
	if (Ability->GetInstancingPolicy() == EGameplayAbilityInstancingPolicy::InstancedPerActor && !InstancedAbility)
	{
		ABILITY_LOG(Warning, TEXT("InternalTryActivateAbility called but instanced ability is missing! NetMode: %d. Ability: %s"), (int32)NetMode, *Ability->GetName());
		return false;
	}
	// Setup a fresh ActivationInfo for this AbilitySpec.
	Spec->ActivationInfo = FGameplayAbilityActivationInfo(ActorInfo->OwnerActor.Get());
	FGameplayAbilityActivationInfo &ActivationInfo = Spec->ActivationInfo;
	// If we are the server or this is local only
	if (Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::LocalOnly || (NetMode == ROLE_Authority))
	{
		// if we're the server and don't have a valid key or this ability should be started on the server create a new activation key
		bool bCreateNewServerKey = NetMode == ROLE_Authority &&
			(!InPredictionKey.IsValidKey() ||
			 (Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::ServerInitiated ||
			  Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::ServerOnly));
		if (bCreateNewServerKey)
		{
			ActivationInfo.ServerSetActivationPredictionKey(FPredictionKey::CreateNewServerInitiatedKey(this));
		}
		else if (InPredictionKey.IsValidKey())
		{
			// Otherwise if available, set the prediction key to what was passed up
			ActivationInfo.ServerSetActivationPredictionKey(InPredictionKey);
		}
		// we may have changed the prediction key so we need to update the scoped key to match
		FScopedPredictionWindow ScopedPredictionWindow(this, ActivationInfo.GetActivationPredictionKey());

		// ----------------------------------------------
		// Tell the client that you activated it (if we're not local and not server only)
		// ----------------------------------------------
		if (!bIsLocal && Ability->GetNetExecutionPolicy() != EGameplayAbilityNetExecutionPolicy::ServerOnly)
		{
			if (TriggerEventData)
			{
				ClientActivateAbilitySucceedWithEventData(Handle, ActivationInfo.GetActivationPredictionKey(), *TriggerEventData);
			}
			else
			{
				ClientActivateAbilitySucceed(Handle, ActivationInfo.GetActivationPredictionKey());
			}
			// This will get copied into the instanced abilities
			ActivationInfo.bCanBeEndedByOtherInstance = Ability->bServerRespectsRemoteAbilityCancellation;
		}
		// ----------------------------------------------
		//	Call ActivateAbility (note this could end the ability too!)
		// ----------------------------------------------
		// Create instance of this ability if necessary
		if (Ability->GetInstancingPolicy() == EGameplayAbilityInstancingPolicy::InstancedPerExecution)
		{
			InstancedAbility = CreateNewInstanceOfAbility(*Spec, Ability);
			InstancedAbility->CallActivateAbility(Handle, ActorInfo, ActivationInfo, OnGameplayAbilityEndedDelegate, TriggerEventData);
		}
		else if (InstancedAbility)
		{
			InstancedAbility->CallActivateAbility(Handle, ActorInfo, ActivationInfo, OnGameplayAbilityEndedDelegate, TriggerEventData);
		}
		else
		{
			Ability->CallActivateAbility(Handle, ActorInfo, ActivationInfo, OnGameplayAbilityEndedDelegate, TriggerEventData);
		}
	}
	else if (Ability->GetNetExecutionPolicy() == EGameplayAbilityNetExecutionPolicy::LocalPredicted)
	{
		// Flush server moves that occurred before this ability activation so that the server receives the RPCs in the correct order
		// Necessary to prevent abilities that trigger animation root motion or impact movement from causing network corrections
		if (!ActorInfo->IsNetAuthority())
		{
			ACharacter* AvatarCharacter = Cast<ACharacter>(ActorInfo->AvatarActor.Get());
			if (AvatarCharacter)
			{
				UCharacterMovementComponent* AvatarCharMoveComp = Cast<UCharacterMovementComponent>(AvatarCharacter->GetMovementComponent());
				if (AvatarCharMoveComp)
				{
					AvatarCharMoveComp->FlushServerMoves();
				}
			}
		}
		// This execution is now officially EGameplayAbilityActivationMode:Predicting and has a PredictionKey
		FScopedPredictionWindow ScopedPredictionWindow(this, true);
		ActivationInfo.SetPredicting(ScopedPredictionKey);
		// This must be called immediately after GeneratePredictionKey to prevent problems with recursively activating abilities
		if (TriggerEventData)
		{
			ServerTryActivateAbilityWithEventData(Handle, Spec->InputPressed, ScopedPredictionKey, *TriggerEventData);
		}
		else
		{
			CallServerTryActivateAbility(Handle, Spec->InputPressed, ScopedPredictionKey);
		}
		// When this prediction key is caught up, we better know if the ability was confirmed or rejected
		ScopedPredictionKey.NewCaughtUpDelegate().BindUObject(this, &UAbilitySystemComponent::OnClientActivateAbilityCaughtUp, Handle, ScopedPredictionKey.Current);
		if (Ability->GetInstancingPolicy() == EGameplayAbilityInstancingPolicy::InstancedPerExecution)
		{
			// For now, only NonReplicated + InstancedPerExecution abilities can be Predictive.
			// We lack the code to predict spawning an instance of the execution and then merge/combine
			// with the server spawned version when it arrives.
			if (Ability->GetReplicationPolicy() == EGameplayAbilityReplicationPolicy::ReplicateNo)
			{
				InstancedAbility = CreateNewInstanceOfAbility(*Spec, Ability);
				InstancedAbility->CallActivateAbility(Handle, ActorInfo, ActivationInfo, OnGameplayAbilityEndedDelegate, TriggerEventData);
			}
			else
			{
				ABILITY_LOG(Error, TEXT("InternalTryActivateAbility called on ability %s that is InstancePerExecution and Replicated. This is an invalid configuration."), *Ability->GetName() );
			}
		}
		else if (InstancedAbility)
		{
			InstancedAbility->CallActivateAbility(Handle, ActorInfo, ActivationInfo, OnGameplayAbilityEndedDelegate, TriggerEventData);
		}
		else 
		{
			Ability->CallActivateAbility(Handle, ActorInfo, ActivationInfo, OnGameplayAbilityEndedDelegate, TriggerEventData);
		}
	}
	if (InstancedAbility)
	{
		if (OutInstancedAbility)
		{
			*OutInstancedAbility = InstancedAbility;
		}
		// UGameplayAbility::PreActivate actually sets this internally (via SetCurrentInfo) which happens after replication (this is only set locally).  Let's cautiously remove this code.
		if (CVarAbilitySystemSetActivationInfoMultipleTimes.GetValueOnGameThread())
		{
			InstancedAbility->SetCurrentActivationInfo(ActivationInfo);	// Need to push this to the ability if it was instanced.
		}
	}
	MarkAbilitySpecDirty(*Spec);
	AbilityLastActivatedTime = GetWorld()->GetTimeSeconds();
	return true;
}
```

</details>

此外，便是与这个流程相关的远程过程调用（RPC）。

ServerTryActivateAbility、ServerTryActivateAbilityWithEventData、ServerEndAbility和ServerCancelAbility为客户端向服务器的调用，携带预测键，用以请求能力的激活、结束和取消。

<details>
<summary>点击展开/折叠 代码：Client->Server RPC</summary>

```cpp
UFUNCTION(Server, reliable, WithValidation)
void	ServerTryActivateAbility(FGameplayAbilitySpecHandle AbilityToActivate, bool InputPressed, FPredictionKey PredictionKey);
UFUNCTION(Server, reliable, WithValidation)
void	ServerTryActivateAbilityWithEventData(FGameplayAbilitySpecHandle AbilityToActivate, bool InputPressed, FPredictionKey PredictionKey, FGameplayEventData TriggerEventData);
UFUNCTION(Server, reliable, WithValidation)
void	ServerEndAbility(FGameplayAbilitySpecHandle AbilityToEnd, FGameplayAbilityActivationInfo ActivationInfo, FPredictionKey PredictionKey);
UFUNCTION(Server, reliable, WithValidation)
void    ServerCancelAbility(FGameplayAbilitySpecHandle AbilityToCancel, FGameplayAbilityActivationInfo ActivationInfo);
```

</details>

ClientTryActivateAbility、ClientEndAbility、ClientCancelAbility、ClientActivateAbilitySucceed、ClientActivateAbilitySucceedWithEventData和ClientActivateAbilityFailed则是服务器向客户端的调用，用以通知客户端能力的激活、结束、取消、成功和失败。并且提供了ClientActivateAbilityFailedCountRecent记录最近激活失败的次数，ClientActivateAbilityFailedStartTime记录开始统计失败的时间。

<details>
<summary>点击展开/折叠 代码：Server->Client RPC</summary>

```cpp
UFUNCTION(Client, reliable)
void	ClientTryActivateAbility(FGameplayAbilitySpecHandle AbilityToActivate);
UFUNCTION(Client, reliable)
void	ClientEndAbility(FGameplayAbilitySpecHandle AbilityToEnd, FGameplayAbilityActivationInfo ActivationInfo);
UFUNCTION(Client, reliable)
void    ClientCancelAbility(FGameplayAbilitySpecHandle AbilityToCancel, FGameplayAbilityActivationInfo ActivationInfo);
UFUNCTION(Client, Reliable)
void	ClientActivateAbilityFailed(FGameplayAbilitySpecHandle AbilityToActivate, int16 PredictionKey);
int32	ClientActivateAbilityFailedCountRecent;
float	ClientActivateAbilityFailedStartTime;
UFUNCTION(Client, Reliable)
void	ClientActivateAbilitySucceed(FGameplayAbilitySpecHandle AbilityToActivate, FPredictionKey PredictionKey);
UFUNCTION(Client, Reliable)
void	ClientActivateAbilitySucceedWithEventData(FGameplayAbilitySpecHandle AbilityToActivate, FPredictionKey PredictionKey, FGameplayEventData TriggerEventData);
```

</details>

ServerTryActivateAbility、ServerTryActivateAbilityWithEventData和ClientTryActivateAbility最终都是调用InternalTryActivateAbility进行能力的激活。但是ServerTryActivateAbility、ServerTryActivateAbilityWithEventData是先调用InternalServerTryActivateAbility筛掉失败的能力激活，并调用ClientActivateAbilityFailed处理失败，未失败才会调用InternalTryActivateAbility进行能力的激活。

<details>
<summary>点击展开/折叠 代码：InternalServerTryActivateAbility</summary>

```cpp
void UAbilitySystemComponent::InternalServerTryActivateAbility(FGameplayAbilitySpecHandle Handle, bool InputPressed, const FPredictionKey& PredictionKey, const FGameplayEventData* TriggerEventData)
{
#if WITH_SERVER_CODE
#if !(UE_BUILD_SHIPPING || UE_BUILD_TEST)
	if (DenyClientActivation > 0)
	{
		DenyClientActivation--;
		ClientActivateAbilityFailed(Handle, PredictionKey.Current);
		return;
	}
#endif
	ABILITYLIST_SCOPE_LOCK();
	FGameplayAbilitySpec* Spec = FindAbilitySpecFromHandle(Handle);
	if (!Spec)
	{
		// Can potentially happen in race conditions where client tries to activate ability that is removed server side before it is received.
		ABILITY_LOG(Display, TEXT("InternalServerTryActivateAbility. Rejecting ClientActivation of ability with invalid SpecHandle!"));
		ClientActivateAbilityFailed(Handle, PredictionKey.Current);
		return;
	}
	const UGameplayAbility* AbilityToActivate = Spec->Ability;
	if (!ensure(AbilityToActivate))
	{
		ABILITY_LOG(Error, TEXT("InternalServerTryActiveAbility. Rejecting ClientActivation of unconfigured spec ability!"));
		ClientActivateAbilityFailed(Handle, PredictionKey.Current);
		return;
	}
	// Ignore a client trying to activate an ability requiring server execution
	if (AbilityToActivate->GetNetSecurityPolicy() == EGameplayAbilityNetSecurityPolicy::ServerOnlyExecution ||
		AbilityToActivate->GetNetSecurityPolicy() == EGameplayAbilityNetSecurityPolicy::ServerOnly)
	{
		ABILITY_LOG(Display, TEXT("InternalServerTryActiveAbility. Rejecting ClientActivation of %s due to security policy violation."), *GetNameSafe(AbilityToActivate));
		ClientActivateAbilityFailed(Handle, PredictionKey.Current);
		return;
	}
	// Consume any pending target info, to clear out cancels from old executions
	ConsumeAllReplicatedData(Handle, PredictionKey);
	FScopedPredictionWindow ScopedPredictionWindow(this, PredictionKey);
	ensure(AbilityActorInfo.IsValid());

	SCOPE_CYCLE_COUNTER(STAT_AbilitySystemComp_ServerTryActivate);
	SCOPE_CYCLE_UOBJECT(Ability, AbilityToActivate);
	UGameplayAbility* InstancedAbility = nullptr;
	Spec->InputPressed = true;
	// Attempt to activate the ability (server side) and tell the client if it succeeded or failed.
	if (InternalTryActivateAbility(Handle, PredictionKey, &InstancedAbility, nullptr, TriggerEventData))
	{
		// TryActivateAbility handles notifying the client of success
	}
	else
	{
		ABILITY_LOG(Display, TEXT("InternalServerTryActivateAbility. Rejecting ClientActivation of %s. InternalTryActivateAbility failed: %s"), *GetNameSafe(Spec->Ability), *InternalTryActivateAbilityFailureTags.ToStringSimple() );
		ClientActivateAbilityFailed(Handle, PredictionKey.Current);
		Spec->InputPressed = false;
		MarkAbilitySpecDirty(*Spec);
	}
#endif
}
```

</details>

ServerEndAbility、ServerCancelAbility、ClientEndAbility和ClientCancelAbility的逻辑主要集中在RemoteEndOrCancelAbility函数中。该函数对于非实例化能力，直接调用其 CancelAbility 或 EndAbility；对于实例化能力，则遍历所有实例，仅对预测键匹配且可被其他实例结束（bCanBeEndedByOtherInstance）的实例执行操作，若为取消调用 ForceCancelAbilityDueToReplication，为结束则调用EndAbility，并在执行前先调用SetRemoteInstanceHasEnded标记远程实例已结束。

<details>
<summary>点击展开/折叠 代码：RemoteEndOrCancelAbility</summary>

```cpp
// This is only called when ending or canceling an ability in response to a remote instruction.
void UAbilitySystemComponent::RemoteEndOrCancelAbility(FGameplayAbilitySpecHandle AbilityToEnd, FGameplayAbilityActivationInfo ActivationInfo, bool bWasCanceled)
{
	FGameplayAbilitySpec* AbilitySpec = FindAbilitySpecFromHandle(AbilityToEnd);
	if (AbilitySpec && AbilitySpec->Ability && AbilitySpec->IsActive())
	{
		// Handle non-instanced case, which cannot perform prediction key validation
		if (AbilitySpec->Ability->GetInstancingPolicy() == EGameplayAbilityInstancingPolicy::NonInstanced)
		{
			// End/Cancel the ability but don't replicate it back to whoever called us
			if (bWasCanceled)
			{
				AbilitySpec->Ability->CancelAbility(AbilityToEnd, AbilityActorInfo.Get(), ActivationInfo, false);
			}
			else
			{
				AbilitySpec->Ability->EndAbility(AbilityToEnd, AbilityActorInfo.Get(), ActivationInfo, false, bWasCanceled);
			}
		}
		else
		{
			TArray<UGameplayAbility*> Instances = AbilitySpec->GetAbilityInstances();
			for (auto Instance : Instances)
			{
				UE_CLOG(Instance == nullptr, LogAbilitySystem, Fatal, TEXT("UAbilitySystemComponent::RemoteEndOrCancelAbility null instance for %s"), *GetNameSafe(AbilitySpec->Ability));
				// Check if the ability is the same prediction key (can both by 0) and has been confirmed. If so cancel it.
				if (Instance->GetCurrentActivationInfoRef().GetActivationPredictionKey() == ActivationInfo.GetActivationPredictionKey())
				{
					// Let the ability know that the remote instance has ended, even if we aren't about to end it here.
					Instance->SetRemoteInstanceHasEnded();
					if (Instance->GetCurrentActivationInfoRef().bCanBeEndedByOtherInstance)
					{
						// End/Cancel the ability but don't replicate it back to whoever called us
						if (bWasCanceled)
						{
							ForceCancelAbilityDueToReplication(Instance);
						}
						else
						{
							Instance->EndAbility(Instance->CurrentSpecHandle, Instance->CurrentActorInfo, Instance->CurrentActivationInfo, false, bWasCanceled);
						}
					}
				}
			}
		}
	}
}
```

</details>

ClientActivateAbilityFailed的处理逻辑为广播该有效预测键的拒绝委托，记录失败时间与频次，将预测键匹配的能力规格（GameplayAbilitySpec）的激活模式设置为拒绝（EGameplayAbilityActivationMode::Rejected），随后将预测键匹配的能力实例的激活模式设置为拒绝并结束能力实例。

<details>
<summary>点击展开/折叠 代码：ClientActivateAbilityFailed</summary>

```cpp
void UAbilitySystemComponent::ClientActivateAbilityFailed_Implementation(FGameplayAbilitySpecHandle Handle, int16 PredictionKey)
{
	// Tell anything else listening that this was rejected
	if (PredictionKey > 0)
	{
		FPredictionKeyDelegates::BroadcastRejectedDelegate(PredictionKey);
	}
	// Find the actual UGameplayAbility		
	FGameplayAbilitySpec* Spec = FindAbilitySpecFromHandle(Handle);
	if (Spec == nullptr)
	{
		ABILITY_LOG(Display, TEXT("ClientActivateAbilityFailed_Implementation. PredictionKey: %d Ability: Could not find!"), PredictionKey);
		return;
	}
	ABILITY_LOG(Display, TEXT("ClientActivateAbilityFailed_Implementation. PredictionKey :%d Ability: %s"), PredictionKey, *GetNameSafe(Spec->Ability));
	
	if (ClientActivateAbilityFailedPrintDebugThreshhold > 0)
	{
		if ((ClientActivateAbilityFailedStartTime <= 0.f) || ((GetWorld()->GetTimeSeconds() - ClientActivateAbilityFailedStartTime) > ClientActivateAbilityFailedPrintDebugThreshholdTime))
		{
			ClientActivateAbilityFailedStartTime = GetWorld()->GetTimeSeconds();
			ClientActivateAbilityFailedCountRecent = 0;
		}
		if (++ClientActivateAbilityFailedCountRecent > ClientActivateAbilityFailedPrintDebugThreshhold)
		{
			ABILITY_LOG(Display, TEXT("Threshold hit! Printing debug information"));
			PrintDebug();
			ClientActivateAbilityFailedCountRecent = 0;
			ClientActivateAbilityFailedStartTime = 0.f;
		}
	}
	// The ability should be either confirmed or rejected by the time we get here
	if (Spec->ActivationInfo.GetActivationPredictionKey().Current == PredictionKey)
	{
		Spec->ActivationInfo.SetActivationRejected();
	}
	TArray<UGameplayAbility*> Instances = Spec->GetAbilityInstances();
	for (UGameplayAbility* Ability : Instances)
	{
		if (Ability->CurrentActivationInfo.GetActivationPredictionKey().Current == PredictionKey)
		{
			Ability->CurrentActivationInfo.SetActivationRejected();
			Ability->K2_EndAbility();
		}
	}
}
```

</details>

ClientActivateAbilitySucceedWithEventData的处理流程如下。

首先，如果客户端不存在该能力规格，便用函数参数生成能力信息（FPendingAbilityInfo）存储在PendingServerActivatedAbilities中并结束函数，以供客户端后续激活。

如果能力有效，将其能力规格的激活模式设置为确认（EGameplayAbilityActivationMode::Confirmed），并设置为可被其他实例结束（bCanBeEndedByOtherInstance = true）。

当能力的网络执行策略为本地预测（EGameplayAbilityNetExecutionPolicy::LocalPredicted）并且能力实例策略不为无实例（EGameplayAbilityInstancingPolicy::NonInstanced）时，调用预测键匹配的实例的ConfirmActivateSucceed函数，广播确认委托。

当能力的网络执行策略不为本地预测时。如果预测键为服务器初始化的（PredictionKey.bIsServerInitiated），便将能力规格的激活预测键设置为该键。随后根据能力的实例策略激活能力。

<details>
<summary>点击展开/折叠 代码：ClientActivateAbilitySucceedWithEventData</summary>

```cpp
void UAbilitySystemComponent::ClientActivateAbilitySucceedWithEventData_Implementation(FGameplayAbilitySpecHandle Handle, FPredictionKey PredictionKey, FGameplayEventData TriggerEventData)
{
	FGameplayAbilitySpec* Spec = FindAbilitySpecFromHandle(Handle);
	if (!Spec)
	{
		// Can happen if the client gets told to activate an ability the same frame that abilities are added on the server
		FPendingAbilityInfo AbilityInfo;
		AbilityInfo.PredictionKey = PredictionKey;
		AbilityInfo.Handle = Handle;
		AbilityInfo.TriggerEventData = TriggerEventData;
		AbilityInfo.bPartiallyActivated = true;
		// This won't add it if we're currently being called from the pending list
		PendingServerActivatedAbilities.AddUnique(AbilityInfo);
		return;
	}
	UGameplayAbility* AbilityToActivate = Spec->Ability;
	check(AbilityToActivate);
	ensure(AbilityActorInfo.IsValid());
	Spec->ActivationInfo.SetActivationConfirmed();
	// ABILITY_LOG(Verbose, TEXT("ClientActivateAbilitySucceedWithEventData_Implementation. PredictionKey :%s Ability: %s"), *PredictionKey.ToString(), *GetNameSafe(Spec->Ability));
	// Fixme: We need a better way to link up/reconcile predictive replicated abilities. It would be ideal if we could predictively spawn an
	// ability and then replace/link it with the server spawned one once the server has confirmed it.
	if (AbilityToActivate->NetExecutionPolicy == EGameplayAbilityNetExecutionPolicy::LocalPredicted)
	{
		if (AbilityToActivate->GetInstancingPolicy() == EGameplayAbilityInstancingPolicy::NonInstanced)
		{
			// AbilityToActivate->ConfirmActivateSucceed(); // This doesn't do anything for non instanced
		}
		else
		{
			// Find the one we predictively spawned, tell them we are confirmed
			bool found = false;
			TArray<UGameplayAbility*> Instances = Spec->GetAbilityInstances();
			for (UGameplayAbility* LocalAbility : Instances)
			{
				if (LocalAbility != nullptr && LocalAbility->GetCurrentActivationInfo().GetActivationPredictionKey() == PredictionKey)
				{
					LocalAbility->ConfirmActivateSucceed();
					found = true;
					break;
				}
			}
			if (!found)
			{
				ABILITY_LOG(Verbose, TEXT("Ability %s was confirmed by server but no longer exists on client (replication key: %d"), *AbilityToActivate->GetName(), PredictionKey.Current);
			}
		}
	}
	else
	{
		// We haven't already executed this ability at all, so kick it off.
		if (PredictionKey.bIsServerInitiated)
		{
			// We have an active server key, set our key equal to it
			Spec->ActivationInfo.ServerSetActivationPredictionKey(PredictionKey);
		}
		if (AbilityToActivate->GetInstancingPolicy() == EGameplayAbilityInstancingPolicy::InstancedPerExecution)
		{
			// Need to instantiate this in order to execute
			UGameplayAbility* InstancedAbility = CreateNewInstanceOfAbility(*Spec, AbilityToActivate);
			InstancedAbility->CallActivateAbility(Handle, AbilityActorInfo.Get(), Spec->ActivationInfo, nullptr, TriggerEventData.EventTag.IsValid() ?  &TriggerEventData : nullptr);
		}
		else if (AbilityToActivate->GetInstancingPolicy() != EGameplayAbilityInstancingPolicy::NonInstanced)
		{
			UGameplayAbility* InstancedAbility = Spec->GetPrimaryInstance();
			if (!InstancedAbility)
			{
				ABILITY_LOG(Warning, TEXT("Ability %s cannot be activated on the client because it's missing a primary instance!"), *AbilityToActivate->GetName());
				return;
			}
			InstancedAbility->CallActivateAbility(Handle, AbilityActorInfo.Get(), Spec->ActivationInfo, nullptr, TriggerEventData.EventTag.IsValid() ? &TriggerEventData : nullptr);
		}
		else
		{
			AbilityToActivate->CallActivateAbility(Handle, AbilityActorInfo.Get(), Spec->ActivationInfo, nullptr, TriggerEventData.EventTag.IsValid() ? &TriggerEventData : nullptr);
		}
	}
}
```

</details>

待定服务器激活能力（PendingServerActivatedAbilities）是一个FPendingAbilityInfo结构体作为元素的数组（TArray）。该数组中的能力会在ActivatableAbilities的复制回调函数OnRep_ActivateAbilities中被激活。该函数会先等待能力资产复制完成，随后清理客户端上已被服务器移除的能力，并处理复制期间积压的激活请求。

<details>
<summary>点击展开/折叠 代码：OnRep_ActivateAbilities</summary>

```cpp
void UAbilitySystemComponent::OnRep_ActivateAbilities()
{
	for (FGameplayAbilitySpec& Spec : ActivatableAbilities.Items)
	{
		const UGameplayAbility* SpecAbility = Spec.Ability;
		if (!SpecAbility)
		{
			// Queue up another call to make sure this gets run again, as our abilities haven't replicated yet
			GetWorld()->GetTimerManager().SetTimer(OnRep_ActivateAbilitiesTimerHandle, this, &UAbilitySystemComponent::OnRep_ActivateAbilities, 0.5);
			return;
		}
	}
	CheckForClearedAbilities();
	// Try to run any pending activations that couldn't run before. If they don't work now, kill them
	for (const FPendingAbilityInfo& PendingAbilityInfo : PendingServerActivatedAbilities)
	{
		if (PendingAbilityInfo.bPartiallyActivated)
		{
			ClientActivateAbilitySucceedWithEventData_Implementation(PendingAbilityInfo.Handle, PendingAbilityInfo.PredictionKey, PendingAbilityInfo.TriggerEventData);
		}
		else
		{
			ClientTryActivateAbility(PendingAbilityInfo.Handle);
		}
	}
	PendingServerActivatedAbilities.Empty();
}
```

</details>
