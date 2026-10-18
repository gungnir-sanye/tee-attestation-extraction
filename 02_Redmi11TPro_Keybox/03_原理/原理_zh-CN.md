# 第二章 Redmi 11T Pro：MediaTek TEE 的 TA 容器和调用者门

## 2.1 研究目标

Redmi 11T Pro 的目标更加集中：证明 MediaTek TEE 中已有的 Keybox 只读处理路径，在特定版本和授权研究环境下是否可以被普通世界调用，并将返回载荷解析为 EC/RSA 私钥和证书链。

这条路线和 Nothing 2a 的区别是：

- Nothing 2a 主要是 Trustonic TCI/Trustlet 路径；
- Redmi 11T Pro 主要是 MediaTek TEE/GenieZone/uTsecManager 路径；
- Keybox TA 的容器和调用者检查完全不同；
- 不能把 Nothing 的补丁逻辑直接移植过来。

---

## 2.2 从启动链到 TA 的完整链路

~~~text
解锁状态下的启动链研究环境
    ↓
允许加载匹配版本的 TEE 镜像
    ↓
TEE 启动 L4Re/安全管理模块
    ↓
uTsecManager 载入 Keybox TA
    ↓
普通世界 TEE client 打开 session
    ↓
发送只读 Keybox 请求
    ↓
TA 校验 command、paramTypes 和 caller mode
    ↓
读取 Keybox secure object
    ↓
返回 descriptor + payload
    ↓
主机解析 DER 和证书链
~~~

这里需要区分两层问题：

1. 修改后的 TEE/TA 能否被加载；
2. 加载成功后，Keybox TA 是否允许当前调用者进入读取 handler。

前者属于启动链和 TA 容器问题，后者属于 TA 自身的调用者门问题。

---

## 2.3 TA 容器为什么不能简单重打包

Redmi 11T Pro 的 TA 不是裸 ELF，而是由加密载荷和验签封装组成。

这条路线只对记录中的精确系统构建成立。原始材料把目标锁定到具体版本、LK、TEE、Keybox TA 和 uTsecManager 摘要；同型号换 OTA、换槽位或换另一台设备，都必须重新采集和反汇编。这里的“版本匹配”不是文档要求，而是补丁能否进入同一条加载路径的前提。

可以抽象为：

~~~text
TA 文件
  ├─ 加密的 ELF 载荷
  ├─ 容器描述
  ├─ 对称 key/IV 封装
  └─ 签名或摘要材料
~~~

uTsecManager 负责：

1. 读取 TA 容器；
2. 使用内部 RSA 材料处理封装；
3. 还原 AES-CTR key/IV；
4. 解密 ELF；
5. 校验载荷摘要；
6. 将 TA 放入安全世界执行。

研究中没有原始 TA 签名私钥，因此修改后的 TA 的 OEM 签名不能真正变成有效签名。

可利用的研究边界不是“删除整个验签流程”，而是：

- 保留容器解析；
- 保留对称材料解封；
- 保留 AES-CTR 解密；
- 只处理验签失败后的错误分支；
- 确保载荷仍然与精确版本匹配；
- 不允许任意版本的 TA 被接受。

这也是为什么 uTsecManager、TA、TEE 镜像和目标槽位必须锁定到同一份版本资料。

---

## 2.4 Keybox TA 的调用者门

Keybox TA 的只读路径存在多层检查：

~~~text
命令类型正确
    ↓
TEE 参数类型正确
    ↓
caller mode 满足 TA 预期
    ↓
进入 Keybox read handler
~~~

普通 Linux TEE client 可以满足命令和参数格式，但不一定满足 TA 预期的 caller mode。

因此研究补丁的目标不是把所有检查删除，而是：

- 保留命令类型检查；
- 保留参数类型检查；
- 只放宽当前只读路径的调用者门；
- 不修改 Keybox 写入路径；
- 不修改安全对象；
- 运行时临时覆盖，重启恢复原厂 TA。

这点非常关键。如果把整个 dispatcher 的检查逻辑全部取消，得到的就不再是“只读 Keybox 研究”，而是一个无法评估边界的任意 TA 调用面。

---

## 2.5 运行时覆盖而不是永久改写

记录中的安全边界是运行时覆盖：

~~~text
原厂 vendor TA
    ↓
启动后建立临时覆盖
    ↓
研究客户端打开补丁 TA
    ↓
完成只读请求
    ↓
重启后覆盖自动消失
~~~

这种方式有几个优点：

- 不改写 vendor 分区；
- 原厂 TA 文件保留；
- 失败后可通过重启清理；
- 可以对比原厂和修改版摘要；
- 方便记录前后状态。

