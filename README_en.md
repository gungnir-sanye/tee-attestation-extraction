# Android TEE Keybox and RKP research archive

[中文](README.md) | English

The current `main` branch contains technical post-mortems for three devices: principles, evidence relationships, and curated research records. Raw device dumps, certificate private keys, seed values, patch binaries, vendor firmware, and executable extraction scripts are excluded from this branch.

AI organized these documents after reading the source project. The historical steps have not all been independently rerun by a human. Each claim should be checked against its evidence; the text does not replace authorization, build checks, or safe operational judgment.

## Device documents

| Device | Recorded result | 中文 | English |
|---|---|---|---|
| Nothing 2a | Keybox and RKP-derived material research | [教程](01_Nothing2A_RKP种子_Keybox/01_教程/教程_zh-CN.md) · [流水](01_Nothing2A_RKP种子_Keybox/02_流水记录/流水记录_zh-CN.md) · [原理](01_Nothing2A_RKP种子_Keybox/03_原理/原理_zh-CN.md) | [Tutorial](01_Nothing2A_RKP种子_Keybox/01_教程/Tutorial_en.md) · [Log](01_Nothing2A_RKP种子_Keybox/02_流水记录/Practice_Log_en.md) · [Principles](01_Nothing2A_RKP种子_Keybox/03_原理/Principles_en.md) · [Artifacts](01_Nothing2A_RKP种子_Keybox/04_补丁镜像固件/README_en.md) |
| Redmi 11T Pro | Keybox research | [教程](02_Redmi11TPro_Keybox/01_教程/教程_zh-CN.md) · [流水](02_Redmi11TPro_Keybox/02_流水记录/流水记录_zh-CN.md) · [原理](02_Redmi11TPro_Keybox/03_原理/原理_zh-CN.md) | [Tutorial](02_Redmi11TPro_Keybox/01_教程/Tutorial_en.md) · [Log](02_Redmi11TPro_Keybox/02_流水记录/Practice_Log_en.md) · [Principles](02_Redmi11TPro_Keybox/03_原理/Principles_en.md) · [Artifacts](02_Redmi11TPro_Keybox/04_补丁镜像固件/README_en.md) |
| Redmi K70E | RKP/DICE identity research | [教程](03_RedmiK70E_RKP种子/01_教程/教程_zh-CN.md) · [流水](03_RedmiK70E_RKP种子/02_流水记录/流水记录_zh-CN.md) · [原理](03_RedmiK70E_RKP种子/03_原理/原理_zh-CN.md) | [Tutorial](03_RedmiK70E_RKP种子/01_教程/Tutorial_en.md) · [Log](03_RedmiK70E_RKP种子/02_流水记录/Practice_Log_en.md) · [Principles](03_RedmiK70E_RKP种子/03_原理/Principles_en.md) · [Artifacts](03_RedmiK70E_RKP种子/04_补丁镜像固件/README_en.md) |

## Detailed principles report

[中文：三台设备从漏洞链到证书验收](详细原理报告_zh-CN.md) | [English: Detailed Principles Report](Detailed_Principles_Report_en.md)

## Donations and support

If this technical archive is useful to your research, you are welcome to use the public channels below to support future maintenance and research. Donations do not provide restricted vulnerability material, real keys, or device-extraction services. Confirm the network and asset type before sending; blockchain transfers are usually irreversible, so a small test transfer is recommended.

### Platform donation

- [Ko-fi donation page](https://ko-fi.com/sanyegungnir)

### Telegram technical discussion

- [Join the Telegram channel](https://t.me/keybox_playintegrityfix) (channel: @keybox_playintegrityfix)
- The channel is for related technical discussion and experience sharing. It does not promise restricted vulnerability material, real keys, or device-extraction services.

### WeChat technical discussion group

Group name: Google Play Integrity Research. Scan the QR code below with WeChat to join:

![Google Play Integrity Research WeChat group QR code](assets/community/wechat-group-google-play-integrity.png)

This QR code is valid for seven days, until October 23. When it expires, revisit this page to obtain an updated QR code.

### WeChat support

![WeChat support QR](assets/donations/wechat.jpg)

The maximum single appreciation is 200 CNY, subject to the rules shown by WeChat at the time of payment.

### Cryptocurrency donations

| Network | Address | Asset note |
|---|---|---|
| Ethereum Mainnet | `0x04989a778AC28A60Cc5eF68DCaea379633E0189e` | ETH / ERC-20; confirm the receiving asset |
| Arbitrum | `0x04989a778AC28A60Cc5eF68DCaea379633E0189e` | ETH / ERC-20 on Arbitrum One; select the correct network |
| TRON Mainnet | `TScTH4VifRKja3NSvaWcPXmb99y3JjgYQ3` | TRX / TRC-20; confirm the receiving asset |
| Bitcoin Mainnet | `bc1q886gc0mt0w9rre3zjct7vvvyrryerm7p2k9nfc` | BTC, native SegWit `bc1q` address |

| Network | QR code |
|---|---|
| Ethereum Mainnet | ![Ethereum donation QR](assets/donations/ethereum.png) |
| Arbitrum | ![Arbitrum donation QR](assets/donations/arbitrum.png) |
| TRON Mainnet | ![TRON donation QR](assets/donations/tron.png) |
| Bitcoin Mainnet | ![Bitcoin donation QR](assets/donations/bitcoin.png) |

Important notes:

- Ethereum and Arbitrum use the same address string but are different networks; select the correct network when sending.
- Use the TRON address only on the TRON network; do not send ERC-20, BEP-20, or other network assets to it.
- The Bitcoin address accepts BTC only; do not send tokens from another chain.
- Cryptocurrency addresses are public and on-chain transfers are generally traceable, so this is not fully anonymous.
- The project will never ask for a seed phrase, private key, verification code, or extra fee by private message.

## Use boundaries, license, and disclaimer

This repository is for research, education, CTF exercises, and defensive review on devices you own or are expressly authorized to examine. Publication does not grant access to third-party devices, services, accounts, or identity material, and does not imply endorsement by any device vendor, platform, or certificate issuer. Users must obtain the necessary authorization and comply with applicable law, service terms, and third-party rights. Do not use this material to impersonate device identities, obtain or distribute another party's keys or certificates, or disrupt production services.

AI organized the documents from historical project material. The steps and conclusions have not been independently rerun or verified one by one by a human. Versions, technical judgments, and results may be wrong or outdated. Before acting, users must independently check the material, environment, risks, and recovery conditions and are responsible for their own actions. These documents provide no safety, legal, or compliance assurance.

Original documentation is licensed under [CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/); see [LICENSE](LICENSE). The license covers only material the publisher has authority to license. It grants no rights to vendor firmware, software, trademarks, keys, certificates, or other third-party material. The material is provided “as is” and “as available,” without warranties of accuracy, completeness, security, fitness, or non-infringement. To the fullest extent permitted by applicable law, the publisher is not liable for losses arising from reading, downloading, using, modifying, or sharing this material. Liabilities that cannot lawfully be excluded remain unaffected.
