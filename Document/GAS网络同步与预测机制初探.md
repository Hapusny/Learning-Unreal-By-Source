# GAS网络同步与预测机制初探

GAS在GameplayPrediction.h中对其实现的预测系统进行了叙述

在能力本身内部的逻辑中，对预测是无感的，不需要为预测实现任何逻辑。

> At the GameplayAbility level (implementing an ability) prediction is transparent.

> 在GameplayAbility层面（实现一个能力时），预测是透明的

预测系统的交互范围是需要服务器参与权威认证的客户端应预测的游戏状态。

> When we say "client side prediction" we really mean client predicting game simulation state. Things can still be 'completely client side' without having to work within a prediction system.

> 当我们说"客户端预测"时，我们实际上是指客户端预测游戏模拟状态。有些事物可以完全在客户端进行，而无需在预测系统内工作。

该系统目前实现的范围为

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

## 1. 预测系统的实现

### 1.1. 总览

GAS的预测基于一个整形标识符预测键（PredictionKey），客户端在激活GameplayAbility时生成，并通过CallServerTryActivateAbility()一起发送到服务器，在预测键有效并且在作用域内时，期间应用的GameplayEffect会被添加该预测键；服务器接收到该预测键时，会将该预测键添加到其应用的GameplayEffect中，同时该预测键会被复制回客户端；如果客户端预测的GameplayEffect与从服务器接收的GameplayEffect具有相同的预测键，则它们是正确预测的；客户端从服务器接收预测键后，该预测键便会被标记为过时；客户端会移除所有添加了过时预测键的GameplayEffect，而由服务器复制的GameplayEffect会被保留；如果被移除的GameplayEffect无匹配的服务器复制版本，则它们是错误预测的。

客户端生成的激活预测键（Activation Prediction Key）只在技能激活瞬间有效，它只负责"激活"这个行为。在技能激活后，来自延迟操作AbilityTasks的任何回调将不再具有有效的预测键，如果需要应用新的GE，应在AbilityTask内置同步点，以新建ScopedPredictionWindow生成新的预测键。

一些AbilityTasks，如所有与输入相关的任务，都内置了创建新的作用域预测窗口的功能。如果需要在没有内置创建作用域预测窗口功能的AbilityTask（如WaitDelay）之后预测动作，应使用带有OnlyServerWait选项的WaitNetSync AbilityTask手动执行此操作。当客户端遇到带有OnlyServerWait的WaitNetSync时，它会基于GameplayAbility的激活预测键生成一个新的作用域预测键。当服务器遇到带有OnlyServerWait的WaitNetSync时，它会等待收到来自客户端的新作用域预测键。此作用域预测键执行与激活预测键相同的操作。

使用WaitNetSync时，会阻止服务器的GameplayAbility继续执行，直到它收到客户端的消息。针对该问题，可以构建一个带有延迟的AbilityTask新版本，如果客户端没有响应，它会自动继续。

### 1.2. 预测键

预测键是预测系统的一个基本概念，它是在客户端生成的一个唯一ID，用于识别在客户端上执行的预测性操作和伴生效果。客户端会将其发送到服务器，并且关联到预测性操作和伴生效果上。服务器会接收或拒绝该预测键，如果接受了，服务器产生的伴生效果也会与其关联。

在预测键从服务器复制回客户端时，它们仅复制给最初将该预测键发送给服务器的那个客户端。

> A fundamental concept in this system is the Prediction Key (FPredictionKey). A prediction key on its own is simply a unique ID that is generated in a central place on the client. The client will send its prediction key to the server, and associate predictive actions and side effects with this key. The server may respond with an accept/reject for the prediction key, and will also associate the server-side created side effects with this prediction key.
>
> (IMPORTANT) FPredictionKey always replicate client -> server, but when replicating server -> clients they *only* replicate to the client that sent the prediction key to the server in the first place.
>
> This happens in FPredictionKey::NetSerialize. All other clients will receive an invalid (0) prediction key when a prediction key sent from a client is replicated back down through a replicated property.

> 本系统的一个基本概念是预测键（FPredictionKey）。预测键本身只是一个在客户端中心位置生成的唯一ID。客户端会将其预测键发送给服务器，并将预测性操作和副作用与此键关联。服务器可能会接受或拒绝该预测键，并也会将服务器端产生的副作用与此预测键关联。
>
> （重要）FPredictionKey 总是从客户端复制到服务器，但当从服务器复制回客户端时，它们仅复制给最初将该预测键发送给服务器的那个客户端。
>
> 这发生在 FPredictionKey::NetSerialize 中。所有其他客户端在通过复制属性接收到从客户端发送的预测键时，将收到一个无效（0）的预测键。

