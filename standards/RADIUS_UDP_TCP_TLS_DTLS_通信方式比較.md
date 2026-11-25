# RADIUSのトランスポート方式：UDP / TCP / TLS / DTLS 詳細比較

> 対象：802.1X Authenticator（RADIUSクライアント）とRADIUSサーバー間。**Supplicant ↔ AuthenticatorのEAPOL通信とは別の区間**である。  
> 作成：2026-11-26。本文は原文の逐語訳ではなく、規格に基づく独自の日本語解説。IETF Internet-Draftの内容は正式RFCと区別する。

## 1. なぜRADIUSに複数の転送方式があるのか

RADIUSは「Access-Request」「Access-Challenge」「Access-Accept」「Access-Reject」などの認証メッセージを交換する。メッセージ自体の意味と、それをネットワークでどう運ぶかは別問題である。オリジナルのRADIUSはUDPを利用し、後からTCP、TLS、DTLSを使う方式が追加された。

ここで分けて考えるべき特性は次の3つ。

- **到達性と順序**：TCPはバイトストリームの順序保持・再送・フロー制御を提供する。ただし、切断後に相手アプリが処理したかまで保証するわけではない。RADIUSの応答確認や再試行の判断は引き続き必要。
- **秘匿性と完全性**：TLS/DTLSは、適切に構成した場合、通信経路上の盗聴・改ざんを防ぐ。単純なTCPへの置換では得られない。
- **相手の認証**：TLS/DTLSでは証明書やPSK等により通信相手を認証する。RADIUS内のEAP-TLSによる**端末認証**と、RADIUS/TLSでの**RADIUSピア認証**は別。

## 2. プロトコル階層

```text
RADIUS over UDP                RADIUS over TCP
┌─────────────┐               ┌─────────────┐
│   RADIUS    │               │   RADIUS    │
├─────────────┤               ├─────────────┤
│     UDP     │               │     TCP     │
├─────────────┤               ├─────────────┤
│     IP      │               │     IP      │
├─────────────┤               ├─────────────┤
│  Ethernet   │               │  Ethernet   │
└─────────────┘               └─────────────┘

RADIUS over TLS                RADIUS over DTLS
┌─────────────┐               ┌─────────────┐
│   RADIUS    │               │   RADIUS    │
├─────────────┤               ├─────────────┤
│     TLS     │               │    DTLS     │
├─────────────┤               ├─────────────┤
│     TCP     │               │     UDP     │
├─────────────┤               ├─────────────┤
│     IP      │               │     IP      │
├─────────────┤               ├─────────────┤
│  Ethernet   │               │  Ethernet   │
└─────────────┘               └─────────────┘
```

IPの下にEthernetを描くのは典型的な有線LANの例であり、IPが必ずEthernetに載るという意味ではない。

## 3. 比較表

| 方式 | 根拠文書 | 下位転送 | 通信路の暗号化 | 順序保証・再送 | 注意 |
|---|---|---|---|---|---|
| RADIUS/UDP | RFC 2865 / 2866 | UDP | なし | UDPは提供しない | 従来RADIUSの一部属性保護やAuthenticatorはTLSによる全体暗号化とは異なる |
| RADIUS/TCP | RFC 6613 (Experimental) | TCP | なし | TCPが提供 | TCPだけでは盗聴を防げない |
| RADIUS/TLS | RFC 6614 (Experimental) | TLS over TCP | あり | TCPが提供 | 通称RadSec。RADIUSピア証明書検証等の設計が必要 |
| RADIUS/DTLS | RFC 7360 (Experimental) | DTLS over UDP | あり | UDPのデータグラム全体の到達や順序は保証しない | DTLSは暗号化と改ざん検出を提供、アプリ側は再送等を考慮 |

**注意**：TLSはパケット配送の最終的な業務成功まで保証しない。またDTLSのハンドシェイク再送は、RADIUSアプリケーションデータの確実な配送保証ではない。

## 4. RFC 2865：RADIUS over UDP

### 処理イメージ

```text
Authenticator                    RADIUS server
    | --- UDP Access-Request ---> |
    | <--- Access-Challenge ----  |
    | --- UDP Access-Request ---> |
    | <--- Access-Accept -------  |
```

RADIUSクライアントはRequestに識別子を付け、対応するResponseを判別する。応答が来なければタイムアウトや再送・別サーバーへの切替を判断する。UDPのため、パケットは損失・重複・順不同到着の可能性がある。

基本のRADIUSパケット：

```text
Code (1 octet) | Identifier (1) | Length (2)
Authenticator (16 octets) | Attributes (可変長)
```

標準的な認証宛先はUDP/1812、AccountingはUDP/1813（古いポート番号との混同に注意）。従来RADIUSのshared secretとMessage-Authenticatorは、全属性を暗号化する仕組みではない。保護されていないネットワークへ平文UDP RADIUSを流さない。

