+++
title = '如何以一种优美的方式来实现状态机，而非利用`switch case`来实现'
date = 2024-10-20T14:43:03+08:00
tag = ['State Machine']
categories = ['Study']
keywords = ['State Machine']
draft = false
summary = '为什么写这一篇文章来学习状态机？最近在工作的时候发现，公司内部的状态机实现实在是十分的“优美动人”，所以决定来学习状态机并且尝试使用一种优美的方式来实现状态机！'

+++

# 1. 写在前面

状态机在处理复杂的状态逻辑时候，传统的`switch-case`或者`if-else`逻辑虽然直接简单，但是面对状态爆炸时却难以维护。这个时候你可以选择社区当中各种开源的状态机实现：***Spring State Machine(Java)***, ***Boost Statechart(C++)***等等。但是你在使用这些开源的状态机的时候，一个很明显的优点是功能十分完备，一个很明显的缺点也是功能十分完备。很多时候我们的应用场景可能并不需要状态机的高级用法：状态机的嵌套、并行状态机或者子状态机等，需要的可能仅仅是一个**性能良好**以及可以**完成状态流转功能**的状态机而已。

在寻找如何以一种优美的方式来实现状态机的过程当中，在公司内的”高人“指引下打开了*DSL（Domain Specific Languages，领域特定语言）*世界的大门！如果您对*DSL*十分模糊，可以先翻阅一些关于*DSL*的资料进行了解。

> 简单的说，DSL是为特定领域或者问题设计的小型编程语言，专注于提供高效、直观的语法来解决特定领域中的问题。例如，用于数据库查询的SQL和用于构建状态机的SCXML都是DSL的实例。

