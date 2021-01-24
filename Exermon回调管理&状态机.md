# 回调管理&状态机
## 基本模型

在程序中，经常会用到回调函数，比如在网络系统中，可能会有连接、断开、异常处理等回调函数，而在战斗系统中，回调函数更是常见，比如：战斗开始、回合开始、发起攻击、受到攻击、获得状态、解除状态等，战斗中的一些被动效果就是通过在这些回调函数中添加代码判断来实现的。

部分回调函数还需要附带参数，比如回合开始，需要附带当前回合数的参数；获得状态，需要附带当前状态的参数，而且不同回调函数的参数类型也可以不同。

状态机同理，这里就不重复描述了（减少重复代码）

## 存在问题

一般来说，我们可以直接调用回调函数就能实现。但是当我们有多个对象需要回调的时候，就需要添加很多重复代码（手动调用每个对象对应的回调函数）

比如：有三个对象`A`，`B`，`C`（他们的类不同），他们都有`onA`, `onB`, `onC`三个方法，其中，`A`包含了`B`和`C`（组合关系或者聚合关系）。在这里，我希望调用`A`的`onA`方法时候也能调用`B`和`C`的`onA`，这个逻辑我直接写到代码里即可。但是对于`onB`, `onC`方法我也想要有同样的效果，这样我就要写3份相同逻辑的代码。如果回调函数越多，那么我重复的逻辑就越多……甚至，如果后面加入了一个`D`对象，也需要有类似的逻辑，我还得在每个回调函数中改，大大增加重复的工作量。

注：在后面的特性处理器（`TraitProcessor`）里就会通过实例讲到这种情况。

因此，在开发这个战斗模型过程中，我花时间写了一个回调管理的类，状态机的类也是同理。

注：这个模式有点类似Unity的`SendMessage`，但是`SendMessage`是在运行时执行反射（Reflection），速度是非常慢的，且依赖Unity的环境。我们希望做一个通用的高效的工具类，希望在其他C#项目中也能使用，或者探讨出一个通用的解决思路，在其他语言也能实现。

## 目标

存在的问题已经清楚了，我们就需要找到一个解决方案。

在找解决方案时候，先不要想底层实现，先想我希望怎么样，就是我的目标是什么。

在这里，我的目标是：

- 定义一个枚举（`Enum`），声明回调的类型有哪些
- 写一个回调管理类（`CallbackManager`），我将通过该类解决上述问题
- 将希望回调的对象（可以有多个）注册到`CallbackManager`中，下面简称注册对象
- `CallbackManager`有一个`on`函数，传入一个枚举值调用这个函数，即可自动调用所有注册对象的对应函数
- 这个`on`函数还可以带其他参数，带参数后也会用这些参数去调用所有注册对象的对应参数类型的函数
- 对于上面两点，如果有的注册对象没有声明对应的函数，则跳过
- `CallbackManager`还可以单独为某个回调类型注册对应的回调函数

用伪代码来表示：

``` CSharp
// 类和枚举定义
enum Type { A, B, C }

class A { 
    void onA() {}
	void onB() {}
	void onC() {}

	void onA(int a) {}
}

class B { 
	void onA() {}
	void onB() {}

	void onA(int a) {}
}

class C { 
	void onA() {}
	void onC() {}
}

// 执行的代码
var a = new A();
var b = new B();
var c = new C();

var cbManager = new CallbackManager();

// 使用枚举类型来配置这个CallbackManager，后面实现中会讲到
cbManager.setupType(typeof(Type));

// 注册对象
cbManager.registerObject(a);
cbManager.registerObject(b);
cbManager.registerObject(c);

// 调用
cbManager.on(Type.A); // 调用 a, b, c 的 onA 方法
cbManager.on(Type.B); // 调用 a, b 的 onB 方法
cbManager.on(Type.C); // 调用 a, c 的 onC 方法

cbManager.on(Type.A, 10); // 调用 a, b 的 onA(int) 方法

```

