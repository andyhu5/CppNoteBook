load_windows_root_certs : 装载Windows系统证书，放入store,使得OpenSSL能读取。

```c++
#include <wincrypt.h>
#pragma comment(lib, "crypt32.lib")

void load_windows_root_certs(ssl::context& ctx)
{
    HCERTSTORE hStore = CertOpenSystemStoreW(0, L"ROOT");
    if (!hStore) return;

    X509_STORE* store = X509_STORE_new();
    PCCERT_CONTEXT pContext = nullptr;

    while ((pContext = CertEnumCertificatesInStore(hStore, pContext)) != nullptr)
    {
        const unsigned char* encoded = pContext->pbCertEncoded;
        X509* x509 = d2i_X509(nullptr, &encoded, pContext->cbCertEncoded);
        if (x509)
        {
            X509_STORE_add_cert(store, x509);
            X509_free(x509);
        }
    }

    CertFreeCertificateContext(pContext);
    CertCloseStore(hStore, 0);

    SSL_CTX_set_cert_store(ctx.native_handle(), store);
}
```

这段代码的功能描述基本正确，但可以说得更精确一点：

**它不是"创建"证书，而是把 Windows 系统已经信任的根证书,"搬运"到 OpenSSL 的信任列表里。**

具体流程：

1. **`CertOpenSystemStoreW(0, L"ROOT")`** — 打开 Windows 系统的 "受信任的根证书颁发机构" 存储区（就是你在 Windows 里通过 `certmgr.msc` 能看到的那个）。这些证书是 Windows Update 维护的，操作系统已经信任它们。
2. **`X509_STORE_new()`** — 创建一个 **空的** OpenSSL 证书信任列表（这一步才是"创建"，但创建的是一个空容器，不是证书本身）。
3. **`CertEnumCertificatesInStore` 循环** — 遍历 Windows 根证书存储里的每一张证书。
4. **`d2i_X509(...)`** — 关键转换步骤：Windows 证书是以 **DER 编码**的二进制格式存储的（`pbCertEncoded`），这一步把这段 DER 字节流**解析**成 OpenSSL 能识别的 `X509*` 对象。这不是"创建"新证书，而是把同一张证书从"Windows 的表示形式"转换成"OpenSSL 的表示形式"。
5. **`X509_STORE_add_cert(store, x509)`** — 把转换好的证书加入到 OpenSSL 的信任列表 `store` 中。
6. **`SSL_CTX_set_cert_store(ctx.native_handle(), store)`** — 把这个信任列表绑定到你的 `ssl::context`，之后握手验证时 OpenSSL 就会用这个列表去校验对方证书链。

所以更准确的说法是：

> **把 Windows 系统信任的根证书列表，逐一转码后导入到 OpenSSL 的信任存储中，让 Boost.Asio/OpenSSL 的 TLS 握手也能使用 Windows 已经信任的 CA 列表。**

### 一个需要注意的小问题（内存泄漏风险）

`CertEnumCertificatesInStore` 在你把上一个 `pContext` 传入调用时，会自动帮你释放它，这是文档规定的行为——**除了最后一次返回 `nullptr` 之前那个 `pContext`**。你代码里循环结束后写了：

```cpp
CertFreeCertificateContext(pContext);
```

但循环退出时 `pContext` 已经是 `nullptr` 了（因为循环条件是 `while ((pContext = ...) != nullptr)`），所以这行 `CertFreeCertificateContext(nullptr)` 实际上什么都没做，是无害但多余的一行——这里没有泄漏，逻辑是对的，只是这行代码有点误导人，可以删掉或者加个注释说明。

真正需要注意的是：**如果 `X509_STORE_new()` 之后、`SSL_CTX_set_cert_store` 之前的某一步抛异常或提前 return（比如 `d2i_X509` 一直失败），`store` 会泄漏**，因为没有 RAII 包装。对于这种一次性初始化函数影响不大，但如果想写得更严谨，可以用 `unique_ptr` 加自定义删除器包装 `store`。