但它也有局限：

- 服务可能已经打开旧 TA session；
- 需要处理 KeyMint 服务重启；
- bind mount 状态不能只看文件路径；
- 读取到的 TA 不一定是当前进程实际使用的 TA；
- 重启前后安全世界状态不同。

所以“文件路径被覆盖”不能直接证明“目标请求走到了补丁 TA”。

---

## 2.6 descriptor 和 payload 的格式

该设备的返回结果由两个部分组成：

~~~text
descriptor
  ├─ magic
  ├─ payload length
  ├─ 两组 key/certificate entry
  ├─ 每个 entry 的 offset/size/count
  └─ payload digest

payload
  ├─ 长度前缀
  ├─ EC private DER
  ├─ EC certificate chain
  ├─ 长度前缀
  ├─ RSA private DER
  └─ RSA certificate chain
~~~

实际成功记录中的 descriptor 为 168 字节，payload 为 8578 字节，并且 descriptor 与 payload 使用同一组完整性关系进行核对。这里的数值只描述该轮样本，不代表所有版本固定使用相同长度。

这两个数字本身不是成功证明。它们只能作为结构门禁的一部分，必须继续完成 DER、SPKI 和证书链验证。

解析器不能只读取 descriptor 中的 offset，因为 payload 内部还存在长度前缀。正确方法是：

1. 解析 descriptor；
2. 验证 magic 和保留字段；
3. 验证 payload 长度；
4. 验证 descriptor 中的摘要；
5. 读取 payload 内部长度前缀；
6. 用 DER 内部长度再次确认；
7. 逐项检查证书数量和边界；
8. 对每一条链执行密码学验证。

---

## 2.7 Redmi 11T Pro 的证书验收

最终结果至少需要满足：

~~~text
descriptor magic 正确
    ↓
descriptor/payload 摘要一致
    ↓
所有 entry 没有越界
    ↓
EC/RSA 私钥 DER 可解析
    ↓
private key SPKI == leaf certificate SPKI
    ↓
leaf -> intermediate -> root 逐级签名有效
~~~

仅有客户端返回值、载荷长度或 PEM 文件，都不足以证明成功。

如果私钥和证书公钥不一致，常见原因包括：

- 解析偏移错位；
- 使用了错误版本的 descriptor；
- payload 截断；
- 把另一轮结果混进当前目录；
- 证书顺序错了；
- 读取到的是测试数据；
- TA 实际没有被补丁版本替换。

---

## 2.8 Redmi 11T Pro 踩过的坑

### 坑一：同型号不等于同版本

所有补丁都绑定精确系统版本、LK、TEE、Keybox TA 和 uTsecManager 摘要。

OTA、降级、内核变化、槽位变化或换另一台同型号设备，都需要重新反汇编和生成补丁。

### 坑二：只修改 TA，不处理加载器

修改后的 TA 如果 OEM 签名不匹配，uTsecManager 可能在载荷解密之前就拒绝它。

### 坑三：只处理验签，不保留容器逻辑

如果连容器解析、AES-CTR 解密或摘要校验一并破坏，TA 即使“被加载”也不会进入正常 handler。

### 坑四：把 caller mode 门全部删除

正确研究目标是放宽只读路径的特定调用者门，而不是删除所有命令检查。

### 坑五：把运行时覆盖当成永久安装

bind mount 只是当前运行时状态。重启后原厂 vendor TA 会恢复。反过来，重启前如果服务仍持有旧 session，也不能只凭路径判断当前使用的代码。

### 坑六：只写一个槽位却没有恢复验证

实验只应使用明确的候选槽位，并保留原厂槽位和完整备份。回滚后必须验证分区摘要，不能用“能开机”替代一致性验证。

---

## 2.9 Redmi 11T Pro 的完整闭环

~~~text
精确版本基线
    ↓
TEE/TA 容器静态解析
    ↓
保留解密和容器解析
    ↓
只放宽特定只读 caller gate
    ↓
运行时临时覆盖 TA
    ↓
打开真实 TEE session
    ↓
读取 descriptor + payload
    ↓
解析 EC/RSA DER
    ↓
私钥/叶证书公钥匹配
    ↓
证书链验证
    ↓
重启恢复原厂 TA
    ↓
分区和服务状态验收
~~~

这条路线说明：真正的研究对象不是某个 Keybox 文件，而是“启动链、TA 容器、调用者门、载荷格式和恢复边界”的组合。

---

---

[返回项目首页](../../README.md) · [完整详细原理报告](../../详细原理报告_zh-CN.md)