## 实现

这个目标，其实有几个思路可以实现。最简单的是模仿Unity的`SendMessage`函数，在`on`的时候通过反射找到每个注册对象的指定名称指定参数的方法并调用。
但是这个方法的问题就是反射操作会大大降低效率，如果每次`on`都进行反射，运行速度就很慢。

既然每次`on`都进行反射，且一般情况下注册的对象不会变化，那么为什么不在注册对象的时候就进行反射，找到对应的回调函数之后缓存起来呢？

如何缓存？我们可以这样想：由于一次`on`的时候会调用多个回调函数，即：执行一次`on`函数，会调用多个`UnityAction`。我们提前把这些`UnityAction`缓存起来，`on`的时候就不需要通过反射获得了。

这个缓存还得根据回调类型进行分组。为了简化代码，我定义了一个`CallbackItem`的类，来管理特定的回调项（管理特定一个回调类型的回调数据）
该阶段下，整体代码大致如下：

``` CSharp
public class CallbackItem {

	string name; // 回调类型名称

	// 由于其他地方的问题，这里不使用构造函数，改用 setup 函数进行初始化（手动调用），具体可以查看工程代码中的 DictionaryUtils.cs
	// public CallbackItem(string name) { this.name = name; }
	public void setup(string name) { this.name = name; }

	public void registerObject(object obj) {
		// 通过反射遍历 obj 的每个函数，找到合适的进行注册
	}
	public void remvoeObject(object obj) {
		// 注销
	}

	public void on(params object params_) {
		// 执行回调
	}

}

public class CallbackManager {

	Dictionary<string, CallbackItem> callbacks = new Dictionary<string, CallbackItem>();

	public void registerObject(object obj) {
		foreach(var callback in callbacks) callback.registerObject(obj);
	}
	public void remvoeObject(object obj) {
		foreach(var callback in callbacks) callback.remvoeObject(obj);
	}

	public void setupType(Type type) {
		// 通过遍历 type 的每一个枚举值，创建对应的 CallbackItem 并存入 callbacks 中（调用 createCallbackItem 函数）
		// 这里在后面会有拓展
		foreach (var name in Enum.GetNames(bindingType))
			createCallbackItem(name);
	}

	public CallbackItem getCallbackItem(string name) {
		if (callbacks.ContainsKey(name)) return callbacks[name];
		return null;
	}

	public CallbackItem createCallbackItem(string name) {
		callbacks[name] = new CallbackItem();
		callbacks[name].setup(name);
		return callbacks[name];
	}

	public void on(Enum type, params object params_) {
		on(type.ToString(), params_);
	}
	public void on(string name, params object params_) {
		// 获取对应的 CallbackItem 执行 on 操作
		getCallbackItem(name)?.on(params_);
	}
}
```

在`CallbackItem`中，为了方便，我们可以用`UnityEventBase`来“缓存”这些回调函数，这也更符合我们的直觉。（`UnityAction`可以用`Action`代替，`UnityEventBase`用起来可以看成是`UnityAction`的数组，手动实现也比较简单，所以在这里我们不把他看作Unity环境）

这里我简单介绍一下`UnityEventBase`，实际上我们常用的`UnityEvent`就是`UnityEventBase`的子类。`UnityEventBase`派生的还有：`UnityEvent<T0>` ~ `UnityEvent<T0, T1, T2, T3>`，即这个`UnityEventBase`体系最多只支持4个参数，详细可以看官方文档：https://docs.unity3d.com/ScriptReference/Events.UnityEvent.html

PS：这里我想了很久，一直想找一个能够通用的方法（可以支持任意个参数，同时可以实现缓存机制），但是都没想到好的办法。不过仔细一想，实际上4个参数已经完全够用了，实在不够用也绝对不是我的问题0.0

