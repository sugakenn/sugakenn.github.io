---
layout: post
title: "nftableの初期設定"
date: 2026-08-01 17:00:00 +0900
description: nftablesの初期設定に関して注意点をまとめています
img: 2026/2026-08-01-1.png # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [debian,routing]
categories: 
  - debian
---

ファイアウォールを設定すれば安心というわけではないのですが、それでも設定しておかないと何かと心配です。
また後付けで稼働させようとすると既存の稼働中の機能に対する設定を洗い出す必要もあり、わりと大変なので最初から設定しておく方が良いと思います。

現代のlinux環境に合わせて、今回はnftablesの初期設定をAIに相談しながら検討してみました。

なお、[nftablesのコマンド類]()、については別記事を見ていただければ幸いです。

## ルール記述の設計

iptablesではIPv6とIPv4の設定は別管理になっていましたが、nftablesでは混在させることも可能です。

両方で受け付けることを前提としているなら、inet テーブルを使って共通で書いた方が漏れが少なくなると思います。
「ip6 nexthdr ipv6-icmp accept」などのどちらかのバージョン固有の設定も記述することができます。
このとき該当しないバージョンのパケットは「ルールに一致しない」という挙動になります。

逆に、IPv4での通信することを前提としたネットワークでIPv6のアドレスも配布されているような環境では、ip とip6 のテーブルを分けた方が楽だと思います。
この場合、ipテーブルにはIPv6固有のルールは書けませんし、ip6テーブルにIPv4固有のルールは書けません。(`meta l4proto ipv6-icmp accept`などエラーにはならないものもあるようです)

同じフックのチェーンが複数あった場合はそれぞれのチェーンに設定されたプライオリティ順に判定されます。
同じプライオリティのチェーンがある場合は、どちらから処理されるかは不定です。
(混同しやすいので念のため書いておきますが、チェーン内のルールの判定順は記述順の保証があります)

テーブルは inet ip ip6を混在させたり、同種のテーブルを複数持つことも可能ですが、分かりづらくなるので避けた方が無難です。

テーブルがひとつの場合でも複数の場合でもチェーンはフック別に一つにまとめられて処理されます。

ひとつのチェーン内ではルールに一致したらそこで終了し、結果がacceptなら次のチェーンに移ります、一つ目がacceptでも次のチェーンがdropならdropとなります。
ルールかポリシーでdropの判定がでると、次のチェーンには移らずdropとなります。

まとめると、**dropは即終了、acceptはまだ覆る可能性がある** という事です。

テーブルに設定する ip ip6 inetなどをファミリと呼びます。

## ポリシー
各ベースチェーンには、記述したルールに一致しなかった場合に適用されるポリシーが設定できます。これはiptabls時代同様です。

昔から、inputとforwardはdrop、outputはaccept というのが標準的に案内されています。

その状態から少しセキュリティレベルをあげるならログを取ります。

```
# uid が0 root出ない場合にログを取る
meta skuid != 0 log

# 新しい接続をカウント(必要ならログを取る)
ct state new counter (log)
```
もちろん、用途的にoutputが限られるならdropにした方がセキュリティは向上します。

その際でもrootで実行するaptは必要かとおもいますので、次の設定ぐらいは必要になると思います。
```
meta skuid 0 tcp dport {80,443} accept

meta skuid 0 udp dport 53 accept
```
## デフォルト設定
inputやoutputポリシーをdorpにする場合、運用上最低限の必要な接続許可はしておかないといけません。それらには次のようなものがあります。

### ローカルホストの許可
linuxにおいてローカルホストはloインターフェースとして設定されていて、この通信を阻害するとサービスが正常に動かなくなる恐れもあるので、デフォルト設定というより必須設定になると思います。
```
#インターフェースを指定
iifname "lo" accept
```
フックのouptputをdropにする場合は、そちらにも書いておきます。ループバック通信は、自分自身の中で完結する通信ですので、forwardには不要です。

### ICMPの許可

