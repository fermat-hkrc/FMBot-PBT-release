# Enterprise certificates

[中文](network.zh.md) · [Installation](installation.md)

Enterprise certificate compatibility is enabled by default. No certificate configuration is needed for the destinations below.

| fmbot-pbt HTTPS destinations with a certificate exception | Hosts |
|---|---|
| Huawei | `huawei.com`, `*.huawei.com`, `repo.huaweicloud.com`, `mirrors.huaweicloud.com`, `ohpm.openharmony.cn` |
| npm / Node / PyPI | `registry.npmjs.org`, `nodejs.org`, `pypi.org`, `files.pythonhosted.org` |
| Debian / Ubuntu | `deb.debian.org`, `security.debian.org`, `archive.debian.org`, `archive.ubuntu.com`, `security.ubuntu.com`, `ports.ubuntu.com`, `old-releases.ubuntu.com` |
| GitHub | `github.com`, `api.github.com`, `codeload.github.com`, `raw.githubusercontent.com`, `objects.githubusercontent.com`, `release-assets.githubusercontent.com` |
| Maven / Gradle / Android | `repo.maven.apache.org`, `repo1.maven.org`, `services.gradle.org`, `downloads.gradle.org`, `dl.google.com` |
| Go / Rust | `proxy.golang.org`, `sum.golang.org`, `crates.io`, `index.crates.io`, `static.crates.io` |

Other destinations retain certificate verification, including redirects to other hosts. A lookalike such as
`huawei.com.example.org` does not match. These exceptions skip certificate verification; they do not authenticate the enterprise CA.

Package tools use separate TLS stacks. npm-family tool processes temporarily disable TLS verification for all destinations;
pip trusts the listed hosts. APT uses a process-local transport configuration with peer verification disabled.
APT archive signatures, package hashes, and pinned download checksums remain checked. No global package-manager files are changed.

Disable this compatibility behavior with `PBT_TLS_EXCEPTIONS=off`. An explicit
`NODE_TLS_REJECT_UNAUTHORIZED=1` also keeps strict verification. To trust your own CA instead, set
`NODE_EXTRA_CA_CERTS=/path/company-ca.pem` before startup and turn these exceptions off.