我们现在打算使用`UnityEventBase`缓存这些函数，而这些函数的参数类型是可以各不相同的，所以我们需要根据函数参数的类型动态获取`UnityEventBase`类型和创建他的的实例，代码如下（由于带泛型的`UnityEvent`是抽象类，我这里全部用`ExerEvent`继承了一遍）：

``` CSharp
/// <summary>
/// 内部定义的UnityEvent
/// </summary>
/// <typeparam name="T"></typeparam>
class ExerEvent : UnityEvent { }
class ExerEvent<T0> : UnityEvent<T0> { }
class ExerEvent<T0, T1> : UnityEvent<T0, T1> { }
class ExerEvent<T0, T1, T2> : UnityEvent<T0, T1, T2> { }
class ExerEvent<T0, T1, T2, T3> : UnityEvent<T0, T1, T2, T3> { }

/// <summary>
/// 获取事件类型
/// </summary>
/// <param name="types"></param>
/// <returns></returns>
static Type getEventType(Type[] types) {
	switch (types.Length) {
		case 0: return typeof(ExerEvent);
		case 1: return typeof(ExerEvent<>).MakeGenericType(types);
		case 2: return typeof(ExerEvent<,>).MakeGenericType(types);
		case 3: return typeof(ExerEvent<,,>).MakeGenericType(types);
		case 4: return typeof(ExerEvent<,,,>).MakeGenericType(types);
		default: return null;
	}
}
```

同样，由于函数参数各不相同，我们需要有一个管理各组参数及对应的一个`UnityEventBase`的数据结构（即一组`Type`对应一个`UnityEventBase`）

为了更好理解，请看以下示意图：

![avatar][events]

这里，最外层的是一个类型为`List<Tuple<Type[], UnityEventBase>>`的`events`变量，该变量用于管理`Type[]`和`UnityEventBase`的对应关系。

注：这里一开始是打算用`Dictionary<Type[], UnityEventBase>`实现，但是因为`Type[]`在`Dictionary`中并非比较内部元素，不能保证唯一性，因此改用`List+Tuple`的方式进行管理。

`events`里面每一个元素都是一个元组，`Item1`为`Type[]`，即回调函数的参数类型数组；`Item2`为`UnityEventBase`，即该参数类型数组所对应的`UnityEventBase`实例。而一个`UnityEventBase`又可以注册多个`UnityAction`，整个`CallbackItem`就是基于这个来运行的。

这个数据结构是怎么使用的呢？

1. 首先，在进行注册对象（`registerObject`）的时候，会遍历该对象的每个函数，找到合适的函数进行注册（合适的函数指的是按照一定规则命名的函数，在这里的命名规则为`on+回调类型名称`，见下面代码）

2. 进行注册函数（`registerMethod`）的时候，会获取该函数的参数类型，然后从`events`查找对应的`UnityEventBase`。如果找不到，动态创建一个实例。

3. 找到`UnityEventBase`后，用对应类型的`UnityAction`创建当前函数的`Delegate`

4. 用`UnityEvent.AddListener`注册这个`Delegate`

这时候`CallbackItem`类为：

