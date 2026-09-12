---
title: 'NPM 爬蟲 SSL 驗證失敗排查記錄'
date: '2026-09-12T21:38:48+08:00'
draft: false
tags: [ "Python", "SSL" ]
categories: [ "GitHub Actions", "爬蟲", "AI寫的", "踩地雷" ]
slug: '/python-ssl-cert-issue-ai-written/'
---

## 說明

故宮（`npm.gov.tw`）的爬蟲在 GitHub Actions 上執行時，固定出現：

```bash
httpx2.ConnectError: [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get local issuer certificate
```

但同一份程式碼，在本機 Windows 執行卻完全正常，沒有任何錯誤。這篇記錄整理排查過程與最終解法。
 
----

## 觀察過程

### 用 curl 重現

在 GitHub Actions 裡加一個 debug step：

```bash
curl -v https://www.npm.gov.tw/
```

輸出顯示 TLS handshake 進行到 `Certificate (11)` 之後，client 端主動送出 `unknown CA` alert 中斷連線，並回報：

```bash
SSL certificate problem: unable to get local issuer certificate
```

確認問題出在憑證信任鏈，而不是連線被防火牆或 CDN 攔截。

### 用 openssl 檢查完整鏈路

```bash
openssl s_client -connect www.npm.gov.tw:443 -servername www.npm.gov.tw -showcerts
```

發現伺服器只送出了 **一張憑證**（Leaf Certificate），簽發者是 `TWCA Secure SSL Certification Authority`。伺服器
**沒有一併送出中繼憑證**，而 client 端（無論 curl 或 Python 的 `certifi`）都不會主動去網路上補齊缺少的中繼憑證。

### 為什麼本機沒事

比對本機執行的 log，發現本機用的 SSL context 其實是 `truststore` 套件，它會把驗證工作交給 **Windows CryptoAPI**。Windows
有自動抓取缺失中繼憑證的機制（透過憑證裡的 AIA 欄位），所以本機「看起來」完全正常，只是問題被 Windows 悄悄補掉了。GitHub
Actions 是乾淨的 Linux 環境，沒有這層自動修復，問題就直接暴露出來。
 
----

## 解法

下載 TWCA 對應的中繼憑證（`secure_sha2_2023G3.crt`），核對官方指紋與到期日後，轉成 PEM 格式存入 repo，並在建立 `SSLContext`
時手動補上：

```python
import ssl
import certifi

ssl_context = ssl.create_default_context(cafile=certifi.where())
ssl_context.load_verify_locations(cafile="twca_intermediate.pem")
```

用 openssl 驗證完整鏈路，`Verify return code: 0 (ok)` 代表修復成功。

考量到中繼憑證本身也有到期日（此版本到 2030 年），額外加了一支每月執行的 GitHub Action，定期檢查憑證剩餘天數，剩 90 天內就自動開一個
issue 提醒，並且避免同一份憑證重複發出提醒，交由人工確認、下載新版、核對指紋後手動更新，不做成全自動抓取更新，降低誤信偽造憑證的風險。