参考：[RFC 2865](https://www.rfc-editor.org/rfc/rfc2865.html)、[RFC 2866](https://www.rfc-editor.org/rfc/rfc2866.html)。

## 5. RFC 6613：RADIUS over TCP

RFC 6613はRADIUSメッセージをTCPで転送する実験的方式を定める。RADIUSのCode・Identifier・Authenticator・Attributeの基本形式は変わらない。UDPのデータグラム境界の代わりに、TCPバイトストリーム上でRADIUSの`Length`を用いてメッセージを切り出す。

TCPは再送・順序保証を行うが、TCP接続が生存していることとRADIUSサーバーが認証処理を実行できることは同義ではない。そのためRFC 6613はStatus-Server等のアプリケーション層の生存確認を扱う。

TCPだけではTLSの秘密性・相互認証を得られない。通常の新規導入で「安全にしたいからUDPをTCPに変更する」は十分な対策ではない。

参考：[RFC 6613 §2.1–2.6](https://www.rfc-editor.org/rfc/rfc6613.html)。

## 6. RFC 6614：RADIUS over TLS（RadSec）

```text
Authenticator / RADIUS client           RADIUS server
        | --- TCP connection setup ---> |
        | <---- TCP established ------- |
        | <---- TLS handshake --------> |
        | ==== TLS-protected RADIUS ===> |
        | <=== TLS-protected response == |
```

RADIUS/TLSではTLSの中にRADIUSパケットを運ぶ。TCP/2083が通常用いられる。TLS証明書の検証、相手識別子との結びつき、信頼するCA、鍵更新、再接続時の処理が重要になる。

**誤解しやすい点**：EAP-TLSとRADIUS/TLSは違う。
- EAP-TLS：SupplicantとEAPサーバー間で行う**端末認証方式**。AuthenticatorがEAPを中継。
- RADIUS/TLS：Authenticator/RADIUSクライアントとRADIUSサーバー間の**通信路保護**。

両者は併用できる。

参考：[RFC 6614](https://www.rfc-editor.org/rfc/rfc6614.html)、[RFC 6613](https://www.rfc-editor.org/rfc/rfc6613.html)。

## 7. RFC 7360：RADIUS over DTLS

RFC 7360は、RADIUSをUDPのままDTLSで保護するExperimental文書。TLSに似たセキュリティ機能をデータグラム用に提供するが、UDPの配送特性そのものは変わらない。

```text
Authenticator / RADIUS client           RADIUS server
        | <--- DTLS handshake --------> |
        | -- encrypted datagram ------> |
        | <- encrypted datagram ------- |
```

RFC 7360 §2.1の重要な規定には、RADIUSのLength検査を**復号後のデータ長**を基準に行うこと、従来のRADIUSに残るMD5計算や属性暗号化に用いるshared secretとして固定文字列 `radius/dtls` を用いることがある。これはTLS/DTLS保護の存在を無視して、従来方式のshared secretを勝手に転用してよいという意味ではない。

DTLSで通信が保護されても、RADIUS要求の重複・損失・アプリケーション応答待ちの扱いは必要である。

参考：[RFC 7360 §2](https://www.rfc-editor.org/rfc/rfc7360.html)。

## 8. 802.1Xシーケンスのどこに適用するか

```text
[Supplicant]
    | EAPOL / EAP-TLS      ※ Ethernet上、非IPでも開始可能
    v
[Authenticator / NAS]
    | RADIUS over UDP/TCP/TLS/DTLS のいずれか
    v
[RADIUS / EAP server]
```

**4種類のRADIUS転送方式は下段のリンクの選択であり、上段のEAPOLを置き換えない。** またKeylimeによるRemote Attestation（通常のIP/TLS通信）とも別。

## 9. 実装・セキュリティ検証で確認する項目

1. AuthenticatorとRADIUSサーバーが選択した転送方式を双方でサポートするか。
2. Port番号とTCP/UDPの違いが一致しているか。
3. TLS/DTLSの場合、証明書チェーン、SANによる相手識別、失効・更新方針が適切か。
4. TCPの接続断、再接続時に処理中の要求・応答をどう扱うか。
5. DTLS/UDPで損失・重複・再送が起きた際に認証を誤許可しないか。
6. パケットキャプチャでRADIUSのAttributesが読めるか（平文転送）／TLS/DTLSにより保護されるか。
7. EAP-TLSによる端末証明書検証とRADIUSピアのTLS認証が混同されていないか。

## 10. 規格の現況（2026-11時点）

RFC 6613、6614、7360はいずれも**Experimental**として発行された。RFC 9765はRADIUS/1.1とALPN `radius/1.1` を定義し、TLS/DTLS上のRADIUSの歴史的な一部MD5依存を見直すExperimental仕様である。これらを単に「RADIUS/TCP/TLS/DTLSと同義」として一括りにしない。

IETFの `draft-ietf-radext-radiusdtls-bis` はRFC 6614と7360を置き換えることを目指しているが、**Internet-DraftはRFCとしての確定済み規格ではない**。採用時には最新版と承認状況を再確認する。

参考：[RFC 9765](https://www.rfc-editor.org/rfc/rfc9765.html)、[IETF Datatracker: radiusdtls-bis](https://datatracker.ietf.org/doc/draft-ietf-radext-radiusdtls-bis/)。
