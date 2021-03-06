# SecuritySDK
安全SDK，通过该sdk为Android APP提供一系列安全防护功能，如基本对抗防护、漏洞防护、威胁情报收集等功能。

##配置文件的更新和获取(配置文件格式，参考根目录下的config.json)：
	1）配置文件：securitysdk_config.json(通过preferences方式存储)，是基于json格式的内容，它提供了4类拦截规则：webview的拦截规则、intent uri拦截规则、应用和插件更新的url白名单和应用插件调用的过滤拦截规则
	2）通过SecuritySDKInit.syncConfig(url, params)方法进行配置文件的更新，新版本的更新取决于设置的time_out和version，当更新间隔时间到了，就回去服务器请求新的版本号，如果版本号大于本地版本号，就会进行规则更新；
	3）获得拦截规则：先通过SecuritySDKInit.getConfigStringValueByKey(key)获得本地存储的配置文件json内容，然后自己通过fastjson讲字符串在转换成SecuritySDKConfig中的相关拦截规则对象。示例代码如下：
	String str = SecuritySDKInit.getConfigStringValueByKey(SecuritySDKInit.WEBVIEWCONFIG);
	WebviewConfig webviewConfig = JSON.parseObject(str, WebviewConfig.class);
##SafeZipFile
	提供函数isLegalZipFile()判断zipFile是否是合法的文件。

##SafeWebView
	安全webview是针对webview常见的远程命令执行漏洞，file域攻击、高危接口对外和中间人劫持，这几个高危漏洞设计的安全webview，可以有效的解决上述几个高危漏洞。同时参考了微信、支付宝和github上的开源代码(JSBridge)，做到了有效性、兼容性和安全性并存。
    业务方使用该webview的步骤：
    1）	申明和实例化。
    布局文件中的申明
    <com.example.safeappdemo.webview.SafeWebView
        android:id="@+id/my_webview"
        android:layout_width="match_parent"
        android:layout_height="400dp" >
    </com.example.safeappdemo.webview.SafeWebView>
    2）	在Java中注册一个handler，以便JS调用
    webView.registerHandler("submitFromWeb");
    3）	实现IMethodInvokeInterface接口，其中data是js传递到java层的调用参数；
    public interface IMethodInvokeInterface {
        String  dispatch(String data);
    }
    4）	进行webview的初始化，并加载url。
    webView.init(this, invokeInterface);
    webView.loadUrl("http://www.baidu.com|file:///android_asset/javascript.html");
    5) Js页面，调用上述webview注册的接口：
    WebViewJavascriptBridge.callHandler('submitFromWeb', {'param': str1}, function(responseData) {
    document.getElementById("show").innerHTML = "send get responseData from java, data = " + responseData
    });

    相关注意事项如下：
    1）	当前webview使用file协议时，只允许加载本地私有目录：/android_asset和android_res这两个目录下的html文件。
    2）	安全webview对于可以使用的JS交互接口的url进行了限制，只允许百度域下的url调用，业务方需要根据当前业务场景进行修改。

    3）关于在javascript中注册handler，java中调用的方式，可以参考JSBridge中的使用说明，安全webview也支持。https://github.com/lzyzsd/JsBridge.
##NativeCoreUtil
	提供debugPresent（检测应用是否处于调试中）、runInEmulator（检测应用是否处于模拟器）、rePackage（重打包检查）、
	detectInject（检测应用是否被xposed，substrate等框架注入）、isExisSUAndExecute（是否被root）、getRemoteAppSign（获取远程应用的签名）6个函数
##BinderSecurityUtil
    提供函数checkClientSign方法，在方法内部获取客户端的包名和签名，传给远端校验。需要注意的是该方法只能放在服务中的接口实现中调用，具体使用见Demo中Myservice。
