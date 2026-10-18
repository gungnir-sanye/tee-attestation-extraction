# 【原创】Android TEE Keybox 与 RKP 证书提取复盘：三台设备从漏洞链到证书验收

> 设备：Nothing 2a、Redmi 11T Pro、Redmi K70E  
> 关键词：KeyMint、TEE、Trustlet、TA、Keybox、RKP、DICE、CDI、CDI_Leaf、UDS、CSR、证书链、逆向分析

---

## 0. 文章口径

这不是一篇把命令复制过去就能复现的教程，而是三台实际研究设备的技术复盘。

每章都回答五个问题：

1. 漏洞或安全边界的薄弱点在哪里；
2. 普通世界的请求怎样进入 TEE；
3. TEE 内部的哪一段数据被观测或导出；
4. 主机侧怎样把结果整理成证书材料；
5. 实验中具体踩了哪些坑，为什么之前的判断会错。

公开版本不包含真实私钥、真实 RKP seed、设备唯一标识、补丁字节和可直接执行的刷写命令。文中的流程、数据结构和验收关系均来自既有工程记录；完整工程由 AI 阅读整理，并不代表每一条历史命令都已经人工重新执行。

本文仅用于自有或明确授权设备的研究和防御性复核。公开技术说明不授予访问第三方设备、服务或身份材料的权限；操作前须自行确认授权、法律与服务条款、版本条件及恢复方案。技术结论可能有误或过时，材料按现状提供，责任限制见 [LICENSE](LICENSE)，且仅在适用法律允许的范围内有效。

---

## 1. 先给最终结论

三台设备的研究不能归纳成一句“把 seed 提出来”。

实际得到的是三种不同问题：

~~~text
Nothing 2a
  本地 Factory Keybox 解密明文的受控观测
  + 一条独立的 RKP 派生材料路径

Redmi 11T Pro
  MediaTek TEE Keybox 只读命令的调用者门和 TA 容器边界
  + descriptor/payload/证书链的完整验收

Redmi K70E
  Proper DICE 下的 final CDI_Leaf 身份确认
  + 同轮 CSRv3、DeviceInfo 和 Google 证书链闭环
~~~

最容易犯的三个错误是：

- 把 Keybox、RKP seed、CDI、UDS 和 CDI_Leaf 当成同一个秘密；
- 把某次成功启动、某段 32 字节输出或一次 HTTP 200 当成最终结果；
- 把一台设备上的 TA、TEE、槽位和 DICE 结论移植到另一台设备。

---

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

# 第三章 Redmi K70E：Proper DICE、CDI_Leaf 与 Google RKP 证书

## 3.1 研究目标发生过变化

K70E 最初的目标是找到更早的 RKP seed 或 UDS 根材料。

但在实际研究中逐步发现：

- raw CDI、CDI_Leaf 和 native per-key scalar 不是同一个对象；
- final CDI_Leaf 会受到启动测量影响；
- 修改 TEE 或启动组件可能改变后续 DICE 链；
- 某个 32-byte 值即使能派生公钥，也不一定是 UDS；
- 早期阶段越早，越可能污染后续测量；
- 只拿到一个身份值并不能自动构造 Google 可接受的请求。

因此研究路线从“直接提最早 root”转向：

1. 先建立 stock RKP 基线；
2. 区分 raw CDI 和 final CDI_Leaf；
3. 在同一 secure-world boot 中捕获 final leaf；
4. 用同轮 DICE/DeviceInfo 构造软件请求；
5. 生成一把新的软件 P-256 私钥；
6. 获取与这把私钥匹配的 Google 证书链；
7. 恢复设备到安全基线。

---

## 3.2 K70E 的 DICE 形态

K70E 的记录更接近完整的 Proper DICE：

~~~text
UDS 根
    ↓
阶段一测量
    ↓
阶段二测量
    ↓
TEE / KeyMint 测量
    ↓
最终 CDI_Leaf
    ↓
RKP CSRv3
~~~

抽象的派生关系是：

~~~text
CDI_next = KDF(CDI_current, measurement, code_id, configuration)
~~~

这意味着 final CDI_Leaf 不是固定不变的“机型种子”。

如果改变：

- preloader；
- LK；
- TEE；
- init_boot；
- 活动槽；
- 测量配置；
- 安全世界加载状态；

那么后续 CDI 可能改变。

所以历史 leaf 不能硬编码到新轮次，旧设备的身份也不能直接复制到另一台 K70E。

---

## 3.3 raw CDI、final CDI_Leaf 和 native per-key scalar

K70E 研究中最容易混淆的三类 32-byte 材料是：

### 第一类：raw CDI