``` CSharp
public class CallbackItem {

	/// <summary>
	/// 最大参数数量
	/// </summary>
	const int MaxParamCount = 4;

	string name; // 回调类型名称

	string methodName => "on" + name;

	List<Tuple<Type[], UnityEventBase>> events = new List<Tuple<Type[], UnityEventBase>>();

	// 由于其他地方的问题，这里不使用构造函数，改用 setup 函数进行初始化（手动调用），具体可以查看工程代码中的 DictionaryUtils.cs
	// public CallbackItem(string name) { this.name = name; }
	public void setup(string name) { this.name = name; }

	public void registerObject(object obj) {
		// 通过反射遍历 obj 的每个函数，找到合适的进行注册
		var oType = obj.GetType();
		// ReflectionUtils.DefaultFlags 定义在工程的 ReflectionUtils 中
		// 在反射时候添加了这个 flags 之后就可以通过反射获取到非公有成员
		var methods = oType.GetMethods(ReflectionUtils.DefaultFlags);

		foreach(var method in methods) {
			if (method.Name != methodName) continue; // 判断函数名
			registerMethod(obj, method);
		}

	}
	public void remvoeObject(object obj) {
		// 注销
	}

	public void registerMethod(object obj, MethodInfo method) {
		// 注册一个函数
		var params_ = method.GetParameters();
		if (params_.Length > MaxParamCount)
			throw new Exception("参数超出限制");

		// CallbackManager.getTypes 是一个工具函数，可以从 params_ 获取 Type[]，即由参数列表获取参数类型数组
		var types = CallbackManager.getTypes(params_);

		var action = getAction(obj, method); // 获取 Delegate（即 UnityAction ）
		var event_ = getEvent(types); // 获取 UnityEventBase，这里需要注意类型

		// 这里使用反射是因为在 UnityEventBase 中没有我们要调用的那个 AddListener 函数
		var eType = event_.GetType();
		var process = eType.GetMethod("AddListener");

		process.Invoke(event_, new object[] { action }); // 添加回调函数
	}

	Delegate getAction(object obj, MethodInfo method) {
		var params_ = method.GetParameters();
		if (params_.Length > MaxParamCount) return null;

		var types = CallbackManager.getTypes(params_);
		return getAction(obj, method, types);
	}
	Delegate getAction(object obj, MethodInfo method, Type[] types) {
		var aType = getActionType(types);

		return method.CreateDelegate(aType, obj); // 创建 Delegate
	}

	UnityEventBase getEvent(Type[] types, bool create = true) {
		var tuple = events.Find(t => isTypesEquals(t.Item1, types)); // 这里遍历搜索相同参数类型数组的值

		if (tuple == null && create) {
			var eType = getEventType(types); // event 的真实类型
			var event_ = Activator.CreateInstance(eType) as UnityEventBase;

			events.Add(tuple = new Tuple<Type[], UnityEventBase>(types, event_));
		}

		return tuple?.Item2;
	}
	
	static bool isTypesEquals(Type[] types1, Type[] types2) {
		if (types1.Length != types2.Length) return false;
		for (int i = 0; i < types1.Length; ++i)
			if (types2[i] != types1[i]) return false;
		return true;
	}

	public void on(params object params_) {
		// 执行回调
	}

}
```

有了上面的代码，如何通过`on`来唤起一个回调就很明显了，这里直接给出代码（`CallbackItem`类）：

``` CSharp
	/// <summary>
	/// 唤起
	/// </summary>
	public void on(params object[] params_) {
		var types = CallbackManager.getTypes(params_);
		var eType = getEventType(types);
		// 使用反射获取 UnityEvent.Invoke 函数
		var invoke = eType.GetMethod("Invoke", types);

		var event_ = getEvent(types);

		// 调用
		invoke?.Invoke(event_, params_);

		onInvoked();
	}
	// 由于只有4个参数，这里全部写出来也无妨，可以避免使用反射，加快速度
	public void on() {
		var event_ = getEvent<ExerEvent>();
		event_?.Invoke();
		onInvoked();
	}
	public void on<T0>(T0 p1) {
		var event_ = getEvent<ExerEvent<T0>>();
		event_?.Invoke(p1);
		onInvoked();
	}
	public void on<T0, T1>(T0 p1, T1 p2) {
		var event_ = getEvent<ExerEvent<T0, T1>>();
		event_?.Invoke(p1, p2);
		onInvoked();
	}
	public void on<T0, T1, T2>(T0 p1, T1 p2, T2 p3) {
		var event_ = getEvent<ExerEvent<T0, T1, T2>>();
		event_?.Invoke(p1, p2, p3);
		onInvoked();
	}
	public void on<T0, T1, T2, T3>(T0 p1, T1 p2, T2 p3, T3 p4) {
		var event_ = getEvent<ExerEvent<T0, T1, T2, T3>>();
		event_?.Invoke(p1, p2, p3, p4);
		onInvoked();
	}

	void onInvoked() { } // 统一处理回调后的逻辑
```

