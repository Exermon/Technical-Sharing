# 网络交互
## 基本模型

客户端发送一个字符串（JSON格式），服务器处理后返回一个字符串（JSON格式）

## 序列化和反序列化

序列化：一个对象→JSON数据
反序列化：JSON数据→一个对象

序列化和反序列化的工具有很多，但是实践发现这些工具不太好用

这里推荐使用Exermon开发的数据模型（BaseData+LitJson）

关于这个数据模型如果你们有兴趣下次再展开讲，这里只是简单过一下

后面涉及到这部分内容的，默认使用该数据模型

## 函数封装

根据基本模型，可以写出一个发起网络请求函数的签名：
（方便起见，这里使用静态类，实际上是使用单例模式）

``` C#

public static class NetworkSystem {
	
	/// <summary>
	/// 发起请求
	/// </summary>
	/// <param name="route">路由</param>
	/// <param name="data">要发送到后台的JSON字符串</param>
	/// <param name="onSuccess">成功时候的回调函数，参数为后台返回的JSON字符串</param>
	/// <param name="onError">异常时候的回调函数，参数为异常描述字符串（根据实际情况确定）</param>
	public static void request(string route, string data, UnityAction<string> onSuccess, UnityAction<string> onError);

}

```

注：本文不会讲解`request`函数的实现。因为根据你用的库不同，具体的实现方式也不同。知道调用`request`函数可以发起请求即可。

一般来说，不可能直接使用这个函数去发起请求，因为传入的`data`是一个JSON字符串，手撸是不可能手撸的，所以要封装：

首先是要有一个对象保存这个JSON数据，这里推荐使用`LitJson`插件的`JsonData`类；
其次是需要将这个函数封装，把传入和传出的数据类型都改为`JsonData`

那么封装之后的函数如下（这里就省略函数注释了）：

``` C#
public static class NetworkSystem {
	
	/// <summary>
	/// 发起请求
	/// </summary>
	/// <param name="route">路由</param>
	/// <param name="data">要发送到后台的JSON字符串</param>
	/// <param name="onSuccess">成功时候的回调函数，参数为后台返回的JSON字符串</param>
	/// <param name="onError">异常时候的回调函数，参数为异常描述字符串（根据实际情况确定）</param>
	public static void request(string route, string data, UnityAction<string> onSuccess, UnityAction<string> onError);
	public static void request(string route, JsonData data, UnityAction<JsonData> onSuccess, UnityAction<string> onError) {
		var _data = data.ToJson(); // 这个函数可以将一个JsonData转化为一个JSON字符串
		var _onSuccess = text => {
			var json = JsonMapper.ToObject(text); // 这个函数可以将一个JSON字符串转化为JsonData
			onSuccess?.Invoke(json);
		};
		request(route, _data, _onSuccess, onError);
	}
}
```

但是，一般来说，程序里面需要先对返回的数据进行处理（根据接口约定，可能需要进行一些异常的判断），或者是进行异常处理，再调用回调函数。所以建议将里面的函数提取出来，这样更直观：

