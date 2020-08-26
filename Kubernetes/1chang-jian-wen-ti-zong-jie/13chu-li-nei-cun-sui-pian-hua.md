# 处理内存碎片化 {#处理内存碎片化}

## 内存碎片化造成的危害 {#内存碎片化造成的危害}

节点的内存碎片化严重，导致docker运行容器时，无法分到大的内存块，导致start docker失败。最终导致服务更新时，状态一直都是启动中

## 判断是否内存碎片化严重 {#判断是否内存碎片化严重}

内核日志显示：![](/image/kubernetes/内存碎片化.png)![](/image/kubernetes/内存碎片化2.png)进一步查看的系统内存\(cache多可能是io导致的，为了提高io效率留下的缓存，这部分内存实际是可以释放的\)：![](/image/kubernetes/内存碎片化3.png)

查看slab \(后面的0多表示伙伴系统没有大块内存了\)：

## 解决方法 {#解决方法}

* 周期性地或者在发现大块内存不足时，先进行drop\_cache操作:

  ```
  echo 3 > /proc/sys/vm/drop_caches
  ```

* 必要时候进行内存整理，开销会比较大，会造成业务卡住一段时间\(慎用\):

  ```
  echo 1 > /proc/sys/vm/compact_memory
  ```

## 附录 {#附录}

相关链接：

* [https://www.lijiaocn.com/%E9%97%AE%E9%A2%98/2017/11/13/problem-unable-create-nf-conn.html](https://www.lijiaocn.com/问题/2017/11/13/problem-unable-create-nf-conn.html)
* [https://blog.csdn.net/wqhlmark64/article/details/79143975](https://blog.csdn.net/wqhlmark64/article/details/79143975)
* [https://huataihuang.gitbooks.io/cloud-atlas/content/os/linux/kernel/memory/drop\_caches\_and\_compact\_memory.html](https://huataihuang.gitbooks.io/cloud-atlas/content/os/linux/kernel/memory/drop_caches_and_compact_memory.html)



