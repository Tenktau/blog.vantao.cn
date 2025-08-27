---
title: 当 AES-256 遇上 SHA256 —— AES 加密中密钥派生策略对兼容性的影响
description: 一次密钥处理陷阱的排查实录…
date: 2025-08-28 +0800
categories: [Development, Software]
tags: [AES, SHA256, Cryptology]
---

> **环境**: Python 3.11  
> **IDE**: PyCharm 2025.1.1.1 (Community Edition)

## 1 问题发现

今天在测试自研的 AES-256 ECB 实现时，遇到了一个有趣的问题。我的代码逻辑看起来完美无缺：完整的轮函数、正确的填充机制、标准的 S 盒生成，但加密结果却与[在线 AES 加解密工具](https://www.toolhelper.cn/SymmetricEncryption/AES)不一致。

我所使用的测试用例如下：

```plain
明文："Hello, AES-256!"
密钥："thisisa32bytestestkeyforaes256!!"
```

## 2 问题定位

经过排查和 LLM 的帮助，发现我的密钥处理部分代码有一个画蛇添足的行为，如下列第四行所示：

```py
def derive_key(key_or_password: bytes | str) -> bytes:
    if isinstance(key_or_password, str):
        key_or_password = key_or_password.encode("utf-8")
    return hashlib.sha256(key_or_password).digest()  # 始终32字节
```

我总是对输入密钥进行 SHA256 散列（尽管密钥本身已经满足 32 字节），以确保得到 32 字节的密钥。但大多数标准 AES 实现直接使用提供的密钥字节。

这就是兼容性问题的根源：

因此我的密钥经过 SHA256 散列之后被转变为了不同的 32 字节输入：

`hashlib.sha256("thisisa32bytestestkeyforaes256!!").digest() != thisisa32bytestestkeyforaes256!!`

## 3 解决方案

我决定提供一个灵活的解决方案，而不是简单地移除散列功能：

```py
# 添加配置选项
HASH_KEY = True  # 默认为True保持向后兼容

def derive_key(key_or_password: bytes | str) -> bytes:
    if isinstance(key_or_password, str):
        key_or_password = key_or_password.encode("utf-8")
    
    if HASH_KEY:
        # 使用SHA256散列得到32字节密钥
        return hashlib.sha256(key_or_password).digest()
    else:
        # 直接使用提供的字节密钥
        if len(key_or_password) != 32:
            raise ValueError("AES-256要求精确的32字节密钥")
        return key_or_password
```

这样提供了两种使用模式：

- 密码模式 (HASH_KEY=True): 接受任意长度输入，通过散列得到固定长度密钥；
- 密钥模式 (HASH_KEY=False): 要求精确的 32 字节密钥，与标准实现兼容。

经测试，解决方案有效！又是 Debug 成功的一天~

~~（LLM 真好用啊）~~