``` C#
public static class NetworkSystem {
	
	/// <summary>
	/// 发起请求
	/// </summary>
	/// <param name="route">路由</param>
	/// <param name="data">要发送到后台的JSON字符串</param>
	/// <param name="onSuccess">成功时候的回调函数，参数为后台返回的JSON字符串</param>
	/// <param name="onError">异常时候的回调函数，参数为异常描述字符串（根据实际情况确定）</param>
	public static void request(string route, string data, UnityAction<string> onSuccess, UnityAction<string> onError);
	public static void request(string route, JsonData data, UnityAction<JsonData> onSuccess, UnityAction<JsonData> onError) {
		var _data = data.ToJson(); // 这个函数可以将一个JsonData转化为一个JSON字符串
		var _onSuccess = processRequestSuccess(onSuccess);
		var _onError = processRequestError(onError);

		// 这里看上去代码会很长，之后会进一步优化
		var _onSuccess = text => onRequestSuccess(text, onSuccess);
		var _onError = text => onRequestError(text, onError);

		onRequestStart(); // 请求发起回调

		request(route, _data, _onSuccess, _onError);
	}

	/// <summary>
	/// 请求成功回调
	/// </summary>
	/// <param name="text">返回的JSON字符串</param>
	/// <param name="action">回调函数</param>
	static void onRequestSuccess(string text, UnityAction<JsonData> action) {
		var json = JsonMapper.ToObject(text);
		action?.Invoke(json);
		onRequestEnd();
	}

	/// <summary>
	/// 请求异常回调
	/// </summary>
	/// <param name="text">异常描述字符串</param>
	/// <param name="action">回调函数</param>
	static void onRequestError(string text, UnityAction<string> action) {
		processError(text); // 进行异常处理，比如弹出弹窗
		action?.Invoke(text);
		onRequestEnd();
	}

	/// <summary>
	/// 请求结束回调
	/// </summary>
	static void onRequestEnd() { 
		// 自定义实现
	}
}

```

由于请求的过程是异步的，在发起请求之后到响应回来之前这段时间往往需要显示一个Loading的遮罩，这个遮罩上还会显示一段提示文本，同时还会有很多其他配置数据。

我们可以定义一个请求对象（`RequestObject`）类来封装这些数据。

这里直接给出我们的`NetworkSystem`的部分代码（有修改）：

``` C#

/// <summary>
/// 回调函数类型
/// </summary>
public delegate void SuccessAction(JsonData data);
public delegate void ErrorAction(string data);

/// <summary>
/// 请求对象
/// </summary>
class RequestObject {

    /// <summary>
    /// 请求参数
    /// </summary>
    public string route; // 请求路由
    public JsonData data; // 发送的数据

    public SuccessAction onSuccess; // 成功回调
    public ErrorAction onError; // 失败回调
    public UnityAction onEnd; // 结束回调

    public bool showLoading; // 是否显示Loading遮罩
    public string tipsText; // Loading遮罩提示文本

    /// <summary>
    /// 构造函数
    /// </summary>
    public RequestObject(string route, JsonData data, 
    	SuccessAction onSuccess = null, ErrorAction onError = null, UnityAction onEnd = null,
        bool showLoading = true, string tipsText = "") {
        this.route = route; this.data = data; 
        this.onSuccess = onSuccess; this.onError = onError; this.onEnd = onEnd;
        this.showLoading = showLoading; this.tipsText = tipsText;
    }
}

```

在上面的代码中，请求对象不仅封装了一些回调函数，还封装了关于这次请求的一些设置数据，甚至还封装了请求的原始数据（方便请求失败时候进行重传）。这样不仅能够简化上面的代码，还有很好的功能性和拓展性。

有了这个`RequestObject`，封装后的代码大致为：

