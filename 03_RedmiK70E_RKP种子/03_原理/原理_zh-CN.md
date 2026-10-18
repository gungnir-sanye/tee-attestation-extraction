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

---

[返回项目首页](../../README.md) · [完整详细原理报告](../../详细原理报告_zh-CN.md)