##UpgradeTool
	提供应用安全升级或安全加载功能，具体吐下：
	（1）UpgradeModel
			1、包含databean：应用升级的信息，如描述信息，下载apk的url地址、是否强制安装，应用的版本（1.0.1），开发版本号，要下载文件的MD5值、以及经过服务端私钥签名后的MD5值
			2、code 和 msg 暂时为扩展字段，待后面增加描述
	（2）ISafeInstall安装接口类，业务需要实现的
			getVerCode 为获取当前应用的开发版本，用于对比升级
			install 为业务需要实现的如何安装等功能
			getErrMsg 获取升级应用时产生的错误消息
	（3）UpgradeTool
	        UpgradeTool实例化时需要4个参数:
	            publicKey RSA公钥，用来验证DataBean中的singnedVerifyCode是否和verifyCode一致，防止下载的非法劫持
                savePath  保存路径
                fileName  保存的文件名
                iSafeInstall  业务实现的安装接口
###使用方法
	（1）实现ISafeInstall接口类
	（2）初始化UpgradeTool对象，传入参数
			Context，publicKey,savePath，fileName，接口类的实现类对象
	（3）调用upgrade方法即可。
	业务方下载更新或者相关插件时，只需要调用upgrade方法，流程如下：判断开发版本号是否需要升级 -> 下载apk -> 调用ISafeInstall的具体实现类的Install方法
###相关事项
    （1）securicysdkcore是安全sdk的核心功能实现，app是测试安全sdk的demo apk
     (2)项目成员：daizy(daizhongyin@126.com)、张林(97615274@qq.com)、唐海洋(ffthy@qq.com)。如有任何问题，欢迎随时联系我们。
#AndroidSo 加解密
	核心加解密过程如下：
		1、首先获取.so 在内存中加载的基址
		2、因为android 系统对于.so的加载是基于段的，dynamic段中包含了多个节区，有hash,dynstr(函数的名字在该节区中)，符号表
		3、由于hash表和dynstr表  节区 的类型不唯一，有可能是别的，所以在dynamic段中寻找
		4、hash表中存储了符号表中对应的索引，通过函数名进行哈希运算得到索引，然后去符号表中，符号表的结构是1、dynstr中的索引 2、地址 3、大小
		5、通过符号表中的第一个字节获取dynstr中对应的字符串 ，然后比对，比对成功则进行加密解密
	参考：
		http://blog.csdn.net/jiangwei0910410003/article/details/49966719
		http://bbs.pediy.com/thread-191649.htm
##sdk各文件及其相关API说明
	1、Rc4Util.h Rc4Util.cpp（RC4加密算法的实现文件）
	注：
		因为涉及到修改内存操作，目前暂考虑同位加密，为此选取了RC4加密算法，RC4加密算法原理这里暂不介绍，请自行google。
	2、 getSign.cpp getSign.h(获取签名）
	 通过NativeCoreUtil类中的getRemoteAppSign函数进行调用。传入上下文和包名，返回远程调用的签名。本地签名哈希暂时写在getSign.cpp文件中。
    3、插件调用拦截。使用PluginInvokeValidate 类中的validate函数进行拦截，初始化安全策略文件写在键intercept_plugin_invoke中。
    4、安全通信 调用接口为CryptAndHttps类中的getHttpsUrlConnection函数。使用前 必须先配置服务器证书，证书文件放在Assets目录。具体使用时见MainActivity中的Https登录按钮。
    
        

###使用过程
	（1）业务方将解密的核心代码加入到自己的native代码中，指定SO_NAME（生成的SO名称），FUNC_NAME（加密函数名），RCE_KEY(加密密钥)，RC4_KEY_LEN（加密密钥的长度），生成so
	注：密钥分发和保存
	（2）将so进行加密，指定加密的函数名、加密的密钥（此时加密SO的RC4密钥需要与解密的密钥一致），加密的so文件路径（EncodeSo类中RC4_KEY,funcName,so_path）
	（3）业务方在Build.gradle中指定加载so的方式，如Jnilibs、libs等等，将加密后的so放到指定的路径下即可
	（4）在需要调用jni的地方， static {
        System.loadLibrary("加密后的so库名称");
    }