``` C#

public static class NetworkSystem {

	/// <summary>
	/// 发起请求
	/// </summary>
	/// <param name="route">路由</param>
	/// <param name="data">要发送到后台的JSON字符串</param>
	/// <param name="onSuccess">成功时候的回调函数，参数为后台返回的JSON字符串</param>
	/// <param name="onError">异常时候的回调函数，参数为异常数据的JSON字符串（根据实际情况确定）</param>
	public static void request(string route, string data, UnityAction<string> onSuccess, UnityAction<string> onError);
	public static void request(string route, JsonData data, 
		SuccessAction onSuccess, ErrorAction onError, UnityAction onEnd = null,
		bool showLoading = true, string tipsText = "") {

		var requestObject = new RequestObject(route, data, 
			onSuccess, onError, onEnd, showLoading, tipsText);

		request(requestObject);
	}
	public static void request(RequestObject requestObject) {
		var route = requestObject.route;
		var _data = requestObject.data.ToJson();

		var _onSuccess = text => onRequestSuccess(text, requestObject);
		var _onError = text => onRequestError(text, requestObject);

		onRequestStart(requestObject); // 请求发起回调

		request(route, _data, _onSuccess, _onError);
	}

	/// <summary>
	/// 请求开始回调
	/// </summary>
	/// <param name="requestObject">请求对象</param>
	static void onRequestStart(RequestObject requestObject) {
		if (requestObject.showLoading)
			doShowLoading(requestObject.tipsText); // 执行遮罩显示
	}

	/// <summary>
	/// 请求成功回调
	/// </summary>
	/// <param name="text">返回的JSON字符串</param>
	/// <param name="requestObject">请求对象</param>
	static void onRequestSuccess(string text, RequestObject requestObject) {
		var json = JsonMapper.ToObject(text);
		requestObject.onSuccess?.Invoke(json);
		onRequestEnd(requestObject);
	}

	/// <summary>
	/// 请求异常回调
	/// </summary>
	/// <param name="text">异常描述字符串</param>
	/// <param name="requestObject">请求对象</param>
	static void onRequestError(string text, RequestObject requestObject) {
		processError(text); // 进行异常处理，比如弹出弹窗
		requestObject.onError?.Invoke(text);
		onRequestEnd(requestObject);
	}

	/// <summary>
	/// 请求结束回调
	/// </summary>
	/// <param name="requestObject">请求对象</param>
	static void onRequestEnd(RequestObject requestObject) { 
		if (requestObject.showLoading) doHideLoading(); // 执行遮罩隐藏
		requestObject.onEnd?.Invoke();
	}
}

```

现在的代码看似很完美了，可拓展性也很强，但仍有两点可以优化：

1. 以下语句因为使用了箭头函数，还是显得不够直观，可以优化：
``` C#
var _onSuccess = text => onRequestSuccess(text, requestObject);
var _onError = text => onRequestError(text, requestObject);
```

2. `request`，`onRequestStart`，`onRequestSuccess`，`onRequestError`，`onRequestEnd`这几个函数存在一个`RequestObejct`类型的参数，函数的内容都是对`RequestObejct`内的数据进行操作，这样是不是很像上个年代的面向过程的编程思想呢？

针对这两点，我们只需要将上面几个函数移动到`RequestObejct`内部即可。新的代码如下：

``` C#

/// <summary>
/// 回调函数类型
/// </summary>
public delegate void SuccessAction(JsonData data);
public delegate void ErrorAction(JsonData data);

/// <summary>
/// 请求对象
/// </summary>
public class RequestObject {

    /// <summary>
    /// 请求参数
    /// </summary>
    public string route; // 请求路由
    public JsonData data; // 发送的数据

    public SuccessAction onSuccess; // 成功回调
    public ErrorAction onError; // 失败回调
    public UnityAction onEnd; // 结束回调

    public bool showLoading; // 是否显示Loading遮罩
    public string tipsText; // Loading遮罩提示文本

    /// <summary>
    /// 构造函数
    /// </summary>
    public RequestObject(string route, JsonData data, 
    	SuccessAction onSuccess = null, ErrorAction onError = null, UnityAction onEnd = null,
        bool showLoading = true, string tipsText = "") {
        this.route = route; this.data = data; 
        this.onSuccess = onSuccess; this.onError = onError; this.onEnd = onEnd;
        this.showLoading = showLoading; this.tipsText = tipsText;
    }

    /// <summary>
    /// 发起请求
    /// </summary>
	public void request() {
		onRequestStart(); 
		NetworkSystem.request(route, data.ToJson(), 
			onRequestSuccess, onRequestError);
	}

	/// <summary>
	/// 请求开始回调
	/// </summary>
	/// <param name="requestObject">请求对象</param>
	void onRequestStart() {
		if (showLoading) doShowLoading(tipsText); // 执行遮罩显示
	}

	/// <summary>
	/// 请求成功回调
	/// </summary>
	/// <param name="text">返回的JSON字符串</param>
	void onRequestSuccess(string text) {
		var json = JsonMapper.ToObject(text);
		onSuccess?.Invoke(json);
		onRequestEnd();
	}

	/// <summary>
	/// 请求异常回调
	/// </summary>
	/// <param name="text">异常描述字符串</param>
	void onRequestError(string text) {
		processError(text); // 进行异常处理，比如弹出弹窗
		onError?.Invoke(text);
		onRequestEnd();
	}

	/// <summary>
	/// 请求结束回调
	/// </summary>
	/// <param name="requestObject">请求对象</param>
	void onRequestEnd() { 
		if (showLoading) doHideLoading(); // 执行遮罩隐藏
		onEnd?.Invoke();
	}
}

public static class NetworkSystem {
	
	/// <summary>
	/// 发起请求
	/// </summary>
	/// <param name="route">路由</param>
	/// <param name="data">要发送到后台的JSON字符串</param>
	/// <param name="onSuccess">成功时候的回调函数，参数为后台返回的JSON字符串</param>
	/// <param name="onError">异常时候的回调函数，参数为异常数据的JSON字符串（根据实际情况确定）</param>
	public static void request(string route, string data, UnityAction<string> onSuccess, UnityAction<string> onError);
	public static void request(string route, JsonData data, 
		SuccessAction onSuccess, ErrorAction onError, UnityAction onEnd = null,
		bool showLoading = true, string tipsText = "") {

		var requestObject = new RequestObject(route, data, 
			onSuccess, onError, onEnd, showLoading, tipsText);

		requestObject.request();
	}
}

```