## 拓展

现在，`CallbackItem`只能自动注册`on....`这类型的函数，而且使用也比较局限。其实只要进行简单的拓展，我们就能将这个功能通用化。

直接上代码：

``` CSharp
public abstract class BaseCallbackItem<T> {

	protected T name;

	public void setup(T name) { this.name = name; }

	protected virtual string methodName => "";

	void processObject(object obj, bool add) {
		if (methodName == "") return;

		var oType = obj.GetType();
		var methods = oType.GetMethods(ReflectionUtils.DefaultFlags);

		foreach (var method in methods) {
			if (method.Name != methodName) continue;
			processMethod(obj, method, add);
		}
	}

	// 其他代码省略，只写出改动的地方
}

// 原来的 CallbackItem 改为这样：
public class CallbackItem : BaseCallbackItem<string> {

	protected override string methodName => "on" + name;

}
```

这部分的完整代码可以在我们的工程里查看，Gitee地址：
https://gitee.com/jyanon/exermon2-client/tree/master/Exermon2/Assets/Scripts/Core/Utils/Callback

注1：如果没有权限先找我邀请你加入

注2：代码还没有进行测试，实际运行可能会有bug，但总体设计思路表达出来了就行。

## 状态机

状态机实际上跟回调管理是类似的。在我们的工程中，对于一个状态，状态机有4种回调：

1. enter回调，状态进入时候自动执行（自动注册`on+状态名+Enter`函数）
2. update回调，每帧执行（自动注册`update+状态名`函数）
3. exit回调，状态退出时候执行（自动注册`on+状态名+Exit`函数）
4. change回调，状态切换到另一个指定状态时候执行（需手动注册函数）

其中，前三种回调方式可以通过上面的拓展，就能实现自动注册：

``` CSharp
public class StateMachine {

	/// <summary>
	/// 更新回调项
	/// </summary>
	public class UpdateCallbackItem : BaseCallbackItem<string> {

		/// <summary>
		/// 函数名
		/// </summary>
		protected override string methodName => "update" + name;

	}

	/// <summary>
	/// 进入回调项
	/// </summary>
	public class EnterCallbackItem : BaseCallbackItem<string> {

		/// <summary>
		/// 函数名
		/// </summary>
		protected override string methodName => "on" + name + "Enter";

	}

	/// <summary>
	/// 退出回调项
	/// </summary>
	public class ExitCallbackItem : BaseCallbackItem<string> {

		/// <summary>
		/// 函数名
		/// </summary>
		protected override string methodName => "on" + name + "Exit";

	}

	/// <summary>
	/// 切换回调项（需要手动注册）
	/// </summary>
	public class ChangeCallbackItem : BaseCallbackItem<Tuple<string ,string>> { }


	Dictionary<string, UpdateCallbackItem> updateCallbacks = 
		new Dictionary<string, UpdateCallbackItem>();
	Dictionary<string, EnterCallbackItem> enterCallbacks = 
		new Dictionary<string, EnterCallbackItem>();
	Dictionary<string, ExitCallbackItem> exitCallbacks = 
		new Dictionary<string, ExitCallbackItem>();

	Dictionary<Tuple<string ,string>, ChangeCallbackItem> changeCallbacks = 
		new Dictionary<Tuple<string ,string>, ChangeCallbackItem>();

	public void registerObject(object obj) {
		foreach(var callback in updateCallbacks) callback.registerObject(obj);
		foreach(var callback in enterCallbacks) callback.registerObject(obj);
		foreach(var callback in exitCallbacks) callback.registerObject(obj);
	}
	public void remvoeObject(object obj) {
		foreach(var callback in updateCallbacks) callback.remvoeObject(obj);
		foreach(var callback in enterCallbacks) callback.remvoeObject(obj);
		foreach(var callback in exitCallbacks) callback.remvoeObject(obj);
	}

	public void setupType(Type type) {
		// 通过遍历 type 的每一个枚举值，创建对应的 CallbackItem 并存入每一个 callbacks 中
		// 这里在后面会有拓展
		foreach (var name in Enum.GetNames(bindingType))
			createCallbackItems(name);
	}

	public void createCallbackItems(string name) {
		updateCallbacks[name] = new UpdateCallbackItem();
		updateCallbacks[name].setup(name);

		enterCallbacks[name] = new EnterCallbackItem();
		enterCallbacks[name].setup(name);

		exitCallbacks[name] = new ExitCallbackItem();
		exitCallbacks[name].setup(name);
	}
}
```

