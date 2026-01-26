[上一页](sso.md)
[回目录](../README.md)
[下一页](faq.md)

## 后端
### 查询流程
```java
public Long start() {
	OpenFlowApi openFlowApi = new OpenFlowApiImpl();
	StartFlowParam param = new StartFlowParam();
	//set StartFlowParam
	return openFlowApi.start(param);
}
```


[上一页](sso.md)
[回目录](../README.md)
[下一页](faq.md)