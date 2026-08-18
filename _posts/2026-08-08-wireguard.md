---
layout: post
title: "WireGuardを使ってみる"
date: 2026-08-08 17:00:00 +0900
last_modified_at: 2026-08-08
description: OpenVpnやStrongSwanを使ってきた筆者がWireGuardの導入のためにその基本を学びました
img: 2026/2026-08-08-1.png # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [debian,routing]
categories: 
  - debian
---


VPNを作成するアプリには[OpenVPN](https://openvpn.net/)や[StrongSwan](https://strongswan.org/)などがありますが、設定が複雑になりがちです。

[WireGuard](https://www.wireguard.com/)はその点が考慮されたアプリです。今回はその導入の検討のために使い方を勉強しました。

**目次**
* TOC
{:toc}

## 構成
今回は中央サーバー型(スター型)接続にします。

```
             端末A
               │
               │
端末B ─── 中央サーバー ─── 端末C
               │
               │
             端末D
```

ピアTOピアでも設定はできますが、端末数の増減時はそれぞれの端末の接続設定を変える必要があるので、中央サーバーがあるスター型の設計すると管理が楽です。

また、固定されたIPアドレスがない場合はDDNSに頼ることになりますが、ピアTOピアだとそれぞれの端末がDDNSでアドレスを特定させないといけないのに対し、中央サーバーがあれば中央サーバー以外はDDNSに頼る必要はなくなります。

更に中央サーバーに接続されているもの同士なら、IPv6かIPv4の外側の接続方式を無視して内側のIP体系に基づいて接続することができます。

端末側からの接続要求に基づいた接続になるので、IPoEのMAP-EやDS-Liteで起きる外側から着信できない問題も考える必要がありません。

中央サーバーがDebian、端末がWindowsという構成で、VVNCやRDPで接続するというケースを考えてみました。

## 中央サーバー

中央サーバーにはDebian13を使います。

### aptでwireguradをインストールします。
```
apt install wiregurad
```

### 鍵のセットを作成します
root権限で作成します。
先に`umask 077`にしておきます。一瞬でも他者に読み込まれる可能性がある権限でファイルを作ると警告がでます。
間違えて作成してしまったら、一度ファイルを削除しないと、アクセス権が継承されますので注意して下さい。
```
umak 077
wg genkey > wg-root-key
```
公開鍵は秘密鍵から作成します。直接ファイルを使うことはないのですが、一応公開鍵は他者からの読み込みは許可します。
```
wg pubkey < wg-root-key > wg-root-pub
chmod 644 wg-root-pub
```

### wgコマンド用の設定ファイルを作成します
INIファイルようなレイアウトで、サーバー側の設定を[Interface]、接続先の設定を[Peer]に書きます。[Peer]セクションは接続する端末の数だけ存在します。
[Interface]側にはPrivateKey、[Peer]側にはPublicKeyを設定しますが、どちらも生の値(先ほど出力したファイルの中身)を使い、ファイルとしては指定できません。
そのため、設定ファイルの読み込み権限も600としておく必要があります。

ここでは`wg-test.conf`へ保存しました。
```
[Interface]
PrivateKey = yAnz5TF+lXXJte14tji3zlMNq+hd2rYUIgJBgB3fBmk=
ListenPort = 51820

[Peer]
PublicKey = xTIBA5rboUvnH4htodjb6e697QjLERt1NAB4mZqp8Dg=
Endpoint = 192.95.5.67:51820
AllowedIPs = 192.168.2.2/32

# Ipv6アドレス
[Peer]
PublicKey = TrMvSoP4jYQlY6RIzBgbssQqY3vxI2Pi+y71lOWWXX0=
Endpoint = [2607:5300:60:6b0::c05f:543]:51820
AllowedIPs = 192.168.2.3/32

# FDQN
[Peer]
PublicKey = gN65BkIKy1eCE9pP1wdc8ROUtkHLF2PfAqYdyYBz6EA=
Endpoint = test.wireguard.com:51820
AllowedIPs = 192.168.2.4/32
PersistentKeepalive = 25

```

#### Inteface ListenPortには
ListenPortにWireGurdの接続を待ち受けるポート(UDP)を指定します。慣例として`51820`が使われることが多いようです。
#### Peer Endpointには、
Endpoint通信先の宛先とポートを指定します。この値は相手側からの正常な接続により更新されます。そのため、初回子側から中央サーバーに確実に送信があるのならEndpointの記述を省略できます。
#### Peer PersistentKeepalive
PersistentKeepaliveには1～65535の値で、Keep Aliveの送信間隔を指定します。記述しない場合はKeep Aliveを送信しません。
多くの場合で不要だということですが、NAT環境にある場合は一定期間接続がないと中央側から子への新規発信が繋がらなくなります。
その場合はNATマッピングのキャッシュ時間に合わせてこの値を25ぐらいに設定するそうです。


### wiregurdインターフェースを作成します

#### Debian上にwireguardインタフェースを作成します
```
ip link add wg0 type wireguard
```
#### インターフェースに内側のIPとネットワークを設定します

```
ip address add dev wg0 192.168.2.1/24
```
#### 先ほど作成した設定ファイルを読み込みwg0へセットします。
ファイルを読み込むほか[wgコマンド](https://git.zx2c4.com/wireguard-tools/about/src/man/wg.8)で個々に設定することも可能です。
```
wg setconf wg0 test.conf
```
#### インターフェースをUPします
```
ip link set up dev wg0
```

### wg-quick
[wg-quick](https://git.zx2c4.com/wireguard-tools/about/src/man/wg-quick.8)を使うと、先ほどのまでの設定をもう少し簡易にすることができます。

設定ファイルの仕様がwg用の物とは少し変わります。
設定ファイル名はwg0(wiregurdインタフェース名)+.confで設定して、`/etc/wiregurad/`へ保存しておくと後で便利に使えます。
Interfaceに`ADDRESS`として、自アドレスを設定します。他、DNSを指定する`DNS`、MTUを指定する`MTU`や、ルーティングテーブルを指定する`Table`、スクリプトを登録できる`PreUp`、`PostUp`、`PreDown`、`PostDown`などが選択できます。
さらに`SaveConfig`の値をTrueにすると、インターフェースシャットダウン時に現在の設定を設定ファイルに書き出します。
```
[Interface]
Address = 10.200.100.8/24
DNS = 10.200.100.1
PrivateKey = oK56DE9Ue9zK76rAc8pBl6opph+1v36lm7cXXsQKrQM=

[Peer]
PublicKey = GtL7fZc/bLnqZldpVofMCD6hDjrK28SsdLxevJ+qtKU=
PresharedKey = /UwcSPg38hW/D9Y3tcS1FOV0K1wuURMbS0sesJEP5ak=
AllowedIPs = 10.8.0.2/32, 172.16.10.0/24
Endpoint = demo.wireguard.com:51820
```

さらに、wg-quickが便利なところは、AllowIPsで設定されたアドレスがWireGurdとしてあらたに作成したネットワーク帯でなくても、自動でルーティングを設定してくれることです。
ただしこれは`Table`の値に`off`を入れると作成されません。
ちなみに`Table`のデフォルト値は`auto`で、これはデフォルトのルーティングテーブルを操作する設定となります。

これは、wgコマンドの場合もそうですが、FQDNを使ったEndpoint設定を入れておく場合は、Peerのうち一つでも解決に失敗すると全体が失敗しますので注意が必要です。

このように設定したファイルを`wg-quick up`コマンドで指定すると、先ほどのDebianのインターフェースの設定も含めてまとめてやってくれます。

また、`/etc/wiregurd/wg0.conf`として設定しておけば、`wg-quick up wg0`として設定ファイルのパス指定も省略可能です。

### 転送と、ファイアウォールの設定
インターフェース間のパケット転送はデフォルトで許可されていませんので、`/etc/sysctl.d/ipfowardv4.conf`に次のように書いて保存します。

```
net.ipv4.ip_forward=1
```
ファイアウオールの設定はwg-quickコマンドでも自動的には設定されません。
nftなら、インプットチェーンに
```
udp dport 51820 accept
```
フォワードチェーンに
```
iifname "wg0" oifname "wg0" accept
```
という設定を入れる必要があります。不正なアクセスはファイアウォールでは通しておいて、wireguardで弾くというイメージです。

### 永続化

永続化は、wiregurdをインストールした際にできるサービスファイルで、管理します。

ファイルは、`/usr/lib/systemd/system/wg-quick@.service`に存在し、インタフェースを可変に設定できるのでここでは
```
 systemctl enable wg-quick@wg0
```
として有効化します。

### 状態の確認
現在の状態は単に`wg`とすることで表示されます。

## 子側の設定
子側は、ウインドウズ用のインストーラーバイナリをダウンロードしてインストールします。

`トンネルの追加`の右側の矢印ボタンから`空のトンネルの追加`を選ぶと自動的にプライベートキーとパブリックキーを用意してくれます。

ルーティングも自動です。
```
[Interface]
PrivateKey = oK56DE9Ue9zK76rAc8pBl6opph+1v36lm7cXXsQKrQM=
ListenPort = 51820
Address = 192.168.2.2/32

[Peer]
PublicKey = dhPoQTwcysKe3JTUJfu9EaVCcVRbN5757FH+8CD7zG8=
AllowedIPs = 192.168.2.0/24
Endpoint = hovpn.komart.jp:51820
PersistentKeepalive = 25
```
子側の場合51820で待ち受けても、実際のポートは`NAT`で変わってしまうのであまり意味がないかもしれません。

## 
しばらく使ってみた結果、NATの内側にある端末は、PersistentKeepaliveによるKEEP ALIVEを中央側に送信しないと、中央側から接続できなくなるようでした。
非常用に回線を用意しておきたい場合、25秒に1回のKEEP ALIVEは過剰な気がしました。StrongSwanとかでIPSec回線をつなげる場合もそのような頻度で回線を維持するので、おかしくはないのでしょうが。

そこで私が考えたのは、通常は子側のKEEP ALIVEを3600(1時間)にしておき、通常時は中央側には設定しないでおく方法です。

使いたくなったら中央側で対象のインターフェスに対してPersistentKeepaliveを25に設定することです。

そうしておけば、1時間に1回のKEEP ALIVEを中央が捕まえた後からは常時接続の状態になります。

その際はコマンドで、中央側のKEEP ALIVEの設定を変えられます。

```
wg set [インターフェース(wg0 等)] peer [対象の公開鍵(wg showで表示される peer名)] persistent-keepalive 25
```

接続したい要件が終わったらまたPersistentKeepaliveを0に戻します。注意が必要なのはデフォルトでは設定ファイルは書き換わらないので、中央側のサーバーが再起動してしまうと再度KEEP ALIVEを送らなくなってしまいます。