状态机的其他实现其实比较简单，可以自行探索。具体的代码请查看工程里的`Exermon2/Assets/Scripts/Core/Utils/Callback/StateMachine.cs`文件。
Gitee地址为：https://gitee.com/jyanon/exermon2-client/blob/master/Exermon2/Assets/Scripts/Core/Utils/DictionaryUtils.cs

## 字典拓展

在上面的代码中（特别是`StateMachine`的代码），涉及到很多字典的操作。为了简化这些操作，可以对C#的字典进行封装和拓展。具体请自行查看工程里的`Exermon2/Assets/Scripts/Core/Utils/DictionaryUtils.cs`文件。
Gitee地址为：https://gitee.com/jyanon/exermon2-client/blob/master/Exermon2/Assets/Scripts/Core/Utils/DictionaryUtils.cs

## 枚举拓展

注：建议大致浏览了上述的工程代码之后再看这部分内容

由于枚举类型（`enum`）无法被继承，在后面的战斗系统中，为了保证框架的可拓展性，一些地方不再使用枚举，改用`EnumExtend`类来代替。

而上述的`CallbackManager`中有一个`setupType`函数来配置这个枚举，在`StateMachine`中也有这个函数。所以我们将其提取出来，定义一个新的基类：`EnumTypeProcessor`，专门用来处理这类型的逻辑（需要遍历每个枚举值，去实现一些功能）

这里直接给出工程里的源代码（有修改）（代码位于`Exermon2/Assets/Scripts/Core/Utils/Enum/EnumTypeProcessor.cs`）：

``` CSharp
using System;
using System.Collections.Generic;
using System.Reflection;

namespace Core.Utils {

	/// <summary>
	/// 枚举类型
	/// </summary>
	public abstract class EnumExtend { }

	/// <summary>
	/// 枚举/类型处理类
	/// </summary>
	public abstract class EnumTypeProcessor {

		/// <summary>
		/// 绑定的类型
		/// </summary>
		protected Type bindingType { get; set; }

		/// <summary>
		/// 配置类型
		/// </summary>
		/// <param name="type"></param>
		public void setupType(Type type, bool bind = false) {
			if (bind) bindingType = type;

			if (type == null) return;
			if (type.IsEnum) setupEnumType(type);
			else if(type.IsSubclassOf(typeof(EnumExtend)))
				setupEnumExtendType(type);
		}

		/// <summary>
		/// 配置普通类型
		/// </summary>
		/// <param name="type"></param>
		void setupEnumExtendType(Type type) {
			// 这里涉及到 ReflectionUtils，代码经过重重封装，比较复杂，这里不详细解释
			// 只需要直到这里的功能是遍历每一个字段，并进行处理即可
			ReflectionUtils.processMember<FieldInfo>(bindingType,
				typeof(string), f => { // 对每个字段进行遍历
					var val = f.GetValue(null) as string;
					if (val == null) f.SetValue(null, val = f.Name);

					processValue(val);
				}, flags: ReflectionUtils.DefaultStaticFlags);
		}

		/// <summary>
		/// 配置普通类型
		/// </summary>
		/// <param name="type"></param>
		void setupEnumType(Type type) {
			foreach (var name in Enum.GetNames(bindingType))
				processValue(name);
		}

		/// <summary>
		/// 处理值
		/// </summary>
		/// <param name="name"></param>
		protected abstract void processValue(string val);

		/// <summary>
		/// 构造函数
		/// </summary>
		public EnumTypeProcessor() { }
		public EnumTypeProcessor(Type type) {
			setupType(type, true);
		}

	}
}
```