IPv4の場合は、Ping応答をを許可しないという選択肢もあると思いますが、IPv6ではARPに相当するNS(Neighbor Solicitation)/NA(Neighbor Advertisement)がICMPv6上で動くので、ICMPv6をすべて許可しておくのが一般的です。


#### IPv4で許可する typeを指定する場合
```
(ip protocol icmp) icmp type { destination-unreachable, time-exceeded, parameter-problem, echo-reply} accept
```

#### IPv4でまとめて許可する場合
```
ip  protocol icmp accept
```

#### IPv6の場合
```
meta l4proto ipv6-icmp accept
```

#### なぜIPv4とIPv6で指定の書式が違うのか

なぜIPv4とIPv6で指定の書式が違うのか疑問に思った人もいるかもしれませんが、それはIPv4とIPv6の仕様の違いからくるものです。

プロトコル番号というデータの中身を示す番号(正確にはIPヘッダの次の内容を示す為の番号)があり、IPv4ではIPヘッダの「プロトコル番号」フィールドに格納されています。

IPv6に変わる際に拡張されフィル―ド名が「Next Header」と変わり、複数のヘッダを持つことができるようになりました。

通常IPv6のNext HeaderにはTCPなら6、UDPなら17、ICMPなら58が入りますが、拡張ヘッダ(可変長)がある場合はそのヘッダを示すプロトコル番号となります。拡張ヘッダは、さらにNext Headerを保持し、次に伝播するようになっています。
つまりその場合は、TCP(6)やUDP(17)などのプロトコル番号は拡張ヘッダのNext Headerへ格納されることになります。

TCPやUDPヘッダなどははIPv6でも仕様は変わりませんので、Next Headerフィールドはありません。それらのヘッダ以降の処理はIPv4時代と同じになります(ヘッダのあとにはデータグラムが来ます)。

ESPなどでカプセル化される時には、続くデータグラムからカプセル化を解いた後のヘッダのプロトコル番号が入っていることもあります。

IPv6の拡張ヘッダには、ルーティングを示すRouting Header(43)や、送信者が設定した(IPv6では経路中に新たにフラグメントはおきません)フラグメントを示すFragment Header(44)などがあります。

これらはプロトコルではないですが、IANA の「Protocol Numbers」に登録されています。

前置きが長くなりましたが、まずこのような経緯でIPv4とIPv6でフィールド名が違うので指定の仕方が変わります。IPv4の場合は、フィールド名がプロトコル番号なので次のような指定方法になります。
```
ip  protocol icmp accept
```
これをそのままIPv6に置き換えるとプロトコル番号→Next Headerで、
```
ip6 nexthdr ipv6-icmp
```
となり、この書式は有効です。しかしこの書き方だと問題が起きることがあります。それは、先ほどあったようにNext Headerは複数存在することがあるということに起因します。

ip6 nexthdrは先頭のNext Headerしか判定しません(IPヘッダの次のヘッダしか判定しません)。そのため拡張ヘッダがあると取りこぼしてしまいます。そのため次のような記述となります。
```
meta l4proto ipv6-icmp accept
```
これは最終的なNext Headerの値を判定するための書式で、ここではそれがIPv6ICMPなら accept としています。

ちなみに、拡張ヘッダのいずれかに存在するルーティングヘッダを対象とするルールを作るなら、
```
exthdr rt exists accept
```
とします。なんとなくこれにipv6-icmp を指定してもいいような気がしますが、ipv6-icmpは **拡張**ヘッダではなく標準のヘッダなのでうまくいきません。

### 定番の記述

自身が接続を開始した場合には、戻りの接続を許可するというiptablsの定番の設定に

```
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```
というものががありました。

これをnftablesに書き換えると
```
ct state established,related accept
```
となり、これをinputフックに設定します。

ctはConnection Tracking(接続追跡)の略で、様々な状態判定ができます。stateの他には、同時接続数を制限する