### 1.3. 能力激活

当客户端预测性地激活一个能力时，它会明确地向服务器请求（ServerTryActivateAbility），服务器也会明确地回应（ClientActivateAbility(Failed/Succeed)）。在能力被预测性地激活时，会为客户端创建一个预测窗口，在此窗口内发生的预测性伴生效果不会被明确"询问"，自动与生成的PredictionKey关联。

能力预测性激活的客户端和服务器间通信由一个往返构成。

客户端调用 TryActivateAbility，生成一个新的 FPredictionKey 并调用ServerTryActivateAbility。随后客户端在收到服务器回复之前继续执行，并调用ActivateAbility，将生成的 PredictionKey 与能力的 ActivationInfo 关联。在ActivateAbility 调用完成之前发生的任何副作用，都与生成的 FPredictionKey关联。

服务器在 ServerTryActivateAbility 中决定该能力是否真的发生，调用ClientActivateAbility(Failed/Succeed) 并将UAbilitySystemComponent::ReplicatedPredictionKey设置为发送来的生成键。

如果客户端收到ClientAbilityFailed，它会立即终止该能力并回滚与预测键关联的副作用。如果被接受，并且客户端的ReplicatedPredictionKey在属性复制完成后达到先前步骤中使用的键值，客户端就可以撤销其预测性副作用。

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

### 1.4. GameplayEffect预测

GameplayEffect被视为预测的副作用，不会被明确询问。只有在存在有效预测键的情况下，GameplayEffect才会在客户端上应用。并且如果GameplayEffect是可预测的，则关联的属性、GameplayCue和GameplayTag都会被预测。

当客户端的FActiveGameplayEffect创建时，它会存储预测键。在服务器上，相同的预测键也会被设置在服务器将要复制下来的FActiveGameplayEffect上。

客户端在收到一个带有有效预测键的复制FActiveGameplayEffect时，会检查是否拥有预测键相同的ActiveGameplayEffect。如果匹配，将不执行"应用时"类型的逻辑，同时ActiveGameplayEffects容器中会暂时有两个"相同"的GameplayEffect。在ReplicatedPredictionKey同步到客户端时，匹配的预测性的效果将被移除，ActiveGameplayEffects容器中就只剩下服务器复制的GameplayEffect。

> GameplayEffects are considered side effects of prediction and are not explicitly asked about.
>
> 1. GameplayEffects are only applied on clients if there is a valid prediction key. (If no prediction key, it simply skips the application on client).
> 2. Attributes, GameplayCues, and GameplayTags are all predicted if the GameplayEffect is predicted.
> 3. When the FActiveGameplayEffect is created, it stores the prediction key (FActiveGameplayEffect::PredictionKey)
>    3a. Instant effects are explained below in "Attribute Prediction".
> 4. On the server, the same prediction key is also set on the server's FActiveGameplayEffect that will be replicated down.
> 5. As a client, if you get a replicated FActiveGameplayEffect with a valid prediction key on it, you check to see if you have an ActiveGameplayEffect with that same key, if there is match, we do not apply the 'on applied' type of logic, e.g., GameplayCues. The solves the "Redo" problem. However we will have 2 of the 'same' GameplayEffects in our ActiveGameplayEffects container, temporarily.
> 6. At the same time, UAbilitySystemComponent::ReplicatedPredictionKey will catch up and the predictive effects will be removed. When they are removed in this case, we again check PredictionKey and decide if we should not do the 'On Remove' logic / GameplayCue.

> GameplayEffect 被视为预测的副作用，不会被明确询问。
>
> 1. 只有在存在有效预测键的情况下，GameplayEffect 才会在客户端上应用。（如果没有预测键，则在客户端上跳过应用）。
> 2. 如果 GameplayEffect 是可预测的，则属性、GameplayCue 和 GameplayTag 都会被预测。
> 3. 当 FActiveGameplayEffect 创建时，它会存储预测键（FActiveGameplayEffect::PredictionKey）。
>    3a. 即时效果在下面的"属性预测"中解释。
> 4. 在服务器上，相同的预测键也会被设置在服务器将要复制下来的 FActiveGameplayEffect 上。
> 5. 作为客户端，如果您收到一个带有有效预测键的复制 FActiveGameplayEffect，您会检查是否拥有相同键的 ActiveGameplayEffect。如果匹配，我们不执行"应用时"类型的逻辑（例如 GameplayCue）。这解决了"重做"问题。但是，我们的 ActiveGameplayEffects 容器中会暂时有两个"相同"的 GameplayEffect。
> 6. 同时，UAbilitySystemComponent::ReplicatedPredictionKey 会赶上，预测性的效果将被移除。当它们在这种情况下被移除时，我们再次检查 PredictionKey 并决定是否不应执行"移除时"逻辑/GameplayCue。