或许你可能对DSL还是一知半解，[张建飞(@Frank)](https://blog.csdn.net/significantfrank?type=blog)写的一篇文章[实现一个状态机引擎，教你看清DSL的本质](https://blog.csdn.net/significantfrank/article/details/104996419)很清楚的讲解了什么是DSL、如何使用DSL来实现状态机都有一个很好的说明。本文后续实现的**无状态的状态机**也是参考其实现的[cola-statemachine](https://github.com/alibaba/COLA/tree/master/cola-components/cola-component-statemachine)。

其实现的状态机简单使用例子：

```java
builder.externalTransition()
          .from(States.STATE1)
          .to(States.STATE2)
          .on(Events.EVENT1)
          .when(checkCondition())
          .perform(doAction());
```

你先别急哈！你先看看书了解一下DSL的好处！了解之后再想着是不是怼我！（内心OS：我当时第一眼看到这个代码直接破口大骂，直呼“这是人能写出来的？！”）

如果你觉得这样写法十分的丑陋，当然也可以换一种接口封装的方式，这样或者还会更加的简单容易！例如后续当中我的实现：

```pascal
FSMBuilder.ExternalTransition(States.stIdle, States.stRunning, Events.evtStart, Condition,
    function: Boolean
    begin
      shpIdle.Brush.Color := clWhite;
      shpRunning.Brush.Color := clRed;
      Result := True;
    end);
```

这两种写法，每个人都有不同的看法，各取所需！我现在看来貌似是前者看着对于状态转移关系的描述更为的简练简单清晰，只不过实现起来可能需要进行更多的封装罢了！

# 2. 状态机核心概念

## 2.1 State machine, state, transition, event

> 来自[[Meta State Machine] UML Short Guide](https://www.boost.org/doc/libs/1_86_0/libs/msm/doc/HTML/ch02s02.html).

A state machine is a concrete model describing the behavior of a system. It is composed of a finite number of states and transitions.

A simple state has no sub states. It can have data, entry and exit behaviors and deferred events. One can provide entry and exit behaviors (also called actions) to states (or state machines), which are executed whenever a state is entered or left, no matter how. A state can also have internal transitions which cause no entry or exit behavior to be called. A state can mark events as deferred. This means the event cannot be processed if this state is active, but it must be retained. Next time a state not deferring this event is active, the event will be processed, as if it had just been fired.

> 对于状态是否持有数据、进入和退出行为，这取决于状态机的具体应用场景，不同的场景可能需要不同的需求，这个时候就需要针对各种不同的情况来进行不同的状态机的实现！后面在具体实现状态机的时候，因为实际应用场景简单，所以只实现最简单的状态流转！

A transition is the switching between active states, triggered by an event. Actions and guard conditions can be attached to the transition. The action executes when the transition fires, the guard is a Boolean operation executed first and which can prevent the transition from firing by returning false.

> 状态转移关系当中可以由[原状态(*source state*)，目标状态(*target state*)，转移条件(*guard conditions*)，动作(*actions*)]所组成。其中在进行状态转移的时候会先进行转移条件的检查，倘若检查通过的话才进行状态转移，否则仍旧保持当前状态。

An initial state marks the first active state of a state machine. It has no real existence and neither has the transition originating from it.

> 状态机大部分的实现都存在**初始状态**以及**当前状态**，这个也是取决于状态机的具体使用场景！
>
> - 如果系统不持有当前的状态，而是通过访问状态机的状态来得知当前系统的状态，这时候状态机的实现就需要初始状态和当前状态。状态机维护当前状态在高并发系统当中性能可能会存在一定的问题，涉及到多线程安全问题；
> - 如果系统持有当前的状态，状态机不持有当前状态以及初始状态，状态机依赖于输入条件和额外的一些信息来计算得到目标状态，这样不会发生线程竞争。

## 2.2 Conflicting transitions

If, for a given event, several transitions are enabled, they are said to be in conflict. There are two kinds of conflicts:

- For a given source state, several transitions are defined, triggered by the same event. Normally, the guard condition in each transition defines which one is fired.
- The source state is a submachine or simple state and the conflict is between a transition internal to this state and a transition triggered by the same event and having as target another state.

> **UML Short Guide**当中指出的状态转移冲突有两种，其实不仅仅只有两种，这个需要根据具体的场景需要。例如有的时候系统想要状态机根据**同一事件根据不同的条件进行不同的状态转移**；有的时候系统想要状态机存在**优先级概念**，同一事件发生的时候，某些事件需要优先进行响应，有的时候甚至需要抢占式事件响应。
>
> 我们平时使用状态机的时候难免会遇到各种各样奇怪的问题，有的问题是可以通过调整状态图来进行解决，有的不能通过调整状态图来解决的话，这个时候需要对状态机的实现进行特殊调整来适应不同的问题。

# 3. 状态机具体实现

> 本文章这里演示的状态机使用Delphi来进行实现，类似于C++！

## 3.1 状态机特点与简单使用

> 代码仓库地址: https://github.com/10-Kirito/StateMachineDSL/tree/main

- 无状态状态机；
- 仅仅具备简单的**状态流转**功能，不具备复杂的状态机功能；

下面的Demo简单展示了目前状态机的配置过程，其中我们需要先定义状态和事件，使用枚举定义相应的状态和事件即可：

```pascal
type
  Events = (evtStart, evtStop, evtResume, evtPause);
  States = (stIdle, stRunning, stPaused, stStopped);
```

状态和事件定义之后，根据绘制的状态图来进行状态机的配置，其中关于`ExternalTransition`函数的声明如下:

```pascal
TAction = TFunc<Boolean>;
TCondition = TFunc<Boolean>;
/// <summary>外部状态转移接口</summary>
/// <param name="ASource"> 源状态</param>
/// <param name="ATarget"> 目标状态</param>
/// <param name="AEvent"> 触发的事件</param>
/// <param name="ACondition"> 状态转移的条件</param>
/// <param name="Action"> 目标状态对应的动作</param>
procedure ExternalTransition(ASource: S; ATarget: S; AEvent: E; ACondition: TCondition; Action: TAction);
```

 其中条件和动作均为函数`TFunc<Boolean>`，所以我们在配置状态机的时候可以直接通过匿名函数配置相关的条件以及动作。

```pascal
procedure TDemo.InitialStateMachine;
begin
  FSMBuilder := TStateMachineBuilder<States, Events>.Create;

  /// 配置状态机
  FSMBuilder.ExternalTransition(States.stIdle, States.stRunning, Events.evtStart, Condition,
    function: Boolean
    begin
      shpIdle.Brush.Color := clWhite;
      shpRunning.Brush.Color := clRed;
      Result := True;
    end);
  FSMBuilder.ExternalTransition(States.stRunning, States.stPaused, Events.evtPause, Condition,
    function: Boolean
    begin
      shpRunning.Brush.Color := clWhite;
      shpPaused.Brush.Color := clRed;
      Result := True;
    end);
  FSMBuilder.ExternalTransition(States.stPaused, States.stRunning, Events.evtResume, Condition,
    function: Boolean
    begin
      shpPaused.Brush.Color := clWhite;
      shpRunning.Brush.Color := clRed;
      Result := True;
    end);
  FSMBuilder.ExternalTransition(States.stRunning, States.stStopped, Events.evtStop, Condition,
    function: Boolean
    begin
      shpRunning.Brush.Color := clWhite;
      shpStopped.Brush.Color := clRed;
      Result := True;
    end);

  /// 构建状态机
  FStateMachine := FSMBuilder.Build;
end;
```

配置并且构建状态机完成之后，触发事件可以这样：

```pascal
procedure TDemo.btnEvStartClick(Sender: TObject);
begin
  Assert(FStateMachine.FireEvent(stIdle, evtStart) = stRunning, '状态[stIdle -> stStart]失败');
end;
```

其中`FireEvent`函数的声明为：

```pascal
/// <summary> 状态机提供的触发事件的接口</summary>
/// <param name="AEvent"> 事件</param>
/// <returns> 返回触发事件导致转移的目标状态</returns>
function FireEvent(ASource: S; AEvent: E): S;
```

## 3.2 状态的实现

这里的状态的实现对于使用状态机的人来讲是看不到具体状态类的，其使用的时候只需要使用枚举定义相应的状态，最后将枚举类型当作泛型的参数传入进来即可。而状态机内部的实现为了方便拓展，内部将状态封装为相应的泛型类。

```pascal
type
	TState<S, E> = class
  strict private
    FStateId: S;
    FEventTransition: TEventTransition<S, E>;
  public
    constructor Create(AStateId: S);
    destructor Destroy; override;

    /// <summary></summary>
    /// <param name="AEvent"></param>
    /// <param name="ATarget"></param>
    /// <param name="ATransitionType"></param>
    /// <returns></returns>
    function AddTranstition(AEvent: E; ATarget: TState<S, E>;
      ATransitionType: TTransitionType): TTransition<S, E>;

    function GetEventTranstitions(AEvent: E): TObjectList<TTransition<S,E>>;
    property StateId: S read FStateId;
  end;
```

该状态类创建的时候会将枚举值传入构造函数作为该状态的唯一标识符！该状态类当中还存储有该状态下所有的状态转移关系，之后向外提供添加状态转移关系以及通过事件访问状态转移关系的函数！

## 3.3 事件的实现

事件的实现即为使用状态机的人定义的事件枚举类型，内部并未对其进行二次封装，仅仅是简单的枚举类型！

## 3.4 状态转移的实现

每一个状态类`TState<S, E>`其中的一个成员为`FEventTransition: TEventTransition<S, E>`。

这里对状态转移进行了进一步的封装，其中基本的状态转移关系：

```pascal
type
	TTransition<S, E> = class
  strict private
    FSource: TState<S, E>;
    FTarget: TState<S, E>;
    FEvent: E;
    FCondition: TCondition;
    FType: TTransitionType;
    FAction: TAction;
  private
    /// <summary> 获取相应的状态转移条件</summary>
    function GetCondition: TCondition;
  public
    constructor Create;
    destructor Destroy; override;

    /// <summary> 比较两个状态转移是否相同</summary>
    /// <param name="ATransition">目标状态转移</param>
    /// <returns></returns>
    /// <remarks>唯一标识组: [Event, Source, Target]</remarks>
    function Equal(const ATransition: TTransition<S, E>): Boolean;

    /// <summary> 状态转移函数</summary>
    function Transit: TState<S, E>;

    /// <summary> 相关属性设置以及访问</summary>
    property Source: TState<S, E> read FSource write FSource;
    property Target: TState<S, E> read FTarget write FTarget;
    property Event: E read FEvent write FEvent;
    property Condition: TCondition read GetCondition write FCondition;
    property TransType: TTransitionType read FType write FType;
    property Action: TAction read FAction write FAction;
  end;
```

状态转移关系当中存储了状态转移关系当中所有必要的元素，其中可以进一步补充。其中重要的就是`FSource`, `FTarget`, `FEvent`, `FCondition`, `FType`(这里原本的设计是状态转移可以为<u>内部状态转移</u>或者<u>外部状态转移</u>), `FAction`.

其中这里两个状态转移关系的比较只取决于**原状态**，**目标状态**以及**触发的事件**。

```pascal
function TTransition<S, E>.Equal(const ATransition: TTransition<S, E>): Boolean;
begin
  Result := (FEvent = ATransition.Event) and
    (FSource.StateId = ATransition.Source.StateId) and
    (FTarget.StateId = ATransition.Target.StateId);
end;
```

这里取决于状态机的设计来决定。

例如，有的状态机设计需要对于遇到的同一个事件根据不同`condition`触发不同的状态流转的业务场景：

```pascal
 if condition == "1", STATE1 --> STATE1
 if condition == "2" , STATE1 --> STATE2
 if condition == "3" , STATE1 --> STATE3
```

该种情况，对于状态转移关系的实现不再能是简单的几个值。而是需要考虑同一个事件对于一个源状态可能存在多个状态转移关系，这个时候你就得考虑哪些元可以唯一确定一个状态转移关系。

对于基本的状态转移关系，这里还做了进一步的封装：

```pascal
type
	TEventTransition<S, E> = class
  strict private
    FEventTransition: TObjectDictionary<E,TObjectList<TTransition<S,E>>>;

    /// <summary> 同一个事件, 两个状态之间的状态转移只能存在一个, 不能存在多个</summary>
    /// <param name="AList"> 一个事件对应的所有的状态转移</param>
    /// <param name="ATransition"> 新的目标状态转移</param>
    procedure Verify(AList: TObjectList<TTransition<S, E>>; ANewTransition: TTransition<S, E>);
  public
    constructor Create;
    destructor Destroy; override;

    /// <summary> 添加一个状态转移，一个事件可能对应多种状态转移</summary>
    /// <param name="AEvent"> 事件</param>
    /// <param name="ATransition"> 状态转移</param>
    procedure Put(AEvent: E; ANewTransition: TTransition<S, E>);

    /// <summary> 当事件触发的时候，返回该事件可以触发的所有的状态转移</summary>
    function Get(AEvent: E): TObjectList<TTransition<S,E>>;
  end;
```

这里提供了两个函数`Put`和`Get`。其中对于`Put`, 添加状态转移的时候，一个事件可能对应多个状态转移！对于`Get`函数，当事件触发的时候，返回该事件可以触发的所有的状态转移关系。

## 3.5 状态机`Builder`实现

状态机的相关配置，例如状态转移关系的添加这些职责由状态机`Builder`来负责：

```pascal
TStateMachineBuilder<S, E> = class
  {$REGION '私有类型声明'}
  strict private type
    StateHeler<S1, E1> = class
    public
      /// <summary> 创建相应的状态，但是并不持有其生命周期</summary>
      /// <remarks> 从StateMap当中获取响应的状态，如果存在则直接返回; 反之则创建相应的状态</remarks>
      class function getState(var AMap: TObjectDictionary<S1, TState<S1, E1>>; AStateId: S1): TState<S1, E1>;
    end;
  {$ENDREGION}
  strict private
    {
      StateMap仅仅是持有相关对象的引用，其生命周期交由状态机来进行管理！
      为什么？因为Builder的存在时间周期是比较短的，而状态机是始终存在!
    }
    FStateMap: TObjectDictionary<S, TState<S, E>>;
    FStateMachine: TStateMachine<S, E>;
  public
    constructor Create;
    destructor Destroy; override;

    /// <summary>外部状态转移接口</summary>
    /// <param name="ASource"> 源状态</param>
    /// <param name="ATarget"> 目标状态</param>
    /// <param name="AEvent"> 触发的事件</param>
    /// <param name="ACondition"> 状态转移的条件</param>
    /// <param name="Action"> 目标状态对应的动作</param>
    procedure ExternalTransition(ASource: S; ATarget: S; AEvent: E;
      ACondition: TCondition; Action: TAction);

    /// <summary> 获取构建的状态机</summary>
    /// <remarks> 并不管理其生命周期，交由用户进行管理</remarks>
    function Build: TStateMachine<S, E>;
  end;
```

其中关于动态创建出来的`TState<S, E>`的声明周期交由具体创建出来的状态机来管理！

## 3.5 状态机实现

状态机持有一个简单的`<S, TState<S, E>>`映射关系，其中所有的状态转移关系全部存放在具体的状态类当中。状态机当中对外仅仅提供一个基本的触发事件的接口，其他对于使用者均不可见！

```pascal
type
	/// <summary>
  ///   状态机类
  /// </summary>
  TStateMachine<S, E> = class
  strict private
    FStateMap: TObjectDictionary<S, TState<S, E>>;

    /// <summary> 获取当前给定<S, E>所对应的Transition</summary>
    /// <param name="ASource"> 当前状态: SourceState</param>
    /// <param name="AEvent"> 触发的事件: Event</param>
    /// <returns> 状态转移: TTransition</returns>
    /// <remarks> 其中在进行状态转移的时候，会进行相关条件的判断, 符合条件才可以进行状态转移</remarks>
    function RouteTransition(ASource: S; AEvent: E): TTransition<S, E>;

    /// <summary> 获取相应的状态</summary>
    /// <remarks> 如果相应状态不存在，则抛出异常</remarks>
    function GetState(AStateId: S): TState<S, E>;
  public
    constructor Create(var AStateMap: TObjectDictionary<S, TState<S, E>>);
    destructor Destroy; override;

    /// <summary> 状态机提供的触发事件的接口</summary>
    /// <param name="AEvent"> 事件</param>
    /// <returns> 返回触发事件导致转移的目标状态</returns>
    function FireEvent(ASource: S; AEvent: E): S;
  end;
```

其中关于事件的触发具体是由函数`FireEvent`以及`RouteTransition`来完成。

```pascal
function TStateMachine<S, E>.FireEvent(ASource: S; AEvent: E): S;
var
  LTransition: TTransition<S, E>;
begin
  LTransition := RouteTransition(ASource, AEvent);

  /// 如果不存在相应的状态转移，则维持当前状态
  if not Assigned(LTransition) then
  begin
    CodeSite.Send('该事件并未存在相关的状态转移！');
    Exit(ASource);
  end;

  Result := LTransition.Transit.StateId;
end;

function TStateMachine<S, E>.RouteTransition(ASource: S; AEvent: E): TTransition<S, E>;
var
  LState: TState<S, E>;
  LTransitionList: TObjectList<TTransition<S, E>>;
  LTransition: TTransition<S, E>;
begin
  Result := nil;
  LState := GetState(ASource);
  LTransitionList := LState.GetEventTranstitions(AEvent);

  /// 如果该事件并没有对应的状态转移，则直接返回nil
  if (not Assigned(LTransitionList)) or (LTransitionList.Count = 0) then
    Exit(nil);

  /// - 如果相应的状态转移提前设置Condition, 则检查相关的Condition是否满足状态转移;
  /// - 如果相应的状态转移没有设置Condition, 则直接返回最后一个状态转移(一般情况下
  ///   用户没有设置相应的状态转移条件，一个事件对于一个状态，只能有一个结果状态)
  for LTransition in LTransitionList do
  begin
    if not Assigned(LTransition.Condition) then
       Result := LTransition
    else if LTransition.Condition() then
    begin
      Result := LTransition;
      Exit;
    end;
  end;
end;
```

# 4. 结尾

以上便是一个简单的无状态的状态机具体概念以及实现， 感谢您有如此的耐心来阅读！另外本文章当中实现语言为Delphi，后续会在仓库当中更新其他类型的语言！

