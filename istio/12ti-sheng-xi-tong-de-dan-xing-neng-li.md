### 弹性设计

* 应对故障的一种方法，让系统具有容错和适应能力
* 防止故障（Fault）转化为失败（Failure）
* 主要包括：

  * 容错性：重试、幂等

  * 伸缩性：自动水平扩展（autoscaling）

  * 过载保护：超时、熔断、降级、限流

  * 弹性测试：故障注入

![](/image/Istio/istio-Retry.png)

![](/image/Istio/istio-timeout-retry.png)