进行拓展之后，上面的`CallbackManager`和`StateMachine`可以改写成：

``` CSharp

public class CallbackManager : EnumTypeProcessor {

	/// <summary>
	/// 处理值
	/// </summary>
	/// <param name="val"></param>
	protected override void processValue(string val) {
		createCallbackItem(val);
	}

	public CallbackManager() {}
	public CallbackManager(Type type) : base(type) {}

	// 删除 setupType 函数，其他代码省略
}

public class StateMachine : EnumTypeProcessor {

	/// <summary>
	/// 处理值
	/// </summary>
	/// <param name="val"></param>
	protected override void processValue(string val) {
		createCallbackItems(val);
	}

	public StateMachine() {}
	public StateMachine(Type type) : base(type) {}

	// 删除 setupType 函数，其他代码省略
}

```

对这个进行拓展之后，最开始的实例里的枚举就写成：

``` CSharp

// 原代码：
// 类和枚举定义
enum Type { A, B, C }

// 新代码：
public class Type : EnumExtend {
	public static string A, B, C;
}

```

使用方法跟枚举一致，而且还可以进行继承，有很好的拓展性。

## 实例及预告

下一次技术分享将在这两个概念的基础上，讲解战斗系统，包括战斗者、战斗控制、行动效果、战斗BUFF、战斗状态、战斗特性等内容。

