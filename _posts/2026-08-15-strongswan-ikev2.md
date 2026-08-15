---
layout: post
title: "WindowsとStrongSwanでIKEv2接続"
date: 2026-08-14 22:00:00 +0900
last_modified_at: 2026-08-11
description: いままでL2TP/IPSecで運用していたVPN接続をIKEv2の証明書接続に変更しました
img: 2026/2026-08-15-1.jpg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [debian,security,network]
categories: 
  - debian
---

L2TP/IPSecが古い接続方式だという記事をみかけたので、現在リモート接続用に運用しているStrongSwanをL2TP/IPSecから証明書を使ったIKEv2接続に変更することにしました。

L2TP/IPSecは、IKEv1のみで、v2では利用できません。そのあたりが「古い」と言われるゆえんだと思います。

L2TP/IPSecではは下流に`pppd`や`xl2tpd`が必要になったり、1701番ポートのファイアウォール設定も必要になったりといろいろ手間だったのですが、IKEv2にすればそのあたりの設定は不要になるので、管理が楽になります。

基本的にIPv4環境で設定例を書いていきますが、IPv6の場合もアドレス表記だけ変えれば接続可能だと思います(環境がないので試せてはいません)。

**目次**
* TOC
{:toc}

## strongSwanのインストール

strongSwanは単一というより、複数のデーモン・コマンド・プラグインから構成されるIPsec/IKEソフトウェア群と言った方が近いと思います。

過去バージョンの互換性維持等もあり、初見だとわかりにくい構成になっています。

| 名前 | 機能 |
| :--- | :---|
| charon | 中心となるデーモン。IKEv1/IKEv2、認証、SA確立などを担当|
| VICI | 管理用API (Versatile IKE Configuration Interface)|
| swanctl | charon を操作するための インターフェース |
| 他　| 各種プラグイン　|


コアパッケージは`charon`と呼ばれます。ユーザーは`swanctl`のコマンドで操作をします。
`swanctl`は`VICI`を利用して、`charon`に設定を伝えます。
最終的に、`charon`がカーネルに指示を出します。

過去バージョンでは`starter`が`Stroke`を介して、`charon`に指示をするという構成になっていました。
過渡期のバージョンではその両方が利用が可能になっており、それぞれ独立`starter`で開始した`charon`を`VICI`で操作するということも可能でした。`ipsec.conf`を利用していたら、`starter`構成です。

また、以前は`starter`がサービスに登録され、それが`charon`を起動していましたが、新バージョンでは、`charon-systemd`という形で、`systemd`が`charon`を直接管理します(起動後は`swanctl`経由で初期設定されます)。新しいバージョンのサービス名には`strongswan.service`がつけられています。

なので、新旧どちらが使われているかは、`strongswan-starter`サービスがいるか、`strongswan`サービス(中身はcharn-systemd)がいるかで判断できます。


Debian13で`strongswan`をAPT経由でインストールすると、`libstrongswan-standard-plugins`だけ入って、`libstrongswan-extra-plugins`が入らないようなのでそれを指定してインストールします。

## サーバーの設定

### プラグインのロード
charonに対する、各種プラグインの設定ファイルは、`/etc/strongswan.d/charon/`に入っています。今回は特にさわる必要はなさそうですが、たとえばagent.confは次のようになっていて、charon起動時にロードされるようになっています。
```
# agent.conf
agent {
    load = yes

}
```
現在どのようなプラグインが読み込まれているかは、状態一覧を出力する`swanctl --stat`コマンドから確認できます。

### 接続設定

接続設定は`/etc/swanctl/conf.d/`に入れます。`swanctl`は`charon`起動時に`swanctl --load-all`コマンドを実行しますが、それが実行されるのこのディレクトリ内のすべての`.conf`ファイルが設定ファイルとしてロードされます。

ここでは`ikev2-cert.conf`と名付けました