现在的代码基本上很完美了，但是还有一个访问性的问题。

在这里，我们不希望用户直接调用原始的`request`函数（不然我们也不用这样封装），因此必须修改他的访问性为`private`或`protected`。修改之后为了能够让`RequestObject`可以访问，我们需要将`RequestObject`移到`NetworkSystem`类内部（这也更符合我们封装的逻辑）。

此外，我们可能也不希望用户直接创建一个`RequestObject`来发起请求，我们也需要在上面的基础上将`RquestObject`类访问性改为私有。

最终代码如下：

``` C#
public static class NetworkSystem {

	/// <summary>
	/// 回调函数类型
	/// </summary>
	public delegate void SuccessAction(JsonData data);
	public delegate void ErrorAction(JsonData data);

	/// <summary>
	/// 请求对象
	/// </summary>
	class RequestObject { // 访问性改为私有

		/// <summary>
		/// 请求参数
		/// </summary>
		public string route; // 请求路由
		public JsonData data; // 发送的数据

		public SuccessAction onSuccess; // 成功回调
		public ErrorAction onError; // 失败回调
		public UnityAction onEnd; // 结束回调

		public bool showLoading; // 是否显示Loading遮罩
		public string tipsText; // Loading遮罩提示文本

		/// <summary>
		/// 构造函数
		/// </summary>
		public RequestObject(string route, JsonData data,
			SuccessAction onSuccess = null, ErrorAction onError = null, UnityAction onEnd = null,
			bool showLoading = true, string tipsText = "") {
			this.route = route; this.data = data;
			this.onSuccess = onSuccess; this.onError = onError; this.onEnd = onEnd;
			this.showLoading = showLoading; this.tipsText = tipsText;
		}

		/// <summary>
		/// 发起请求
		/// </summary>
		public void request() {
			onRequestStart();
			NetworkSystem.request(route, data.ToJson(),
				onRequestSuccess, onRequestError);
		}

		/// <summary>
		/// 请求开始回调
		/// </summary>
		/// <param name="requestObject">请求对象</param>
		void onRequestStart() {
			if (showLoading) doShowLoading(tipsText); // 执行遮罩显示
		}

		/// <summary>
		/// 请求成功回调
		/// </summary>
		/// <param name="text">返回的JSON字符串</param>
		void onRequestSuccess(string text) {
			var json = JsonMapper.ToObject(text);
			onSuccess?.Invoke(json);
			onRequestEnd();
		}

		/// <summary>
		/// 请求异常回调
		/// </summary>
		/// <param name="text">异常描述字符串</param>
		void onRequestError(string text) {
			processError(text); // 进行异常处理，比如弹出弹窗
			onError?.Invoke(text);
			onRequestEnd();
		}

		/// <summary>
		/// 请求结束回调
		/// </summary>
		/// <param name="requestObject">请求对象</param>
		void onRequestEnd() {
			if (showLoading) doHideLoading(); // 执行遮罩隐藏
			onEnd?.Invoke();
		}
	}

	/// <summary>
	/// 发起请求
	/// </summary>
	/// <param name="route">路由</param>
	/// <param name="data">要发送到后台的JSON字符串</param>
	/// <param name="onSuccess">成功时候的回调函数，参数为后台返回的JSON字符串</param>
	/// <param name="onError">异常时候的回调函数，参数为异常数据的JSON字符串（根据实际情况确定）</param>

	// 访问性改为私有
	static void request(string route, string data, UnityAction<string> onSuccess, UnityAction<string> onError);

	public static void request(string route, JsonData data,
		SuccessAction onSuccess, ErrorAction onError, UnityAction onEnd = null,
		bool showLoading = true, string tipsText = "") {

		var requestObject = new RequestObject(route, data,
			onSuccess, onError, onEnd, showLoading, tipsText);

		requestObject.request();
	}
}

```