它是进入后续 DICE 或 KeyMint 派生路径的输入之一。

它不一定是：

- 最终 P-256 私钥；
- Google CSR 直接使用的身份；
- UDS；
- 可以直接交给软件 RKP 工具的 seed。

### 第二类：final CDI_Leaf

它是在 DICE 派生完成后，真正参与当前身份签名的最终 leaf 私钥标量。

当前软件 RKP 路线使用的是这一层，而不是随便抓到的 raw CDI。

### 第三类：native per-key scalar

这是 KeyMint 为某一把具体 RKP 证明密钥生成的私钥标量。

它可以与对应 MacedPublicKey、叶证书和签名闭环，但它不是 CDI_Leaf 身份。

正确关系应该写成：

~~~text
raw CDI
    ↓
DICE / KeyMint 派生
    ↓
final CDI_Leaf identity scalar
    ↓
签署设备身份 CSR

另一路：
KeyMint 生成新的 P-256 attestation key
    ↓
KeysToSign
    ↓
Google 返回该公钥的 RKP 叶证书
~~~

这两把私钥不应该相等。一个负责证明“请求来自设备身份”，另一个负责承载后续 Android Key Attestation。

---

## 3.4 TEE exact-frame 研究路线

K70E 的 TEE 研究有一个非常重要的经验：

> 修改 TEE 不等于可以任意重打包 TEE。

这里也要把“签名被绕过”说准确。原始 TA 的 OEM RSA 签名字节没有被伪造，验签器本身仍然运行，body/header digest 也仍然保留。研究修改的是 SProvider 对验签返回值的错误分支：返回非零时不再立即阻止后续加载，但容器解析、解压和长度契约仍然必须成立。

所以这不是“破解 OEM 私钥”，而是一个加载器错误处理边界问题：

~~~text
RSA 验签结果 = failure
    ↓
原始逻辑：进入拒绝分支
研究逻辑：继续执行后续解析
~~~

若把它写成“已经伪造 TA 签名”，会同时把漏洞位置和实际能力说错。

早期全量重压缩候选破坏了 LZ4 frame 尾部的证书、DTB 或压缩块边界，设备卡在 logo。后来才收敛到 exact-frame 方案：

~~~text
保留原始 TEE frame 结构
    ↓
只改变目标指令附近的最小字节
    ↓
保持解压后内容长度不变
    ↓
保持其他压缩 block 不变
    ↓
保持 frame 尾部、证书和 DTB 不变
    ↓
重新验证镜像结构和摘要
~~~

研究目标也不是解决 OEM TA 签名，而是：

- 保留 body/header digest；
- 让目标 SProvider 路径继续完成解析；
- 只改变失败结果对后续分支的影响；
- 不破坏加载契约和 frame 尾部。

这解释了为什么早期“所有 TEE 补丁都会卡 logo”的结论是错误的：真正的问题是压缩 frame 和加载契约被破坏，而不是所有最小补丁都不可行。

---

## 3.5 one-shot early-bind 与自动恢复

K70E 的动态实验不能采用“刷入后永久保留修改 TEE”的方式。

研究记录中的安全设计是：

~~~text
stock TEE 恢复件预置
    ↓
只在目标设备、目标槽位、目标摘要全部匹配时触发
    ↓
当前启动使用候选 TEE
    ↓
捕获同轮 CDI_Leaf 和 CSR
    ↓
系统完成启动后自动把磁盘 TEE 恢复为 stock
    ↓
不重启当前 secure world
    ↓
下一次启动回到 stock TEE
~~~

这个设计带来一个容易误判的现象：

> 启动完成后磁盘上的 TEE 已经是 stock，并不表示刚才候选 TEE 没有运行。

自动恢复的目的就是让磁盘回到安全状态，同时保留当前 secure-world boot 的研究结果。

---

## 3.6 同轮身份捕获

最终身份捕获必须满足严格顺序：

~~~text
stock/approved 研究环境
    ↓
native zero-key CSR
    ↓
验证同轮 DICE chain、outer signature、DeviceInfo
    ↓
捕获 final CDI_Leaf scalar
    ↓
派生 SPKI
    ↓
与同轮 native CSR 的 leaf SPKI 做常数时间比较
    ↓
只在匹配时保存身份
~~~

不能这样做：

~~~text
先抓一个 32-byte 值
    ↓
过几天再找一份 CSR
    ↓
把两者拼到一起
    ↓
希望 Google 接受
~~~

因为 CDI_Leaf 受测量绑定，CSR、DICE、DeviceInfo、challenge 和身份标量必须来自同一轮。

---

## 3.7 google-rkp-sw 的两把钥匙

软件签发阶段存在两类私钥。

### 设备身份私钥

