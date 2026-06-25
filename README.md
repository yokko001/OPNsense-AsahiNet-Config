# OPNsense + ASAHIネット IPoE(RA方式) + DS-Lite + RFC7278(NDP Proxy) セットアップ手順

## 対象環境

-   OPNsense 26.x
-   ASAHIネット IPoE
-   フレッツ光ネクスト（ひかり電話なし）
-   WAN IPv6：RA（SLAAC）
-   IPv4：DS-Lite
-   IPv6：RFC7278 + NDP Proxy

------------------------------------------------------------------------

# ネットワーク構成

``` text
                Internet
                    │
          フレッツ光ネクスト
                    │
                  ONU
                    │
                WAN(ix0)
              OPNsense
        +------------------+
        |                  |
        |  DS-Lite(gif0)   |
        |  NDP Proxy       |
        +------------------+
                    │
               LAN(igc0)
                    │
          Windows / Linux
```

------------------------------------------------------------------------

# 1. WAN設定

## IPv4

-   Configuration Type: **None**

## IPv6

-   Configuration Type: **SLAAC**

ASAHIネット（ひかり電話なし）はRA方式でIPv6アドレスを取得する。

取得確認:

``` sh
ifconfig ix0
```

グローバルIPv6アドレス（2405:...）が表示されればOK。

------------------------------------------------------------------------

# 2. DS-Liteプラグイン導入

使用プラグイン

https://github.com/kawaii-not-kawaii/ds-lite-opnsense

GitHubから取得し、scpでOPNsenseへ転送。

展開後、`/usr/local/opnsense/...` 配下へファイルを配置し、

``` sh
service configd restart
```

を実行。

------------------------------------------------------------------------

# 3. DS-Lite設定

Interfaces → DS-Lite

-   Enable: ON
-   Mode: DS-Lite
-   ISP Profile: Auto Detect
-   WAN Interface: WAN

Apply後、

-   Status: Connected
-   Connectivity: OK
-   Tunnel IPv4: 192.0.0.2

となることを確認。

IPv4疎通確認:

``` sh
ping 8.8.8.8
```

------------------------------------------------------------------------

# 4. LAN設定

IPv4

-   Static
-   192.168.1.1/24

IPv6

-   Link-local

※ Track Interface は使用しない。

------------------------------------------------------------------------

# 5. NDP Proxy

Services → NDP Proxy

-   Enable
-   Upstream: WAN
-   Downstream: LAN
-   Proxy Router Advertisements: ON
-   Install host routes: ON

動作確認:

``` sh
ps aux | grep ndp
```

`ndp-proxy-go` が起動していること。

# NDP Proxy導入後の追加設定（重要）

`ndp-proxy-go` はインストールしただけでは正常に動作しません。

以下の公式ドキュメントに従い、ファイアウォールルール等の追加設定を実施してください。

https://docs.opnsense.org/manual/ndp-proxy-go.html#firewall-rules

特に以下の設定は必須です。

- WAN → LAN の ICMPv6 通信許可
- Neighbor Solicitation (NS)
- Neighbor Advertisement (NA)
- Router Solicitation (RS)
- Router Advertisement (RA)
- MLD (Multicast Listener Discovery)
- 必要に応じて Floating Rule の追加

今回の環境では、この設定だけでは不十分で、さらに OPNsense が自動生成する

```
let out anything from firewall host itself (force gw)
```

による `route-to` ルールが RFC7278 構成と競合していました。

最終的には、この自動生成ルールより優先順位の高い Floating Rule を追加して `route-to` を無効化することで正常動作しました。


------------------------------------------------------------------------

# 6. Router Advertisement

Services → Router Advertisements → LAN

-   Mode: Managed

Windowsで

``` cmd
ipconfig
```

を実行し、2405:... のIPv6アドレスが取得できることを確認。

------------------------------------------------------------------------

# 7. RFC7278確認

WindowsとWANが同じ /64 プレフィックスになっていること。

``` sh
netstat -rn -f inet6
```

で確認可能。

------------------------------------------------------------------------

# 8. DS-Lite確認

Diagnostics画面で

-   Connected
-   Connectivity OK
-   Tunnel IPv4 = 192.0.0.2

となること。

------------------------------------------------------------------------

# 9. 発生した問題

WindowsではIPv6アドレスは取得できるが、

``` cmd
ping -6 ipv6.google.com
```

が失敗し、

https://test-ipv6.com

でも 0/10 となる。

------------------------------------------------------------------------

# 10. 原因

原因はOPNsenseが自動生成する Floating Rule。

    let out anything from firewall host itself (force gw)

このルールが `route-to` を強制することで、RFC7278構成（WANとLANで同一
/64）と競合し、LANクライアント向けIPv6通信までWANへ送ろうとしてしまう。

確認:

``` sh
pfctl -a '*' -sr | grep route-to
```

------------------------------------------------------------------------

# 11. 解決方法

Floating Rules に自動生成された **force gateway** ルールより上位の
**Quick Pass** ルールを作成し、

-   Gateway: None

として `route-to` を適用させないようにする。

適用後、

``` cmd
ping -6 ipv6.google.com
```

が成功し、

https://test-ipv6.com

も正常（10/10）となった。

------------------------------------------------------------------------

# 最終構成

  項目        設定
  ----------- ---------------------
  WAN IPv4    None
  WAN IPv6    SLAAC
  LAN IPv4    192.168.1.1/24
  LAN IPv6    Link-local
  RA          Managed
  NDP Proxy   Enabled
  DS-Lite     Enabled
  IPv4        DS-Lite
  IPv6        RFC7278 + NDP Proxy

------------------------------------------------------------------------

# 備考

この構成は、ASAHIネット（RA方式）で配布される単一の /64
プレフィックスをRFC7278によりLANへ延伸し、DS-LiteでIPv4通信を提供する構成である。

通常のDHCPv6-PDとは異なり、WANとLANで同一プレフィックスを共有するため、OPNsenseの自動生成する
`route-to` ルールとの競合に注意する必要がある。
