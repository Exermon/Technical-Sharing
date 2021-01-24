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

用代码来表示：

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
/// 获取委托类型
/// </summary>
/// <param name="types"></param>
/// <returns></returns>
static Type getActionType(Type[] types) {
	switch (types.Length) {
		case 0: return typeof(UnityAction);
		case 1: return typeof(UnityAction<>).MakeGenericType(types);
		case 2: return typeof(UnityAction<,>).MakeGenericType(types);
		case 3: return typeof(UnityAction<,,>).MakeGenericType(types);
		case 4: return typeof(UnityAction<,,,>).MakeGenericType(types);
		default: return null;
	}
}

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

![avatar](https://img-blog.csdnimg.cn/20210124191139914.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L20wXzM3OTcwNTE0,size_16,color_FFFFFF,t_70)

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