```
# 1つのIPから同時に3つ以上のSSH接続が来たら破棄する
tcp dport 22 ct count >= 3 drop
```
などがあります。

ちなみに上記でiptablesやnftablesで指定したESTABLISHEDは、TCPにおけるESTABLISHEDとは少し意味が異なり、UDPにおいてもESTABLISHEDの概念があります。


## その他の設定

他は環境に合わせて設定をしていくことになりますが、iptablesから仕様が変わった主な点を解説しておこうと思います。

### L2フィルタリング

nftablesは ebtablesやarptablesも統合しています。

ebtablesで行っていたようなMACアドレスに対してする制限は、inet ip ip6 ファミリのチェーンに対して`ether`キーワードを使ってルールを記述できます。inputのdaddrと、outputのsaddrは常に自分になるので指定できません。

```
table inet my_filter {
    chain input {
        type filter hook input priority filter; policy drop;

        # 特定のMACアドレスからの通信であれば、すべてのポートへの接続を許可
        ether saddr 00:11:22:33:44:55 accept

        # 特定のMACアドレスからのSSH許可
        ether saddr 00:aa:bb:cc:dd:ee tcp dport 22 accept

        # 確立済みの通信の維持
        ct state established,related accept
    }
}
```
arptablesで行っていた、ARPスプーフィング対策や、ロードバランサ等のARP応答抑制はnftablesでは、arpファミリを使って書きます。

```
# arptablesの代わりとなる専用のテーブルを定義
table arp my_arp_filter {
    chain input {
        type filter hook input priority filter; policy accept;

        # 192.168.1.10 の MACが00:11:22:33:44:55でない場合はdrop
        arp saddr ip 192.168.1.10 arp saddr ether != 00:11:22:33:44:55 drop
    }

    chain output {
        type filter hook output priority filter; policy accept;
        
        # ロードバランサの応答抑制
        oifname "eth0" arp operation reply drop
    }
}
```

### netdev
netdevはnftablesから新設されたファミリで、インターフェースに入った瞬間や出る瞬間をとらえられるフックを持つファミリです。hookは ingress(入った瞬間)とegress(出る瞬間)があります。

DOS攻撃などを早めに判定して拒否する際に役立ちます。

```
table netdev global_blacklist {
    # 遮断したいIPアドレスのブラックリスト（動的に追加も可能）
    set banned_ips {
        type ipv4_addr
        elements = { 192.0.2.1, 198.51.100.10, 203.0.113.5 }
    }

    chain my_ingress {
        type filter hook ingress device "eth0" priority filter; policy accept;

        # セットに登録されているIPからの通信を最速で遮断
        ip saddr @banned_ips drop
    }
}
```

### トンネル時の挙動
これは iptables時代から変わりはありませんが、トンネル処理をしているサーバーの場合は、カプセル化された外側のデータはinputフィルターに入り、カプセル化を解除したデータは解除後の宛先に応じて、inputまたはforwardに入ります。
外側のパケットの宛先により INPUT または FORWARD に入り、その後トンネル解除処理があった場合は、新たなパケットとして再度ルーティングテーブルに渡されます。その際に宛先が自分なら input 他なら forward のフィルターが適用されます。


## サンプル設定
```
#!/usr/sbin/nft -f

flush ruleset

table ip filter {
        chain input {
                type filter hook input priority filter;policy drop;
                ip protocol icmp accept
                iifname "lo" accept
                ct state established, related accept
                # IPv4だけsshを許可
                tcp dport 22 ct state new accept
        }
        chain forward {
                type filter hook forward priority filter;policy drop;
        }
        chain output {
                type filter hook output priority filter;policy accept;
        }
}

table ip6 filter {
        chain input {
                type filter hook input priority filter;policy drop;
                meta l4proto ipv6-icmp accept
                iifname "lo" accept
                ct state established, related accept
        }
        chain forward {
                type filter hook forward priority filter;policy drop;
        }
        chain output {
                type filter hook output priority filter;policy accept;
        }
}
```