用途：

- 签署 fresh challenge；
- 签署 CSR 身份数据；
- 证明请求来自设备的 DICE/CDI 身份。

### 软件生成的 P-256 私钥

用途：

- 作为 KeysToSign 中的公钥；
- 接收 Google 返回的 RKP 叶证书；
- 后续用于 Android Key Attestation。

完整关系是：

~~~text
CDI_Leaf identity private
    ↓
签署设备身份 CSR
    ↓
Google 验证 DICE 和 DeviceInfo

software P-256 private
    ↓
public key enters KeysToSign
    ↓
Google signs this public key
    ↓
leaf certificate public key == software private public key
~~~

如果错误地要求两把私钥相等，会把正确结果判成失败。

---

## 3.8 RKP 请求的对象关系

K70E 研究最终围绕这些对象建立关系：

~~~text
AuthenticatedRequest
  ├─ outer signed data
  ├─ DICE certificate chain
  ├─ DeviceInfo
  ├─ challenge
  └─ CSR payload

CSR payload
  ├─ request version
  ├─ keymint profile
  ├─ keysToSign
  └─ canonical encoded fields
~~~

关键验证关系：

~~~text
同轮 CDI_Leaf 公钥
    ==
AuthenticatedRequest 签名身份

software private public key
    ==
KeysToSign 公钥

software private public key
    ==
Google 返回叶证书公钥

叶证书
    ↓
中间 CA
    ↓
根 CA
~~~

同时还要检查：

- CBOR 是否 canonical；
- CSRv3 是否混入了不属于该版本的字段；
- DeviceInfo 是否与目标 profile 一致；
- EEK 和 challenge 是否是 fresh；
- 返回证书是否完整消费；
- 是否存在重复 leaf、断链、环或额外游离证书。

---

## 3.9 K70E 的实际踩坑

### 坑一：把 native per-key scalar 当作 CDI_Leaf

native per-key scalar 能和单把 RKP 证明密钥闭环，但它不是设备身份。

### 坑二：把 raw CDI 直接交给软件工具

raw CDI 还没有经过当前设备对应的 DICE/KeyMint 派生，不能直接当作 final P-256 identity scalar。

### 坑三：把两把私钥要求成同一个

身份私钥和待签发 P-256 私钥承担不同角色，本来就应该不同。

### 坑四：CSRv3 混入旧版本字段

不同 CSR 版本的 outer structure 和 DeviceInfo 字段不能混用。一个看似合理的字段放错层级，也会让服务端返回 400。

### 坑五：把完整 AuthenticatedRequest 当成 bare DICE chain

DeviceInfo 和 outer signed data 的可信关系不能被删掉。只保存一段裸 DICE chain，可能失去服务端需要的签名上下文。

### 坑六：把修改后的 DICE 一定判定为拒绝原因

早期某轮 modified DICE 与错误 DeviceInfo 同时变化，因此只能证明相关性。

后来使用 exact modified DICE 和 clean DeviceInfo 得到 APPROVED，说明不能把“DICE 改了”直接写成唯一因果。

### 坑七：旧解析器丢掉未知 Provisioning Info

早期验收器没有保留证书中未知的 Provisioning Info 字段，曾把包含 NOT_APPROVED 的证书误判为 PASS。

正确做法是：

- 保留未知字段；
- 对 provisioning status 做显式门禁；
- NOT_APPROVED 不能被摘要或证书链通过覆盖。

### 坑八：把自动恢复后的 stock TEE 当作没有运行候选 TEE

动态恢复目标是磁盘安全，不是让当前 secure world 立即重启。必须同时看启动标记、运行时 namespace、TA 摘要、恢复记录和 boot ID。

### 坑九：全量重压缩 TEE

只要破坏 LZ4 frame、证书尾窗、DTB 或固定容量，就可能卡 logo。exact-frame 的关键是“最小改动、长度不变、其余内容逐字节保留”。

---

## 3.10 NOT_APPROVED 的因果纠偏

K70E 这条线最容易被写错的地方，是把一次 NOT_APPROVED 直接归因到“DICE 被修改”。

旧轮次同时改变了多个变量：

~~~text
modified DICE
    +
历史或不匹配的 DeviceInfo
    +
新的 challenge、request 和服务端状态
~~~

因此旧轮只能说明这一组组合没有通过，不能证明 modified DICE 是唯一原因。

后续对照实验保留同一组 exact modified DICE，只替换为与目标设备 profile 一致的 clean DeviceInfo，记录中出现了 APPROVED。这个结果至少说明：