# 实例分析
## 源码阅读

``` C#
public class HTTPService : MonoBehaviour {

	/// <summary>
	/// HTTP POST
	/// </summary>
	/// <param name="url"></param>
	/// <param name="_postData">json数据</param>
	public void Post(string url, PostData data,
	    UnityAction<string> onSuccess = null, UnityAction<string> onFail = null) {
	    StartCoroutine(PostRequest(url, data, onSuccess, onFail));
	}

	/// <summary>
	/// POST携程
	/// </summary>
	/// <param name="url"></param>
	/// <param name="_postData"></param>
	/// <returns></returns>
	private IEnumerator PostRequest(string url, PostData data,
	    UnityAction<string> onSuccess = null, UnityAction<string> onFail = null) {
	    var js = JsonUtility.ToJson(data);
	    Debug.Log(js);

	    UnityWebRequest webRequest = new UnityWebRequest(url, UnityWebRequest.kHttpVerbPOST);
	        //UnityWebRequest.Post(url, js);
	    UploadHandler uploader = new UploadHandlerRaw(
	        System.Text.Encoding.Default.GetBytes(js));
	    webRequest.uploadHandler = uploader;
	    webRequest.uploadHandler.contentType = "application/json";  //设置HTTP协议的请求头，默认的请求头HTTP服务器无法识别
	    webRequest.certificateHandler = new WebRequestCert();

	    //这里需要创建新的对象用于存储请求并响应后返回的消息体，否则报空引用的错误
	    DownloadHandler downloadHandler = new DownloadHandlerBuffer();
	    webRequest.downloadHandler = downloadHandler;

	    // Request and wait for the desired page.
	    yield return webRequest.SendWebRequest();

	    if (webRequest.isNetworkError || webRequest.isHttpError) {
	        onFail?.Invoke(webRequest.error);
	    }
	    else {
	        onSuccess.Invoke(webRequest.downloadHandler.text);
	    }
	    Debug.Log("respone code:" + webRequest.responseCode);
	}

	/// <summary>
	/// 原料数据显示
	/// </summary>
	public void loadShowMaterial(int dbID = 0, int labID = 0, 
	    UnityAction<int, MaterialDatabase> onSuccess = null,
	    UnityAction<int> onFail = null) {
	    var postData = new ShowMaterialPostData();
	    postData.material_database_id = dbID;
	    postData.lab_machine_id = labID;

	    Post(webURL + showMaterialRoote, postData, (text) => {
	        Debug.Log("material");
	        Debug.Log(text);

	        var js = JsonMapper.ToObject(text);
	        var newJs = new JsonData();
	        newJs.SetJsonType(JsonType.Array);
	        foreach (var val in js.Keys)
	            newJs.Add(js[val]);

	        var ms = DataLoader.load<MaterialDatabase.LabMaterial[]>(newJs);
	        var m = new MaterialDatabase();
	        m.labMaterials = ms;
	        onSuccess?.Invoke(labID, m);
	        Debug.Log(m.toJson().ToJson());
	    }, text => { Debug.Log(text); onFail?.Invoke(labID); });
	}

	/// <summary>
	/// 载入实验室参数
	/// </summary>
	public void loadLabVar(int dbID = 0, UnityAction<Data.Lab> onSuccess = null,
	    UnityAction onFail = null) {
	    var postData = new LabVarPostData();
	    postData.lab_var_id = dbID;
	    postData.chapter_id = dataContainer.chapterID;

	    Post(webURL + labVarRoote, postData, (text) => {
	        Debug.Log("labVar");
	        Debug.Log(text);
	        var m = JsonUtility.FromJson<Lab>(text);
	        onSuccess?.Invoke(m);
	        Debug.Log(JsonUtility.ToJson(m, true));
	    }, text => { Debug.Log(text); onFail?.Invoke(); });
	}

	/// <summary>
	/// 原料数据提交
	/// </summary>
	/// <param name="lab_machine_id">实验仪ID</param>
	/// <param name="lab_row_id">容器仓ID </param>
	/// <param name="lab_bottle_id">容器位ID</param>
	/// <param name="material_id">原料ID</param>
	public void confirmMaterial(int lab_machine_id, int lab_row_id, int lab_bottle_id, 
	    int material_id, UnityAction<OperationScoreMessage> onSuccess = null, UnityAction onFail = null) {
	    var postData = new EvaluateShowPostData();
	    postData.material_database_id = dataContainer.materialDatabaseID;
	    postData.lab_machine_id = lab_machine_id;
	    postData.lab_row_id = lab_row_id;
	    postData.lab_bottle_id = lab_bottle_id;
	    postData.material_id = material_id;
	    postData.chapter_id = dataContainer.chapterID;

	    Post(webURL + scoreShowRoote, postData, (text) => {
	        Debug.Log(text);
	        var m = JsonUtility.FromJson<OperationScoreMessage>(text);
	        onSuccess?.Invoke(m);
	    }, text => { Debug.Log(text); onSuccess?.Invoke(new OperationScoreMessage()); });
	}

}
```

## 重构策略

1. 目前代码里使用了两套序列化反序列化工具（`JsonUtility`和`LitJson`），调用混乱，不利于维护。采用Exermon数据模型进行重构；
2. `PostData`建议直接用`LitJson`的`JsonData`代替。不然当接口数量的增加，要声明的`PostData`的类就更多，会导致大量的重复性工作，使用Exermon数据模型进行重构；
3. 代码中调用`Post`函数的第一个参数均为`webURL + route`的形式。对于这种只涉及单个地址的通讯，直接把`webURL`封装到`Post`函数内即可；
4. 成功回调函数中，有太多重复的逻辑（将`text`转化为一个对象），可以结合第一点封装到预处理的过程中；
5. 大量使用箭头函数，导致代码不够直观。建议提取回调函数；
6. `Get/Post`函数和具体接口调用的函数都写在同一个类中，耦合度过大，建议分离；
7. 在6的基础上，分离后`Get/Post`所在的类建议使用单例模式（考虑到需要使用协程，需要继承`MonoBehaviour`，我们可以用`DontDestroyOnLoad`，并把当前实例存放到一个静态类/单例类中。总之就是要保证只有一个实例，并且有一个公有的入口可以访问）；
8. 添加一个Loading遮罩功能。