[events]:data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAgkAAAJjCAYAAACGDYLEAAAHjXRFWHRteGZpbGUAJTNDbXhmaWxlJTIwaG9zdCUzRCUyMmFwcC5kaWFncmFtcy5uZXQlMjIlMjBtb2RpZmllZCUzRCUyMjIwMjEtMDEtMjRUMDYlM0EyOCUzQTE3LjE0NlolMjIlMjBhZ2VudCUzRCUyMjUuMCUyMChXaW5kb3dzJTIwTlQlMjAxMC4wJTNCJTIwV09XNjQpJTIwQXBwbGVXZWJLaXQlMkY1MzcuMzYlMjAoS0hUTUwlMkMlMjBsaWtlJTIwR2Vja28pJTIwQ2hyb21lJTJGNzguMC4zOTA0LjEwOCUyMFNhZmFyaSUyRjUzNy4zNiUyMiUyMGV0YWclM0QlMjJHeWRaR2JGemdyMGlOVXd3bzNCVCUyMiUyMHZlcnNpb24lM0QlMjIxNC4yLjclMjIlMjB0eXBlJTNEJTIyZGV2aWNlJTIyJTNFJTNDZGlhZ3JhbSUyMGlkJTNEJTIyUTI5R0ZUYTRNTzc3UXQ5ZkRRZ1klMjIlMjBuYW1lJTNEJTIyUGFnZS0xJTIyJTNFN1ZwZGI1c3dGUDAxUEc0Q2pDRThObW5hYVdxbFNkMjA3dEVMSGxBUmpJalRoUDM2R1RBaFlBSWs0U3ZiWGlKOHNRMCUyQjUlMkZyZTQwc2tzRmp2SDBNVU9NJTJGRXdwNmt5dFplQXZlU3FocUt4bjVqUThRTnFwSWE3TkMxVXRPUjRjWDlqYmxSNXRhdGElMkJGTm9TTWx4S051VURTdWlPJTJGakZTM1lVQmlTWGJIYkwlMkJJVm54b2dHd3VHbHhYeVJPdDMxNkpPYXAycFJtNyUyRmhGM2J5WjZzNkdaNlo0Mnl6bndsR3dkWlpIZGtBa3NKTEVKQ2FIcTEzaSUyQndGMk9YNFpLT2V6aHg5JTJGQmlJZlpwbXdGTDlQQzIwNSUyQlUyZjZ6cjBmNEdVRGo4UU1FNlRUdnlOdnlGVCUyQjVHeXFwT2xvSEVwaDdiT3I1MTIzQTRDaWFvZ0JMY0M1Qk5tTEJ4biUyRnpYUm90MzltcnpORW03MnZIZmNzdEdjZjlOaHdWR21WUWgyVHJXemglMkJXNW4xMnprdXhTOEJXc1YzZDh5M21NMmhhNCUyQjFGSGFKUE5mMjJmV0tUWVZEWm5qSElYVVpiM2Y4QmlWQjhyUmtkZXdlM3AlMkZFVFRtd3did1lreldtWWNTNjhBRUhWJTJCUWVyUEhtTG5jSG1ISHNITG1Dbm8xRDNBWHR3OVE1UyUyQnlDRTNVR2FVQVZTTHVhb211NHFJWmVZS2dMTGtDUkN3QkVNclJaQlJtSzJoY1o2dXclMkZHVHkwajglMkJGTG1DSExSYk9lWk9FMUNFMjhaRzN6SzN6SXJwNW55Y1NJNWRnJTJCb1lwalhodVFsdEtpb2pqdlV0Zmo2NSUyRnhGTjloTHgxdiUyQmN6SjQwb2ElMkZoc3ZhJTJGSGphTlJjVE1mbHJTeWNlbjY0a1hWazhheUpBcHRUR3ZBVXRScWRrUHNJZXElMkJGeDlRUlJVZiUyQm9XNDdORUhyempFd215TDZxVXBObVFicmpBZlZTTDg4QnBYJTJCSUFzN01jeG5LSTlVU2tlZFVUQlhvZ0Nzd2FpVWclMkZxalNoTjRDa1BpcmVpRGNvWUtvb1lBMEdWT0RENkNvRlF6RVpwNnBHVDNDTkx4bHd5N3ElMkZEdDR2Y29aV0FxMGprVmJrRDlJV2Jjc0lkNzFiVUpmN29lQUd6akJjVThESUh4VXVVUFRjZVptSEx2S25WTzdaZUlxbzNzYU5NMm1FMUUwN0xZY1V6MDVUZzB1V0o3VzhJeDlqT2wwanBEa01BQUMxakFEZ1JCRnBMc3V2TyUyRjJMUmhpZDU5WkRrRTlHMlNBZ01YZCUyQmVZczRIRlFmR3Fwd1BlNnVqMUV2UTQlMkZON0djN2VEJTJCeGRhQVpCNEk4dFRvR29UZ3RCZUNERU84QVd3aEsycG9pdG9nMHF5TVNnY0tQWWxyWHUlMkJOakNGbUxYWmpBRkoxZlBQMCUyQmduMWwzJTJCZXpvV2FxMnFiS0l5dERsTmxQTVEzOUw2YlBHRDBTS3hxUEFPQyUyQkRkWWQxbHpXV3M3RE9waWtINExhNVRlc3IlMkZ1cmlaa2c5WHltSXNrR0UyRVdJRmdHRnFnRG9vQVVaVFl5NTlmbHM0QngyRWNTbDRnQTBSMDVzbWhqQmJ4NWtvRTRNWkVVRVdZQnVHcVd5V3ZIVCUyRkoyQ1I4REcwN05SeTElMkY1QXhMTVNvM2R5N29XeEF3dDZ6UmRMUHdNckNteUlQWHZ5RHF6bXFJ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