```
# ikev2-cert.conf

# 固定名 connections
connections {
    # 設定名(任意)
    ikev2cert {
        # IKEのバージョン
        version = 2
        # Phase1
        # 暗号化方式 (- Integrity) (- PRF ) (- DHグループ)
        # AES-GCMの時は暗号化とIntegrityが一体なので、Integrity部分は書かない
        # PRFが省略された時は、自動でセットされる
        # 設定によってはDHの省略も可能
        # 複数の設定は,を区切りで優先順に書く
        # defaultは自動設定だが何が採用されるか分かりにくいので明示した方がいい
        proposals = aes256-sha256-modp1024, default
        
        # サーバー側のNICについているアドレス入るNICを固定したい場合に利用
        # %anyで指定なし
        local_addrs = %any
        
        # 相手側のアドレス、今回の用途だと不定にする 
        remote_addrs= %any

        # IPアドレスプール名
        pools = vpn-pool
        
        # サーバー側がクライアントに認証してもらう際の設定
        # IKEv2では相互認証が必須
        # 固定名local
        local {
            # 証明書認証
            auth = pubkey
            # サーバー証明書
            certs = server.crt
            # 証明書のCommonName や SANと同じにしておく
            id = server.jp
        }
        
        # 接続側を認証する方法
        # 固定名 remote
        remote {
            # クライアント認証方式
            auth = pubkey
            # 失効リストを指定(任意) 
            # strict(失効リストが確認できなとエラー 有効期限の設定に注意) 
            # relaxed(失効リストがあれば採用)
            revocation = relaxed
        }
        
        # CHILD SAの設定
        # 認証後、どの通信をIPsecに流すか
        # 固定名 children
        # local_tsとremote_tsはトラフィックセレクタ選定の基準で
        # どの宛先のパケットをIPSecに乗せるかを指定する
        children {
            # 任意名
            net-remote-vpn {
                # すべての通信をIPSecに向ける場合
                #local_ts = 0.0.0.0/0
                # 特定のアドレスのみをIPSecに向ける
                local_ts = 192.168.1.0/24, 192.168.100.0/24
                
                # 相手先は割り当てたアドレス
                remote_ts = dynamic
                
                # Phase2ではPRFの指定はない他は、Phase1のproposalsと同様
                # 暗号化方式 (- Integrity) (- DHグループ)
                esp_proposals = aes256-sha1
            }
        }
        # 通常証明書は要求に応じて送られる
        # 一部のクライアントは証明書送信要求をしてこないことがあるので
        # そのような場合にも送る(always)にすると互換性が高まる
        send_cert = always
    }
}

# 固定名 pools
pools {
    # 任意名(IPアドレスプール名)
    vpn-pool {
        # アドレスプール（ネットワーク指定時)
        # addrs = 10.10.10.0/24
        # レンジ指定
        addrs = 10.8.202.100-10.8.202.199

        # DNSを通知することも可能
        # dns = 8.8.8.8
    }
}

# 固定名 secrets
secrets {

    # 任意名(鍵設定名)
    private-key {
        # サーバー秘密鍵
        # ファイルとして読み込ませることで、あとは自動判別される
        # 証明書との組み合わせもユーザー側での指定は不要
        file = server.key
    }
    # 注意：PSKを設定する時は鍵名はike-プレフィックスが必要です
}
```

### ファイアウォールの設定

ファイアウォールを動かしている場合は、NAT越しの時はudpの500(IPSec)と、4500(NAT-T)を許可します。

直接接続の場合は、IPプロトコルの50番(ESP)とudpの500を通します。

nftablesの場合は次のような感じになります。
sshなど他の許可は書いてありませんので自身の環境に合わせて下さい。

```
table inet filter {
        chain input {
                type filter hook input priority filter; policy drop;
                ip protocol icmp accept
                iifname "lo" accept
                
                # 確立済の接続を許可
                ct state established,related accept
                
                # IPv4 NAT環境
                udp dport { 500, 4500 } ct state new accept
                
                # IPv4 直接接続の場合はESPを通す
                # udp dport 500 ct state new accept
                # ip protocol esp ct state new accept
                
                # IPv6 直接接続 
                # udp dport 500 ct state new accept
                # ESP指定が拡張ヘッダにいる可能性があるのでnexthdr指定ではない
                # meta l4proto esp accept
                
        }

        chain forward {
                type filter hook forward priority filter; policy drop;
                ip saddr 172.16.0.0/16 ip daddr 172.16.0.0/16 accept
        }

        chain output {
                type filter hook output priority filter; policy accept;
        }
}
```

### 証明書の用意と配置

証明書はつぎのような構成で作成します。VPN用のCAがstrongSwanサーバーと、Windowsクライアント用の証明書を発行します。

strongSwanは基本的に自身の証明書を発行したCAが発行した証明書をもつクライアントに対して接続を許可します。

鍵漏洩時対策などで、特定の証明書だけ利用不可にしたい場合は、失効リスト(CRL)を運用します。

CA
├── サーバー(strongSwan)
└── クライアント(Windows)

それぞれ秘密鍵と証明書が必要です。クライアント用の証明書と秘密鍵は共用も可能ですが、端末毎に分けるのが理想です。
そうしておかないとひとつの秘密鍵が漏れた際に全台を証明書を入れ替える必要がでてきます。