### 1.5. 属性预测

属性预测的基本策略是将其视为增量预测，而非绝对值预测。

将预测性的即时GameplayEffect视为无限持续时间的GameplayEffect，并且始终接收属性的RepNotify调用（不仅仅是在与上次本地值有变化时）。在属性RepNotify中，调用AbilitySystemComponent::ActiveGameplayEffects根据新的服务器复制值，来更新的施加预测修改后的最终值。当复制预测键被同步时，匹配的预测性的GameplayEffect 被移除，客户端的值恢复到服务器给定的值。

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

### 1.6. GameplayCue事件

除了通过GameplayEffect关联以外，GameplayCue也可以独立激活。

相关函数会考虑网络角色和预测键，表现如下

> 1. In UAbilitySystemComponent::ExecuteGameplayCue, if authority then do the multicast event (with replication key). If non authority but w/ a valid prediction key, predict the GameplayCue.
> 2. On the receiving end (NetMulticast_InvokeGameplayCueExecuted etc), if there is a replication key, then don't do the event (assume you predicted it).

> 1. 在 UAbilitySystemComponent::ExecuteGameplayCue 中，如果是权威端，则执行多播事件（带有复制键）。如果是非权威端但具有有效的预测键，则预测 GameplayCue。
> 2. 在接收端（NetMulticast_InvokeGameplayCueExecuted 等），如果存在复制键，则不执行事件（假设您已经预测了它）。

### 1.7. 触发数据预测

触发数据当前用于激活能力，其预测与ActivateAbility的代码路径相同。

能力是由游戏代码驱动的事件激活，因此客户端能够预测性地执行这些事件，从而预测性地激活能力。

服务器也会运行触发事件的代码，并保留通过预测性能力激活的触发能力的列表。当从触发能力接收到TryActivate时，服务器会查看是否已经运行了该能力，并用该信息进行响应。

> Triggered Data is currently used to activate abilities. Essentially this all goes through the same code path as ActivateAbility. Rather than the ability being activated from input press, it is activated from another game code driven event. Clients are able to predictively execute these events which predictively activate abilities.
>
> There are some nuances to however, since the server will also run the code that triggers events. The server won't just wait to hear from the client. The server will keep a list of triggered abilities that have been activated from a predictive ability. When receiving a TryActivate from a triggered ability, the server will look to see if /it/ has already run this ability, and respond with that information.

> 触发数据当前用于激活能力。基本上，这都经过与 ActivateAbility 相同的代码路径。能力不是由输入按键激活，而是由另一个游戏代码驱动的事件激活。客户端能够预测性地执行这些事件，从而预测性地激活能力。
>
> 然而，这有一些细微差别，因为服务器也会运行触发事件的代码。服务器不会只是等待客户端的消息。服务器会保留已从预测性能力激活的触发能力的列表。当从触发能力接收到 TryActivate 时，服务器会查看它是否已经运行了该能力，并用该信息进行响应。

## 2. 高级主题

### 2.1. 依赖性

GAS通过基础预测键（Base PredictionKey）处理能力连锁触发的依赖链，它是FPredictionKey的一个成员。

链头能力的预测键作为后续连锁能力预测键的基础键，并且后续能力的依赖关系在客户端处理，通过FPredictionKeyDelegates::AddDependancy实现。如果存在能力被拒绝/接受，会添加委托以拒绝/跟进后续能力。

为了简化状态，客户端发送到服务器的预测键只有链头能力的预测键，链条后续由服务器自己基于自身的权威状态重新执行，用最初的基础键来标记整个预测窗口。

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

### 2.2. 能力内的额外预测窗口

FScopedPredictionWindow提供了一种方法，可以向服务器发送一个新的预测键，并让服务器在相同的逻辑范围内接收并使用该键。

例子UAbilityTask_WaitInputRelease::OnReleaseCallback的事件流程为

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

## 3. 不支持/问题/待办事项

触发事件不会显式复制。如果触发事件仅在服务器上运行，客户端将永远不会知道它。

