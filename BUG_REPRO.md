# Bug Reproduction

## 包的性质

当前 test_model_fix 保存的是被测模型修复后的结果源码，不是初始含 Bug 源码。要复现原始缺陷，必须检出下面固定的 parent SHA；不要在当前修复结果源码上期待重新出现修复前失败。生成系统使用的可信验证补丁和完整验证日志仅在本地留存，不提交到结果分支。

## 问题现象

牧场经理给一台可用的拌料设备提交外部检修派工，供应商确认还没返回时他取消了请求。页面却一直显示工单正在派遣、设备处于维护中，新的派工也被资源冲突挡住；只有供应商自己结束调用后两个状态才恢复，随后重试才正常。请先不要修改代码，查明请求取消为什么没能终止外部确认和释放占用，说明工单、设备与后台确认之间的生命周期和因果链。

## 含 Bug 版本

- 仓库：VanceMichael/go-label-18
- 仓库地址：https://github.com/VanceMichael/go-label-18.git
- parent SHA：35ba8316c0159d090a56e8665b4ef89ad3412af8

## 复现步骤

```bash
git clone -- https://github.com/VanceMichael/go-label-18.git bug-repro
cd bug-repro
git checkout --detach 35ba8316c0159d090a56e8665b4ef89ad3412af8
go test ./internal/maintenance -run ^TestExternalDispatchCancellationRestoresEquipmentAndAllowsRetry$ -count=1
```

## 双架构完整错误信息

### linux/amd64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/maintenance -run ^TestExternalDispatchCancellationRestoresEquipmentAndAllowsRetry$ -count=1
--- FAIL: TestExternalDispatchCancellationRestoresEquipmentAndAllowsRetry (0.21s)
    model_test.go:137: cancelled dispatch did not stop vendor confirmation
FAIL
FAIL	go-base/internal/maintenance	0.240s
FAIL

```

stderr：

```text
(empty)
```

### linux/arm64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/maintenance -run ^TestExternalDispatchCancellationRestoresEquipmentAndAllowsRetry$ -count=1
--- FAIL: TestExternalDispatchCancellationRestoresEquipmentAndAllowsRetry (0.20s)
    model_test.go:137: cancelled dispatch did not stop vendor confirmation
FAIL
FAIL	go-base/internal/maintenance	0.203s
FAIL

```

stderr：

```text
(empty)
```

## 通过条件

诊断结论应准确定位 internal/maintenance/model.go 的 DispatchExternal 以及它传给 VendorDispatcher.Confirm 的上下文替换，解释为什么调用方取消后供应商确认不返回，进而阻断工单与设备的补偿恢复和结果通道交付；还需说明供应商自行结束后占用为何解除、正常确认与后续重试的边界。以 TestExternalDispatchCancellationRestoresEquipmentAndAllowsRetry 的红测作为运行证据，目标仓库的生产代码、测试和配置必须保持零改动，不得实施修复或只写成笼统的 goroutine 阻塞。
