[上一页](flow.md)
[回目录](../README.md)
[下一页](../instruction/ISO8601.md)

# 概述
字典数据主要为企业基础数据提供存储或实时查询能力、比如门店、商品、用于流程过程所需要的基础、aiflow为企业提供Api写入aiflow引擎；

## 整体流程
### 1、定义字典
- 在aiflow后台先定义字典基础信息、存储字段、如下面的门店信息、定义了门店编码、门店名称、门店负责人三个字段；

beta:
https://beta.aiflow.fan/admin/dict

线上:
https://aiflow.fan/admin/dict


  ![alt text](../image/dict/definition.png)


### 2、同步字典数据到aiflow
[开发之前准备工作](quickstart.md),如已经开发对接过、跳过此步。

样例工程代码:

https://github.com/AFlowTech/aflow-demo

代码：com.ai.demo.biz.AFlowBiz#saveDictData

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

	public Long saveADictData() {
		ADictData dictData = ADictData.builder()
				.dictCode("shop")
				.fieldValues(Lists.newArrayList(
						ADictFieldValue.of("shopCode", "100000"),
						ADictFieldValue.of("shopName", "望京店"),
						ADictFieldValue.of("showOwner", "c94b1dcd")
				))
				.convertUserCodeFields(Sets.newHashSet(
						AConvertField.builder().fieldCode("showOwner")
								.linkEnterpriseType(ALinkEnterpriseType.FEISHU)
								.build()))
				.build();
		return flowClient.saveDictData(dictData);
	}
}

```


[上一页](quickstart.md)
[回目录](../README.md)
[下一页](flow.md)