> Triggered events do not explicitly replicate. E.g., if a triggered event only runs on the server, the client will never hear about it. This also prevents us from doing cross player/AI etc events. Support for this should eventually be added and it should follow the same pattern that GameplayEffect and GameplayCues follow (predict triggered event with a prediction key, ignore the RPC event if it has a prediction key).

> 触发事件不会显式复制。例如，如果触发事件仅在服务器上运行，客户端将永远不会知道它。这也阻止了我们进行跨玩家/AI 等事件。最终应该添加对此的支持，并且它应该遵循 GameplayEffect 和 GameplayCue 相同的模式（使用预测键预测触发事件，如果 RPC 事件带有预测键则忽略它）。

### 3.1. 预测"元"属性（如伤害/治疗）与"真实"属性（如生命值）

无法预测性地应用元属性，元属性仅对即时效果有效。

> We are unable to apply meta attributes predictively. Meta attributes only work on instant effects, in the back end of GameplayEffect (Pre/Post Modify Attribute on the UAttributeSet). These events are not called when applying duration-based gameplay effects. E.g., a GameplayEffect that modifies damage for 5 seconds doesn't make sense.
>
> In order to support this, we would probably add some limited support for duration based meta attributes, and move the transform of the instant gameplay effect from the front end (UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf) to the backend (UAttributeSet::PostModifyAttribute).

> 我们无法预测性地应用元属性。元属性仅对即时效果有效，在 GameplayEffect 的后端（UAttributeSet 上的 Pre/Post Modify Attribute）中起作用。这些事件在应用基于持续时间的 GameplayEffect 时不会被调用。例如，一个修改伤害持续 5 秒的 GameplayEffect 是没有意义的。
>
> 为了支持这一点，我们可能会添加对基于持续时间的元属性的有限支持，并将即时 GameplayEffect 的转换从前端（UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf）移动到后端（UAttributeSet::PostModifyAttribute）。

### 3.2. 预测持续性的乘法型 GameplayEffect

在预测基于百分比的 GameplayEffect 时也存在限制。

服务器复制的是属性的"最终值"，而不是完整的聚合器链（所有修改它的内容），客户端无法准确预测新GameplayEffect 的情况。这将需要通过复制属性的聚合器链来解决。

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

### 3.3. "弱预测"

为了解决不完全适合此系统的情况，GAS设想了弱预测模式。当没有可以准确关联副作用的新的预测键时，某些能力（所有能力？）会回退到该模式。在弱预测模式下，只有某些操作可以被预测。

> We will probably still have cases that do not fit well into this system. Some situations will exist where a prediction key exchange is not feasible. For example, an ability where any one that player collides with/touches receives a GameplayEffect that slows them and their material blue. Since we can't send Server RPCs every time this happens (and the server couldn't necessarily handle the message at its point in the simulation), there is no way to correlate the gameplay effect side effects between client and server.
>
> One approach here may be to think about a weaker form of prediction. One where there is not a fresh prediction key used and instead the server assumes the client will predict all side effects from an entire ability. This would at least solve the "redo" problem but would not solve the "completeness" problem. If the client side prediction could be made as minimal as possible - for example only predicting an initial particle effect rather than predicting the state and attribute change - then the problems get less severe.
>
> I can envision a weak prediction mode which is what (certain abilities? All abilities?) fall back to when there is no fresh prediction key that can accurately correlate side effects. When in weak prediction mode, perhaps only certain actions can be predicted - for example GameplayCue execute events, but not OnAdded/OnRemove events.

> 我们可能仍然会遇到不完全适合此系统的情况。在某些情况下，预测键交换可能不可行。例如，一个能力，任何与该玩家碰撞/接触的玩家都会收到一个减慢其速度并改变其材质的 GameplayEffect。由于我们无法在每次发生时都发送 Server RPC（并且服务器在其模拟时间点也不一定能够处理该消息），因此无法关联客户端和服务器之间的 GameplayEffect 副作用。
>
> 这里的一种方法可能是考虑一种较弱形式的预测。这种预测不使用全新的预测键，而是服务器假定客户端将预测整个能力的所有副作用。这至少可以解决"重做"问题，但不能解决"完整性"问题。如果客户端的预测可以做得尽可能少——例如仅预测一个初始粒子效果，而不是预测状态和属性变化——那么问题就不会那么严重。
>
> 我可以设想一种弱预测模式，当没有可以准确关联副作用的新的预测键时，某些能力（所有能力？）会回退到该模式。在弱预测模式下，也许只有某些操作可以被预测——例如 GameplayCue 执行事件，但不包括 OnAdded/OnRemove 事件。
