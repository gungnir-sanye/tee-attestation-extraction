# 第一章 Nothing 2a：Trustonic KeyMint 的 Keybox 明文外流

## 1.1 研究目标

Nothing 2a 同时出现了两条研究路线：

1. 本地 Factory Keybox 路线；
2. RKP 派生材料路线。

Factory Keybox 路线的最终目标是得到可以解析的 EC/RSA attestation keybox 以及对应证书链。

RKP 路线的目标是确认 KeyMint TA 内部某个 KDF 派生出来的 32-byte 材料，能否作为当前设备软件 RKP 身份的一部分使用。

两条路线共享底层 KeyMint 和 Trustonic TEE，但结果语义不同。

---

## 1.2 漏洞不是一个点，而是一条链

Nothing 2a 的 Keybox 研究链可以概括为：

~~~text
Bootloader 解锁状态
    ↓
启动链验证策略缺陷
    ↓
受控加载修改后的 LK/TEE
    ↓
Trustonic MCLF/Trustlet 加载边界被放宽
    ↓
修改后的 KeyMint Trustlet 进入安全世界
    ↓
TA 正常读取并解密 Factory Keybox secure object
    ↓
明文短时间出现在 TA 的栈/堆对象中
    ↓
在 parser 之前保存明文
    ↓
在 handler 返回前复制到 mapped buffer
    ↓
普通世界 direct client 读回缓冲区
    ↓
主机解析 DER、私钥和证书链
~~~

这里真正被利用的不是 attest_keybox 文件自身的加密算法，而是：

> 研究人员获得了在同一 TEE 环境加载修改后 Trustlet 的能力，同时修改后的 TA 在处理明文时增加了一个普通世界可观察的输出路径。

也就是说，秘密没有在普通文件系统中被“破解”，而是在安全世界中被正常解密后，通过新的数据流复制出来。

---

## 1.3 KeyMint 到 Trustlet 的数据流

正常调用链大致如下：

~~~text
KeyMint HAL
    ↓
Trustonic client
    ↓
打开 Trustlet session
    ↓
映射普通世界 buffer
    ↓
填写 TCI 请求
    ↓
通知安全世界
    ↓
等待 TA 返回
~~~

TCI 中主要包含：

- 命令类型；
- 返回状态；
- mapped buffer 的虚拟地址；
- mapped buffer 的长度；
- 当前请求上下文。

原始 HAL 只希望 TA 使用 mapped buffer 中的加密对象完成处理，stock TA 不会把安全世界明文写回原缓冲区。

研究用 direct client 复刻的是这条通信协议，而不是调用 Java Keystore API。它的作用是：

1. 建立 Trustlet session；
2. 把输入 secure object 映射给 TA；
3. 发送与 HAL 相同类别的处理请求；
4. 等待 TA 完成；
5. 读取调用前后的 buffer 差异。

未修改 TA 时，调用前后 buffer 保持不变；这恰好证明 stock 路径没有把明文外送。

---

## 1.4 TA 内部真正发生了什么

根据反汇编和流水记录，处理路径可以抽象为：

~~~text
handler
  ├─ 读取 mapped buffer 地址和长度
  ├─ 校验输入范围
  ├─ 将加密对象复制到 TA heap
  ├─ 调用 secure object 解密函数
  ├─ 得到 plaintext buffer
  ├─ 调用 Keybox parser
  ├─ 释放 plaintext
  └─ 返回状态
~~~

研究补丁并没有重新实现解密算法，也没有从外部猜测 Keybox 格式。

它只增加两个观测点：

~~~text
解密完成、parser 尚未调用
    ↓
保存 plaintext pointer 和 length

parser 完成、handler 尚未返回
    ↓
把保存的明文复制到 mapped buffer
~~~

这样做的好处是：

- 原始 decrypt 逻辑仍然执行；
- 原始 parser 仍然执行；
- 原始成功/失败返回值仍然保留；
- 只读请求不需要增加写入能力；
- 可以将“成功返回但内容错误”的情况排除。

---

## 1.5 明文 Keybox 的结构

明文不是直接的 XML，而是长度前缀序列。

逻辑结构类似：

~~~text
版本或 keybox 编号
EC private DER length
EC private DER
EC certificate count
EC certificate 0
EC certificate 1
EC certificate 2
RSA private DER length
RSA private DER
RSA certificate count
RSA certificate 0
RSA certificate 1
RSA certificate 2
~~~

