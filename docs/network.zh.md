# 企业证书

[English](network.md) · [安装指南](installation.zh.md)

默认开启企业证书兼容，访问下表中的目标无需额外配置证书。

| pi-pbt 的 HTTPS 证书例外范围 | 域名 |
|---|---|
| 华为 | `huawei.com`、`*.huawei.com`、`repo.huaweicloud.com`、`mirrors.huaweicloud.com`、`ohpm.openharmony.cn` |
| npm / Node / PyPI | `registry.npmjs.org`、`nodejs.org`、`pypi.org`、`files.pythonhosted.org` |
| Debian / Ubuntu | `deb.debian.org`、`security.debian.org`、`archive.debian.org`、`archive.ubuntu.com`、`security.ubuntu.com`、`ports.ubuntu.com`、`old-releases.ubuntu.com` |
| GitHub | `github.com`、`api.github.com`、`codeload.github.com`、`raw.githubusercontent.com`、`objects.githubusercontent.com`、`release-assets.githubusercontent.com` |
| Maven / Gradle / Android | `repo.maven.apache.org`、`repo1.maven.org`、`services.gradle.org`、`downloads.gradle.org`、`dl.google.com` |
| Go / Rust | `proxy.golang.org`、`sum.golang.org`、`crates.io`、`index.crates.io`、`static.crates.io` |

其他域名仍校验证书，跳转到其他域名也重新判定。`huawei.com.example.org` 不匹配华为域名。
这些例外跳过证书校验，不代表已验证企业 CA 的身份。

包管理器有独立的 TLS 实现：npm 系列工具进程暂时对所有目标关闭证书校验；pip 信任表中列出的域名；
APT 通过当前进程的专用配置关闭传输证书校验。APT 软件包签名、文件哈希和固定下载校验值仍会检查。
程序不修改包管理器的全局配置文件。

设置 `PBT_TLS_EXCEPTIONS=off` 可关闭这项兼容；显式设置 `NODE_TLS_REJECT_UNAUTHORIZED=1` 也会保持严格校验。
需要改用自己的 CA 时，启动前设置 `NODE_EXTRA_CA_CERTS=/path/company-ca.pem`，并关闭上述例外。
