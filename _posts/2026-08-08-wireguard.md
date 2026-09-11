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

中央サーバーがDebian、端末がWindowsという構成で、VNCやRDPで接続するというケースを考えてみました。

## 中央サーバーの設定

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

#### Inteface - ListenPortには
ListenPortにWireGurdの接続を待ち受けるポート(UDP)を指定します。慣例として`51820`が使われることが多いようです。

#### Peer - AllowedIPsには、
AllowedIPsには通信を受け付けるIPアドレスの範囲を指定します。ここで指定したアドレス(範囲)外からのデータは受け付けません。,で区切って複数指定もできます。

また、送信時もAllowedIPsの範囲と照合させることで利用するPeerを決定します。

さらに、ここで設定されている範囲については自動でルーティングが設定されます。

#### Peer - Endpointには、
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

## wg-quick
[wg-quick](https://git.zx2c4.com/wireguard-tools/about/src/man/wg-quick.8)を使うと、先ほどの中央サーバーの設定をもう少し簡易にすることができます。

おそらく実運用はこちらに頼ることになると思います。

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

さらに、wg-quickが便利なところは、`AllowIPs`で設定されたアドレスが`WireGurd`用に作成したネットワーク帯でなくても、自動でルーティングを設定してくれることです。
ただしこれはInterfaceセクションの`Table`の値に`off`を入れると作成されません。
ちなみに`Table`のデフォルト値は`auto`で、これはデフォルトのルーティングテーブルを操作する設定となります。テーブルを別に指定したい場合はテーブル番号を入れます。

wgコマンドの場合もそうですが、FQDNを使ったEndpoint設定を入れておく場合は、Peerのうち一つでも解決に失敗すると全体が失敗しますので注意が必要です。

中央サーバーの場合は、Peerのエンドポイントを書かずに相手からの接続を待った方が安定すると思います。

このように設定したファイルを`wg-quick up`コマンドで指定すると、先ほどのDebianのインターフェースの設定も含めてまとめてやってくれます。

また、`/etc/wiregurd/wg0.conf`として設定しておけば、`wg-quick up wg0`として設定ファイルのパス指定も省略可能です。

### 転送と、ファイアウォールの設定

ファイアウオールの設定はwg-quickコマンドでも自動的には設定されません。
nftなら、インプットチェーンに
```
udp dport 51820 accept

iifname "wg0" tcp dport 22

```
と、記述して各接続のパケットを受け入れるようにしておきます。それと、中央サーバーに対しての接続許可があれば追加します。

ここでは、ssh(TCP 22)を許可する設定にしてあります。`iiname "wg0"`の`wg0`の部分は、設定したネットワークインタフェース名をいれますが、これによってwireguard経由の接続だけを対象とすることができます。

wireguard同士の接続を許可したい場合は、フォワードチェーンに次のように書きます。
```
iifname "wg0" oifname "wg0" accept
```
この時、通常インターフェース間のパケット転送はデフォルトで許可されていませんので、それを許可するために`/etc/sysctl.d/ipfowardv4.conf`に次のように書いて保存します。

```
net.ipv4.ip_forward=1
```


ここで設定した51820は一般的なwireguardのポートで、ここにスキャンがくることも考えられますが、wireguardは正規の通信以外に関してはレスポンスをまったく返さないので、攻撃者へヒントを与えづらくなっています。

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
子側の場合51820で待ち受けるのは、ポートフォワードを使って目的のPCへ着信させるた場合には必要ですが、使わない場合はNAT経由で秘匿されてしまうので設定なし(ランダム)でもいいかもしれません。

## バックアップ回線で使う場合の方法

しばらく使ってみた結果、NATの内側にある端末は、Persistent Keepaliveを設定しないと、中央側から接続できなくなるようでした。

ただ非常用に回線を用意しておきたい場合においては、NATテーブルを維持するために推奨される25秒に1回のKEEP ALIVEは、過剰な気がしました。StrongSwanとかでIPSec回線をつなげる場合もそのような頻度で回線を維持するので、おかしい設定ではないのですが。

しかも25秒にKEEP ALIVEを設定したとしても、子側がNATの内側にいる時は通信できなくなることがあります。

最終通信時のendPointに通信をするWireGuardの仕様だと、中央サーバーのIPアドレスの変更時より通信相手を見失います。

そこで考えたのは、子側から中央側へ1時間に1回Pingを送信する方法です。

そして、Pingを送信する前に、endPointのIPアドレスや、ポートの設定等が動的に変わっていた場合に備えて、FQDNを指定するデフォルト値に戻します。

これは管理者権限で実行する必要があるのでWindowsだと設定が面倒ですが、実行時に一旦DNSが解決されるので、アドレスの変更にも対応可能です。

```
wg set [インターフェース(wg0 等)] peer [対象の公開鍵(wg showで表示される peer名)] endpoint dns.fqdn.jp:51820
```


この状態にしておいて回線を使いたくなったら中央側で対象のインターフェスに対してPersistent Keepaliveを25に設定します。

そうすれば1時間に1回のPingを中央サーバーがが捕まえた後からは常時接続の状態になります。

その際はコマンドで、中央側のKEEP ALIVEの設定を変えられます。

```
wg set [インターフェース(wg0 等)] peer [対象の公開鍵(wg showで表示される peer名)] persistent-keepalive 25
```

接続したい要件が終わったらまたPersistentKeepaliveを0に戻します。注意が必要なのはデフォルトでは設定ファイルは書き換わらないので、中央側のサーバーが再起動してしまうと再度KEEP ALIVEを送らなくなってしまいます。

Pingで接続を維持するためのPowersShellのスクリプトは次のような感じになると思います。Debianならrootのcronにいれればよく簡単なのですが、Windowsでは管理者権限でのタスクの実行は若干複雑です。

ここではc:\ProgramDataにファイルを配置する前提ですが、こうしておけばユーザー権限でps1ファイルを編集されることはなくなると思います。
```
# WireGuard用の定時更新用 .ps1ファイル
# 次のようにしてWindowsタスクへ登録
# [全般]
# タスク実行時に使うユーザー 「SYSTEM」
# [トリガー]
# タスクの開始「任意のユーザーのログイン時」
# 遅延時間 「1分」
# 繰り返し時間 「1時間/無期限」
# [操作]
# プログラムの開始
# プログラム powershell.exe / 引き数 このファイルへのフルパス / 開始 このファイルがあるフォルダ 
# [条件]
#「AC電源でなくても実行」「任意の接続が使用可能な時に実行」
# 筆者の環境では任意の接続の代わりにwg0を指定しても動きました
# このファイルはユーザー権限では書き込めないようにしておく必要がある
# ProgramDataの任意のフォルダを作成してそこへ配置、
# 実行プログラムpowershell.exe 引き数にファイルフルパス、実行フォルダをこのファイルの親フォルダにする

# 設定 -------------------------------------------------
$wg = "C:\Program Files\WireGuard\wg.exe"
$pubkey = "中央サーバーのPUBLIC_KEY"
$net = "wg0(トンネル名)"
$endpoint = "endpoint.jp:51820(中央サーバー)"
$logFile = "C:\ProgramData\WGBeacon\wg-beacon.log(ログファイル)"
$pingIp = "192.168.255.1(中央サーバーのWGアドレス)"
# -------------------------------------------------------

$msg = "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') INFO WireGuard endpoint update start"
Add-Content -Path $logFile -Value $msg

& $wg set $net peer $pubkey endpoint $endpoint


if ($LASTEXITCODE -ne 0) {
    $msg = "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') ERROR WireGuard endpoint update fail"
    Add-Content -Path $logFile -Value $msg
    exit 1
}

$pingResult = ping $pingIp 2>&1
Add-Content -Path $logFile -Value $pingResult
```

## 端末管理

端末が増えてくると管理が少し面倒なので、簡易的なWebUIを用意してみました。お好きにカスタマイズしてください。

こちらの構成例は数種のファイルに及ぶので、[レポジトリ](https://github.com/sugakenn/blog_box/tree/main/wg)で、説明・公開しています。

## WindowsのWireGuard端末をルーター化

子側のWindows端末をルーター化してネットワーク接続接続する方法は次のようになります。

まず、中央側の設定に端末のホストアドレスに加えて、接続したいネットワークを,で区切って指定します。

```
#中央側
[Interface]
Address = 192.168.255.1/24
PrivateKey = ...

# 拠点A(端末のNIC側にある192.168.1.0/24へ接続)
[Peer]
PublicKey = ...
AllowedIPs = 192.168.255.2/32, 192.168.1.0/24


# 拠点B
[Peer]
PublicKey = ...
AllowedIPs = 192.168.255.3/32, 192.168.2.0/24
```

```
#子側（拠点A)
[Interface]
Address = 192.168.255.2/32
PrivateKey = ...

[Peer]
PublicKey = ...
# WireGurdネットワーク以外で、接続したいネットワークをすべて入れる
AllowedIPs = 192.168.255.0/24, 192.168.2.0/24, 192.168.3.0/24
```

中央側も子側もAllowedIPsに設定されているネットワークはWireGuardの接続であるwg0へ流れるようになります。

Debianで処理している場合は、`wg-quick donw wg0` として再びUPするなど、wg-quickで処理させないと、ルーティングまでは自動で反映されません。

子側からくるパケットの宛先は、WireGuardのネットワーク外であってもかまいません。その際は中央サーバー側の通常のルーティングテーブルに流れていき、経路が正しくせっていしてあればそちらへ流れます。また、WireGuardネットワークの外からくるパケットも同様に中央サーバーの処理で内側に流されます。

この機能を使うことで、たとえば拠点Aのネットワークである192.168.1.0/24からつながっている10.8.0.0/24への経路がダウンした時にWireGuardを使って別ルートを作れます。

その際は、拠点AのWireGuardのAllowedIPsに10.8.0.0/24を加えて中央に転送するようにします。
この時、拠点Aのデフォルトゲートウェイに10.8.0.0/24宛のパケットを拠点AのWireGuard端末に転送するようにルーティングするか、自身がデフォルトゲートウェイになります。

中央サーバーには通常のルーティングテーブルに10.8.0.0/24宛のエントリーが入っていなければ設定します。

同様に戻りのルートも設定します。10.8.0.0/24のネットワークのルーターにも、192.168.1.0/24ネットワークへのルーティングWireGuardの中央サーバー宛に送信するように設定します。

念のため断っておきますが、拠点Bの経路ダウンの原因が直下のインターネット回線切断によるものだったら回避はできません。

Debianでルーター機能を有効化する方法は前述しましたが、Windows機でルーター機能を有効にするには、次のようにします。

Windows機の場合は、外側のNICと、WireGuardトンネルに対して、Forwardingを設定しないといけません。

それらの指定をするための識別子と必要な設定の状態を確認します。

```
Get-NetIPInterface -AddressFamily IPv4 | Select-Object  InterfaceAlias, InterfaceIndex, InterfaceIndex, Forwarding, WeakHostSend, WeakHostRecieve

```

設定は、InterfaceAliasかInterfaceIndexを使って、Fowardingを指定します。

```
Set-NetIPInterface -InterfaceAlias "Ethernet"  -AddressFamily IPv4  -Forwarding Enabled 

Set-NetIPInterface -InterfaceIndex 32  -AddressFamily IPv4  -Forwarding Enabled  
```

ForwardingはNIC間の転送を許可するフラグです。WeakHostSend/RecieveはインターフェースとIPの結びつきの強さの設定で、デフォルト(Disabled)の状態だと、たとえば、実在NICの送信元IPアドレスを使ってWireGuard側へデータを送ることができません。
Fowardingを設定すればこれらの設定は不要だと思いますが、念のため覚えておくとよいと思います。

また通常、WindowsはWireGuardをPublicネットワークとして認識します。Privateにしておいた方が問題が少ないと思いますのでその際は、次のコマンドを実行します。

```
Set-NetConnectionProfile -InterfaceAlias "wg0" -NetworkCategory Private
```

注意が必要なのは、WireGuardを再起動するとWireGuardトンネルの転送やネットワークカテゴリ設定はデフォルトに戻るということです。


