[上一页](quickstart.md)
[回目录](../README.md)
[下一页](dict_data.md)

# 概述
aiflowSSO登录是基于 OAuth 2.0 标准协议实现的，通过双向安全认证、免密登录aiflow。

## 整体流程
### Step1 申请AppID 和 AppSecret
- 在对接aiflow引擎之前，首先你需要向aiflow科技申请 enterpriseCode,AppID 和 AppSecret，联系公司对接人获取。

### Step2 绑定企业平台
- 如果贵公司与飞书、钉钉、企业微信等有集成、且需要aiflow与这些平台打通、同步用户/部门等基础信息、则需要
在对应平台申请AppId，AppSecret并发给aiflow技术进行配置；参考 [飞书对接](飞书对接.md)

### Step3 开发与测试
 先了解如下与aiflow系统交互流程、整体需要贵公司前端与后端依赖aiflow提供的client加载aiflow页面、并完成SSO登录；

#### Iframe-SSO登录交互流程
- 1、用户访问aiflow页面时、贵公司的前端应用通过aiflow的前端组件(qfcore.js)集成加载aiflow页面 、见[前端集成](#前端集成)；
- 2、调用qfcore.js后，qfcore.js会发起登录aiflow请求、该请求接口需要贵公司的后端参考[accessToken标准接口](#accessToken标准接口)实现该标准接口、获取到正确的accessToken后、qfcore.js就会完成继续调用aiflow登录接口、完成
SSO登录，全程用户无感知;
##### 所以集成整体开发工作量就两点：
- 1、贵公司的前端调用qfcore.js加载页面;
- 2、贵公司的后端实现 [accessToken标准接口](#accessToken标准接口);

```mermaid
sequenceDiagram
    actor user as 用户
    participant aflowPage as aiflow应用页面
    participant customServer as 企业后端应用
    participant aflowServer as aiflow服务

    user ->> aflowPage: 进入aiflow页面
    aflowPage ->> customServer: 获取 accessToken
    note left of aflowPage: 接口地址配置在前端SDK中<br/>请求中会携带当前页面域下的 cookie 信息
    customServer ->> aflowServer: 获取 accessToken
    note left of customServer: 携带用户信息，参考SDK
    aflowServer ->> customServer: 返回 accessToken
    customServer ->> aflowPage: 返回 accessToken
    aflowPage ->> aflowServer: 通过 accessToken 登录
    aflowServer ->> aflowPage: 返回登录结果，添加 qtoken 到aiflow对应域下
```

#### 飞书/钉钉-SSO登录交互流程
- 1、用户进入aiflow页面后、请求aiflow后端；
- 2、aiflow通过配置的飞书/钉钉/企微的 工作台应用的  App ID 和 App Secret 去调用对应平台user_access_token
- 3、校验合法性后，完成aiflow的sso登陆

```mermaid
sequenceDiagram
    actor user as 用户
    participant aflowPage as aiflow应用页面
    participant aflowServer as aiflow服务(统一SSO登陆标准接口)
    participant customServer as 飞书/钉钉/企业微信

    user ->> aflowPage: 进入aiflow页面
    aflowPage ->> aflowServer: 获取 accessToken
    note left of aflowPage: 接口地址配置在前端SDK中<br/>请求中会携带当前页面域下的 cookie 信息
    aflowServer ->> customServer: 获取 accessToken
    note left of aflowServer: 携带用户信息，参考SDK
    customServer ->> aflowServer: 校验合法性后 / 返回 accessToken
    aflowServer ->> aflowPage: 返回 accessToken
    aflowPage ->> aflowServer: 通过 accessToken 登录
    aflowServer ->> aflowPage: 返回登录结果，添加 qtoken 到aiflow对应域下
```

## 前端集成
PC浏览器端采用iframe嵌入的方式加载aiflow工作台界面，步骤如下：

1、下载[qfcore.js](../source/qfcore.js)文件，放置于工程目录中。qfcore是aiflow前端基础api库，主要用于自动化登录、获取待办等统计信息。

2、代码中引用qfcore.js
```javascript
// commonjs
const qfcore = require('./qfcore');

// es6
import qfcore from './qfcore';
```

3、qfcore api包括
```javascript
// qfcore版本号，返回版本号string
qfcore.getVersion() 

// qfcore初始化，返回promise对象，then参数为是否已登录boolean
qfcore.init({
  // accessToken通过后端接口获取
  accessToken: '',
  // 终端，比如pc端(pc)、飞书小程序(lark-mini)
  client: 'pc',
  // 运行环境，可以不填，默认为prod，也可填beta，beta主要用于测试
  env: 'beta'
})

// 是否已登录，返回登录boolean
qfcore.isLogged()

// 登录，返回promise对象，then参数为是否已登录boolean
qfcore.login()

// 登出，返回promise对象，then参数为是否登出成功boolean
qfcore.logout()

// 请求任务数量统计，返回promise对象，then参数为统计summary object
qfcore.querySummary()
summary: {
  // 待办
  "allTodo": 0,
  // 已超时
  "overdue": 0,
  // 即将超市
  "soonOverdue": 0,
  // 催办
  "urge": 0
}

// 监听任务数量统计，参数为function，用于回调统计数据
qfcore.listenSummary((data) => {
  console.log(data);
});
```

4、PC web端，在页面中调用qfcore.init，初始化成功后展示iframe，示例：
```javascript
// react
import React, { useState, useEffect } from 'react';
import request from 'request';
import qfcore from './qfcore';

export default function Page() {
  const [logged, setLogged] = useState(false);

  useEffect(() => {
    // ACCESS_TOKEN_URL从后端获取
    request.post(ACCESS_TOKEN_URL).then((res) => {
      const accessToken = res.data.data;
      qfcore
        .init({
          accessToken: accessToken,
          client: 'pc',
          // 线上prod，开发测试用beta
          env: 'prod'
        })
        .then((success) => {
          if (success) {
            setLogged(true);
  
            qfcore.querySummary().then((data) => {
              // summary数据可用于入口处气泡等
              console.log(data);
            });
          }
        });
    });
  }, []);
  // 线上src用http://www.kuaiflow.com/user/embed，beta或dev用http://gray.kuaiflow.com/user/embed
  return logged && <iframe style={{ width: '100%', height: '100vh' }} src="http://www.kuaiflow.com/user/embed"></iframe>;
}
```
iframe的src线上生产环境为https://www.kuaiflow.com/user/embed，目前内测阶段，先填写http://beta.kuaiflow.com/user/embed

5、小程序端，先执行1-4步骤（文件下载qfcore-miniapp），再下载qflow-mini.zip组件包，根据自己工程规范将其放置在根目录、/pages或者其他任意目录下，然后做如下配置：
```
//app.json文件中增加pages，下面路径中前面“pages/qflow”取决于放置qflow组件包的路径，比如放置在/pages下，则取pages/qflow，放置在根目录下，则取flow
[
  "pages/qflow/pages/index/index",
  "pages/qflow/pages/detail/index",
  "pages/qflow/pages/handle/index",
  "pages/qflow/pages/list/index",
  "pages/qflow/pages/new/index",
  "pages/qflow/pages/subflow/index"
]

//qfcore部分参照4增加如下代码
+ const taroApp = require('./qflow/app.js').taroApp;

request.post(ACCESS_TOKEN_URL).then((res) => {
  const accessToken = res.data.data;
  qfcore
    .init({
      accessToken: accessToken,
      client: 'pc',
      // 线上prod，开发测试用beta
      env: 'prod'
    })
    .then((token) => {
      if (token) {
        setLogged(true);
        
+       taroApp.onShow({
+         token: token,
+         pathPrefix: '/pages/qflow' // pathPrefix取决于qflow文件夹位置
+       });

        qfcore.querySummary().then((data) => {
          // summary数据可用于入口处气泡等
          console.log(data);
        });
      }
    });
});

//跳转到aiflow首页
+ goToIndex() {
+   wx.navigateTo({
+     url: '/pages/qflow/pages/index/index'
+   });
+ }

```


## accessToken标准接口
样例工程代码:
https://github.com/QFlowTech/kuaiflow-demo

后端工作量：
主要需要参考样例工程中的com.kuaiflow.demo.controller.KuaiFlowController#getAccessToken,
实现该http接口。

### Java-SDK
尽量使用线上最新版本:https://repo1.maven.org/maven2/com/kuaiflow/kuaiflow-client/
```xml
<dependency>
    <groupId>com.aflow</groupId>
    <artifactId>aflow-client</artifactId>
    <version>0.0.23</version>
</dependency>
```

### 获取access_token
```java
@Service
public class AFlowBiz {

	private FlowClient flowClient;

	@Value("${a.flow.enterpriseCode}")
	private String enterpriseCode;

	@Value("${a.flow.appId}")
	private String appId;

	@Value("${a.flow.appSecret}")
	private String appSecret;

	//环境[beta:测试/prod:线上]
	@Value("${a.flow.environmentType}")
	private String environmentType;

	@PostConstruct
	public void init() {
		// 实例化一个FlowClient、为了保护密钥安全，建议将密钥设置在环境变量中或者配置文件中。
		// Credential cred = new Credential("enterpriseCode","beta", "appId","appSecret");
		Credential credential = new Credential(enterpriseCode, environmentType, appId, appSecret);
		flowClient = new FlowClient(credential);
	}

	public String getAccessToken() {
		CustomAuthentication customAuthentication = new CustomAuthentication();
		// 企业的用户编码-这里获取当前登陆用户的企业userCode、即贵公司的用户唯一Id
		// customAuthentication.setCustomUserCode(UserContext.getUserCode());
		customAuthentication.setCustomUserCode("9910031941");
		// 贵公司使用的三方平台用户编码、如飞书Id
		// customAuthentication.setLinkUserCode(UserContext.getLinkUserCode());
		customAuthentication.setLinkUserCode("c94b1dcd");
		return flowClient.getAccessToken(customAuthentication);
	}
}

```


[上一页](quickstart.md)
[回目录](../README.md)
[下一页](flow.md)