証明書と秘密鍵は`opnessl`で作ることができますが、これらの説明は長くなりますので別記事の[OpenSSLでプライベートCAを運用する](https://sugakenn.github.io/debian/make-cert.html)を参考にしてください。

strongSwanではわりと厳格な運用を求められれますので、署名時にバージョン3拡張が必要です。
具体出来には、**CA証明書のbasicConstraints**や、**CRL署名のauthorityKeyIdentifier**などがあります。
**サーバー証明書にはsubjectAltName(SAN)**が必要です。SANかCommonNameどちらが使われるかは実装に依存するところですが、どちらが使われてもいいように同じにしておくと良いと思います。
**IPアドレスでstrongSwanに接続する場合はIPで、FQDNで接続する場合はFQDNと、接続方法に合わせた名前にする必要もあります。**

CA証明書は`/etc/swanctl/x509ca`、サーバーの秘密鍵は`/etc/swanctl/private`、公開鍵は`/etc/swanctl/x509`へ配置します。

サーバーの証明書と秘密鍵は先の設定ファイル上に記述がありますが、それはファイル名だけでパスを含めることはできません。
またCA証明書や失効リスト(CRL)はディレクトリ内にあるものが必要に応じて自動で使われます。

## クライアントの設定(Windows11)

Windowsから接続するには、まずCAの証明書と自身の証明書をWindowsへインストールしておきます。

PFXでバンドルしておけばそれぞれ自動ではいりますが、CA証明書はルートCAなら「信頼されたルート証明機関」へ、中間CAなら「中間証明機関」へ入れます。

中間証明書だった場合は、それを署名したルートCAの証明書が別途「信頼されたルート証明機関」に入っていなければいけません。

クライアント用の秘密鍵と証明書は「個人」に入れます。

**これらの証明書は、「ユーザーの証明書」ではなく「コンピューターの証明書」の領域へ保存しないと意図したように機能しません。**

証明書のインストールが終わったら、[設定]→[ネットワークとインターネット]→[VPN]→[VPNの追加]と進みます。

![VPNの追加]({{ '/assets/img/2026/2026-08-15-2.jpg' | relative_url }})

[接続名]には任意の名前を付けます。[サーバー名またはアドレス]部分は、接続先をIPかFQDNで設定しますが、strongSwanの証明書にかかれた名前にしておく必要があります。

[VPNの種類]は`IKEv2`を、[サインイン情報の種類]には`証明書`を選択して一度保存します。

作成したVPN名の横にある[▼]を押して[詳細オプション][その他のVPNプロパティ]に進みます。

![その他のプロパティ]({{ '/assets/img/2026/2026-08-15-3.jpg' | relative_url }})


[セキュリティ]タブの下段にある、[認証]欄のラジオボタンを`コンピューターの証明書を使う`にして[OK]を押します。

これで設定は完了です。


## 起動確認と切断

`systemctl restart strongswan`としてサービス全体を再起動させれば変更した設定は反映されますが、既存の接続はそのままで新しい接続からは新設定を採用する場合には`swantcl --load-all`とします。

plugin 'agent': failed to load - agent_plugin_create returned NULL
現在確立されている接続は`swanctl --list-sas`で確認できます。IKE SAや、CHILD SA などで出てくる`SA`はSecurity Associationの略で、このコマンドのsasはその複数形です。

```
# --list-sasの省略形-l
swanctl -l

vpn01: #12, ESTABLISHED, IKEv2, ...
  local  'server.example.com' @ 203.0.113.10[4500]
  remote 'client01' @ 198.51.100.20[4500]
  ...
  child01: #8, reqid 1, INSTALLED, TUNNEL, ...
```

ここで出てきたPhase1(IKE SA)のSA `vpn01`を切断するには

```
swanctl --terminate --ike vpn01
```

Phase2(CHILD SA)のSA `child01`を切断するには
```
swanctl --terminate --child child01
```
とします。IKEを切断した場合はその配下にあるCHILDもすべて切断されます。

ログは、`journalctl -u strongswan`から見られますので、接続ができないようならそのログを参考にします。

## agentプラグインのエラー
筆者の環境ではデフォルトで次のようなエラーがでました。

```
agent plugin requires CAP_SETUID/CAP_SETGID capability
plugin 'agent': failed to load - agent_plugin_create returned NULL
```

先ほどのcharonのプラグインの設定でloadしないようにしても、エラーがでるようです。
（たぶんswanctlがagentプラグインを読み込む)

[strongSwanのgitレポジトリ](https://github.com/strongswan/strongswan/discussions/2971)にも掲載されていて、DebianAPTパッケージのunstable 6.0.5では解消されているそうです。

capabilityが不足しているということなので、権限を付けてしまえばいいとも思いましたが、全体としてどのように作用するかわからない部分もあるので更新されるまで待とうと思います。