- 不能把“DICE 变了”写成充分拒绝条件；
- DeviceInfo 的完整 tuple 和服务端登记 profile 可能是重要门禁；
- 只能把该对照描述为组合因素被拆开后的证据；
- 没有做单字段 A/B，就不能声称已经证明某一个字段是唯一根因。

同样，证书中的 Provisioning Info 未知字段不能被旧解析器静默丢弃。早期解析器曾把包含 NOT_APPROVED 的证书当成 PASS，根因不是密码学验签错误，而是验收器丢失了服务端状态字段。

因此最终验收必须把下面两件事分开：

~~~text
X.509 / DICE 数学签名有效
    ≠
服务端 provisioning 状态合格
~~~

只有两层都通过，才能进入最终提交。

---

## 3.11 K70E 的最终证书闭环

K70E 记录中最终闭环的逻辑顺序是：

~~~text
同轮 stock/approved 研究环境
    ↓
捕获 final CDI_Leaf
    ↓
派生身份 SPKI 并与 native zero-key CSR 比较
    ↓
主机生成新的 P-256 软件私钥
    ↓
复用同轮完整 DICE/DeviceInfo source
    ↓
获取 fresh challenge/EEK
    ↓
构造 canonical CSRv3
    ↓
远程 fetch/sign
    ↓
得到一条完整证书链
    ↓
叶证书公钥 == 软件私钥公钥
    ↓
逐级验证到唯一自签根
    ↓
保存 CSR、响应、私钥和证书链
    ↓
恢复设备到 stock 基线
~~~

这里的最终产物不是“把 native KeyMint 私钥导出来”，而是：

- 同轮设备身份；
- 一把新生成的软件 P-256 私钥；
- Google 为该公钥签发的完整证书链；
- 可复核的 CSR、响应、证书和恢复证据。

---

# 4. 三台设备的共同经验

## 4.1 漏洞入口和结果出口不是同一个问题

漏洞入口可能是：

- 启动链允许研究组件加载；
- TA 容器错误处理验签失败；
- caller mode 与命令权限绑定不严；
- TEE 内部明文生命周期存在可观测点；
- CSR 路径没有完成密钥来源绑定。

结果出口则是：

- mapped buffer；
- descriptor + payload；
- CSR；
- SignedData；
- Google 返回证书链。

必须分别证明入口和出口，不能因为入口成立就自动认为出口有效。

## 4.2 静态分析不能代替动态证据

静态看到一个 KDF、一个 parser、一个复制函数，不等于实际运行时走到了这条路径。

动态结果还需要：

- 当前版本和摘要；
- 当前槽位；
- 当前 TA/TEE；
- 当前服务状态；
- 请求输入；
- 返回状态；
- 实际输出；
- 恢复状态。

## 4.3 证书链验收必须做关系验证

证书链通过，只能说明签名关系正确。

还要验证：

~~~text
私钥 == 叶证书公钥
CSR 身份 == 同轮 CDI_Leaf
DeviceInfo == 当前设备 profile
请求 == 当前 challenge
结果 == 当前运行生成
~~~

## 4.4 失败记录不能被最终成功覆盖

三台设备都出现过“看起来已经成功”的中间状态：

- 有输出但不是目标材料；
- 有私钥但不是设备身份；
- 有 HTTP 200 但证书状态不合格；
- 有启动但测量状态不正确；
- 有证书链但私钥对应关系不成立。

这些中间结果必须保留，并明确标记为旁证、失败或待复核。

---

# 5. 结语

三台设备的真正研究成果，不是几个可以复制的文件，而是把以下层次串起来：

~~~text
启动链加载边界
    ↓
TEE/TA 调用路径
    ↓
安全对象解密或身份派生
    ↓
明文/CSR/载荷观测点
    ↓
结构化解析
    ↓
私钥、公钥、证书链关系
    ↓
远程签发状态
    ↓
回滚和恢复证据
~~~

Nothing 2a 说明了 Trustonic KeyMint 明文如何在安全世界内部短暂出现，并通过受控观测点形成 Keybox 证据。

Redmi 11T Pro 说明了 MediaTek TEE 中 TA 容器、uTsecManager、caller mode 和 descriptor/payload 之间如何共同决定结果是否成立。

Redmi K70E 说明了 Proper DICE 下 raw CDI、CDI_Leaf、native per-key scalar 和软件签发私钥必须严格分层，并且最终证书必须绑定到同轮身份、同轮 DeviceInfo 和同轮 challenge。

如果缺少这些关系，提取到的只是一段字节；只有完成从漏洞入口到密码学验收的闭环，才可以称为一次完整的 TEE 研究结果。

本文和配套工程仅用于授权研究、CTF、安全分析和防御性复盘。请勿在未授权设备、生产环境或第三方身份材料上进行测试。

