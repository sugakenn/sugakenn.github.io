---
layout: post
title: "プライベートCAでVBAプロジェクトのコード署名をする"
date: 2026-07-21 17:00:00 +0900
description: MS WORDのVBAにプライベートCAで作成した鍵をつかって署名をしてみました
img: 2026/2026-07-21-1.gif # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [pc_tips]
categories: 
  - pc_tips
---


AIを利用したマルウェアやソフトウェア改ざんなどの脅威が増える中、OfficeのVBAマクロにも信頼性を保証するためのコード署名が重要になっています。

今回は、**プライベートCA（Private CA）** が既に構築されている前提で、VBAプロジェクトへコード署名を行うための証明書を発行する手順をまとめます。

例示にはWORD環境を使います。
---

# コード署名用の秘密鍵を作成する

まずはコード署名専用の秘密鍵を作成します。

> **ポイント**
>
> CAの秘密鍵を直接コード署名に使用してはいけません。
> コード署名専用の秘密鍵を新たに作成します。

```bash
openssl genpkey \
    -algorithm RSA \
    -pkeyopt rsa_keygen_bits:3072 \
    -out codesign.key
```

---

# CSR（証明書署名要求）を作成する

秘密鍵からCSRを作成します。

```bash
openssl req \
    -new \
    -key codesign.key \
    -out codesign.csr \
    -subj "/C=国/ST=都道府県/L=市町村/O=組織/OU=部署/CN=Common Name"
```

---

# X.509 Version3 の設定ファイルを作成する

コード署名用証明書として利用するため、Version3拡張を設定します。

ここでは `v3sign-code` というファイル名にしました。

特に

```text
extendedKeyUsage = codeSigning
```

がコード署名証明書として最も重要な設定になります。

```ini
[codesign_cert]
basicConstraints = critical,CA:FALSE
keyUsage = critical,digitalSignature
extendedKeyUsage = codeSigning
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
```

---

# プライベートCAで署名する

作成したCSRを、プライベートCAで署名します。

```bash
openssl x509 \
    -req \
    -in codesign.csr \
    -CA rootca.crt \
    -CAkey rootca.key \
    -CAcreateserial \
    -out codesign.crt \
    -days 3650 \
    -sha256 \
    -extfile v3sign-code \
    -extensions codesign_cert
```

---

# PFXファイルを作成する

Windowsでは秘密鍵と証明書をまとめた **PKCS#12（PFX）形式** を使用します。

```bash
openssl pkcs12 -export \
    -inkey codesign.key \
    -in codesign.crt \
    -out codesign.pfx \
    -name "Code Signing"
```

実行時にはPFXのパスワードを聞かれます。

> **注意**
>
> PFXには秘密鍵が含まれています。
> 第三者に利用されないよう、十分に強固なパスワードを設定してください。

---

# Windowsへ証明書をインポートする

作成した

```
codesign.pfx
```

をWindowsへコピーし、ダブルクリックしてインポートします。

通常は

```
現在のユーザー
 └ 個人
```

へインポートします。

これでOfficeからコード署名証明書として利用できるようになります。

---

# VBAプロジェクトへ署名する

VBAエディターを開き、

```
開発
    ↓
Visual Basic
```

または

```
Alt + F11
```

を押します。

続いて

```
ツール
    ↓
デジタル署名
```

を開きます。

「**選択**」を押すと、Windowsへインポート済みの証明書一覧が表示されます。

コード署名証明書を選択して保存すると、VBAプロジェクトへデジタル署名が行われます。

---

# コードを修正した場合

一度証明書を設定しておくと、**秘密鍵が利用可能な環境では**VBAプロジェクトを保存した際に同じ証明書で再署名されます。

ただし、

- 証明書が削除されている
- 秘密鍵が存在しない
- PFXが利用できない

などの場合は再署名できません。

また、証明書を変更する場合は以前の署名を削除するか確認されます。

---

# 文書の署名とは別

今回の例ではWord文書を使用していますが、署名対象はWord文書ではなくVBAプロジェクトです。
ドキュメントが改変されていないかという署名はまた別となります。
そのため、文書本文の編集やファイル名変更などでは、署名は更新されません。

ちなみに、ドキュメントを署名したければ、先ほどの X.509 Version3 の設定ファイル内で、extendedKeyUsage での指定を documentSigning に変えて鍵を作り、Wordでは[ファイル]→[情報]→[文書の保護]→[デジタル署名の追加]と進み、署名をする流れになります。

となります。

---

# 注意事項

プライベートCAが各PCで信頼されている環境であれば、署名済みVBAマクロとして利用できます。

- 「すべてのVBAマクロをブロックする」

というOfficeのセキュリティポリシーが設定されていた場合は、コード署名だけでは回避できません。

---

# まとめ

署名を行う事で、Office VBAマクロの信頼性を高めることができます。
ただし、ユーザーが一度編集してしまえば、ブロックから外せますし署名がなくてもコード自体は実行可能なので、セキュリティ対策としての高価は限定的です。

PCを分かっている利用者なら署名がはずれていたら実行しないという選択肢もでてくるのですが、今だと普通の人ならAIにブロック解除の方法と、マクロのセキュリティレベル変更を聞いたりして中身の危険性が考慮されないまま実行されてしまう可能性もあると思います。
ただ、改変があったかどうかがわかったり、多少ではありますがWindowsのブロックの緩和にもなるので、今後はVBAには署名をつけておこうと思います。