解析时不能只按固定偏移切割，因为：

- 证书长度可能随版本变化；
- 证书链长度可能变化；
- 末尾可能保留未覆盖的原始缓冲区内容；
- 错误的长度解释会产生看似合法的 DER 片段。

可靠解析顺序是：

1. 读取长度前缀；
2. 检查剩余缓冲区是否足够；
3. 检查 DER 首字节和内部长度；
4. 检查 descriptor 与实际 payload 一致；
5. 对每个私钥导出 SPKI；
6. 与对应叶证书 SPKI 比较；
7. 验证完整证书链。

---

## 1.6 为什么能证明结果是 Factory Keybox

仅仅拿到 EC/RSA 私钥还不够。

必须完成三层证明：

### 第一层：私钥与叶证书一致

~~~text
PublicKey(private_EC) == PublicKey(ec_leaf_certificate)
PublicKey(private_RSA) == PublicKey(rsa_leaf_certificate)
~~~

### 第二层：证书链有效

~~~text
leaf signed by intermediate
intermediate signed by root
root self-signed and constraints valid
~~~

### 第三层：Android KeyMint 实际链路一致

由 Android KeyStore 触发新的 attestation 请求，将得到的叶证书与导出的 Factory Keybox 证书进行比对。

只有三层都通过，才能排除：

- 随机私钥；
- 其他用途的 Keybox；
- 错位解析；
- 测试 CA；
- 仅格式正确但不属于本设备的材料。

---

## 1.7 Nothing 2a 的 RKP 派生材料

RKP 路线不是从 Keybox 明文中直接取值。

研究记录显示，KeyMint TA 内部存在一条与 rkp_bcc_km 相关的 KDF 路径：

~~~text
设备绑定秘密
    ↓
带标签的 KDF
    ↓
32-byte 派生材料
    ↓
RKP/CDI 相关身份处理
    ↓
CSR 和 challenge 签名
~~~

这 32-byte 材料在当前设备和当前软件实现中可以作为 RKP 软件身份的一部分使用，但不能因此断言它是：

- 所有平台通用的 UDS；
- 所有设备通用的 raw CDI；
- 所有厂商相同格式的 CDI_Leaf；
- 可以跨设备直接复用的根种子。

正确表述应该是：

> Nothing 2a 当前 TA 路径中，rkp_bcc_km KDF 输出的 32-byte 派生材料，经同轮公钥、CSR 和签名关系验证后，可作为该设备当前软件 RKP 身份的输入。

---

## 1.8 Nothing 2a 踩过的坑

### 坑一：把 Keybox 当作普通文件

文件名和路径只能证明输入对象存在，不能证明其中存在可直接读取的明文。

### 坑二：只观察 client 返回码

TA 返回成功只说明解密和解析路径没有报错，不说明输出缓冲区是正确的 Keybox。

### 坑三：把尾部残留当成 Keybox 主体

复制长度不足时，mapped buffer 尾部可能保留原始输入残留。必须按长度前缀、DER 结构和证书关系解析，不能把整块 buffer 当成明文。

### 坑四：把 RKP fallback 当成 Keybox 不存在

设备可以优先使用 RKP，也可能在网络失败或服务不可用时回退到本地 Factory Keybox。RKP 存在不等于本地 Keybox 不存在。

### 坑五：按固定偏移解析所有版本

证书长度和容器字段可能变化。必须以长度前缀和 DER 内部长度为准，并把版本和输入摘要作为硬门禁。

---

## 1.9 Nothing 2a 的最终逻辑闭环

~~~text
能够加载受控 Trustlet
    ↓
TA 在安全世界正常解密 Keybox
    ↓
明文在 parser 前被观测
    ↓
mapped buffer 得到受控副本
    ↓
主机按长度前缀解析
    ↓
私钥与叶证书公钥匹配
    ↓
证书链验证
    ↓
Android KeyStore 实际链路交叉验证
~~~

这条链中任何一环缺失，都只能称为“看到了一段数据”，不能称为“提取了可用 Factory Keybox”。

---

---

[返回项目首页](../../README.md) · [完整详细原理报告](../../详细原理报告_zh-CN.md)
