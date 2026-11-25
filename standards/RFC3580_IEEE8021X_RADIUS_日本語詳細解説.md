# RFC 3580 — IEEE 802.1XでのRADIUS利用指針 — 日本語詳細解説
> 原典: https://www.rfc-editor.org/rfc/rfc3580.html  
> 文書の位置付け: Informational（IEEE 802.1Xとの利用ガイド）  
> **注意**: 原文の翻訳・転載ではなく、章・概念に対応づけた独自の日本語技術解説。条文の正確なMUST/SHOULD/MAY、更新関係、Errataは原典で最終確認する。IEEE規格は公開資料で確認できる範囲の解説。

## 0. この規格で何を理解するか
802.1X AuthenticatorがRADIUS属性を使って端末・ポートを識別し、認証結果と認可ポリシーを取得する方法を解説する文書。規格の位置づけはInformational。802.1Xの実体であるポート制御はIEEE側が定義する。

## 1. 規格の仕組みを順番に理解する
### 1.1 NASは誰か
802.1XのAuthenticatorがRADIUSクライアント＝NAS。SupplicantはRADIUSパケットを直接送らない。NAS-IP-AddressやNAS-IdentifierはAuthenticatorを識別するための情報で、SupplicantのIPとは別。

### 1.2 ポートの識別
NAS-PortとNAS-Port-Typeで認証対象のポートを表現する。Calling-Station-Idには通常端末MACに関連する情報、Called-Station-Idには接続先関連の識別を載せる。値の表現方法をRADIUS Serverのポリシーで一致させる。

### 1.3 EAPと802.1Xの関係
EAPOLは端末とAuthenticator間のL2通信。RADIUSにおけるEAP-Messageの運搬はRFC 3579。RADIUS ServerはEAP Method（例えばEAP-TLS）を処理し、Authenticatorはポート制御結果として扱う。

### 1.4 VLAN割当が意味すること
RADIUS Access-AcceptでTunnel-Type、Tunnel-Medium-Type、Tunnel-Private-Group-IDの組合せを使いVLANを指示できる。Tunnel-Private-Group-IDには通常VLAN IDを文字列で指定する。標準属性を送ってもAuthenticatorのVLAN切替機能がなければ作用しない。

### 1.5 Session-TimeoutとTermination-Action
Session-Timeoutは単なる再認証固定タイマーではない。Termination-Actionや適用コンテキストで動作が変わる。実機の再認証/セッション終了処理とRADIUS属性解釈の対応を検証する。

### 1.6 Filter-IdとACL
RADIUSがFilter-Idでフィルターを指示しても、フィルターの具体的な内容はNAS側の実装・事前設定に依存する場合がある。Attestationが未完了のときは隔離ACLだけを有効にする設計が可能。

### 1.7 802.1Xの成功と健全性の成功を分離
端末証明書が有効でも起動状態が健全とは限らない。RADIUS Access-Acceptを隔離ネットワークへの認可とし、KeylimeのPASS後にCoA等で通常通信へ変更するのはシステム統合設計であり、このRFCがKeylimeを規定するわけではない。

## 2. フィールド・データ・設定値の読み解き
| 項目 | 値・形式の例 | 意味と実装上の注意 |
|---|---|---|
| NAS-IP-Address | Type 4 | NASのIPv4アドレス。端末IPとは異なる |
| NAS-Port | Type 5 | NAS側ポート識別子 |
| NAS-Identifier | Type 32 | NASを識別する文字列 |
| Calling-Station-Id | Type 31 | 接続端末の識別情報として用いる |
| NAS-Port-Type | Type 61 | IEEE 802系のポートタイプを識別 |
| Tunnel-Type | Type 64: VLAN=13 | VLANを指定する際の構成要素 |
| Tunnel-Medium-Type | Type 65: IEEE-802=6 | 対象トンネル媒体を示す |
| Tunnel-Private-Group-ID | Type 81: 例 "100" | VLAN IDを表す文字列として扱う |
| Session-Timeout | Type 27 | 再認証等への影響はTermination-Actionと合わせて確認 |

### 表の読み方
ここに記載する識別番号・ビット長はプロトコル形式の説明、パス・タイマー・証明書名は設計例またはOSS固有の設定である。RFCが規定しないローカル設定を規格必須と誤解しないこと。

## 3. 通信・処理の具体例
```text
Access-Accept（隔離VLAN割当の例）
  Tunnel-Type = VLAN
  Tunnel-Medium-Type = IEEE-802
  Tunnel-Private-Group-ID = "100"
  Session-Timeout = 3600  # ここでは設計例
```
VLAN 100をKeylime Verifierにしか到達できない隔離セグメントとして用意する場合、通信制御を担当するスイッチ/hostapd/nftables側が属性の適用を実現しなければならない。

## 4. OSS設定との対応（原理と実装を区別）
FreeRADIUSでは認証条件に応じて返信属性を生成する。hostapd等が属性に対応しているかはドライバ・Linux Bridge・L2スイッチ構成で異なる。`Tunnel-*`属性を書いただけでLinux内のVLANインターフェースが自動的に完成するわけではない。

## 5. 正常系・異常系で確認すること
- [ ] NAS識別子と端末識別子を混同しない
- [ ] VLAN IDの文字列形式が正しい
- [ ] RADIUS側のAccess-Acceptと実ポートのACL/VLANが一致
- [ ] 未実装属性を安全側で扱う
- [ ] Session-Timeout後の振る舞いを確認
- [ ] Attestation失敗時に通常VLANへ入らない

## 6. 他規格との接続・読む順序
IEEE 802.1Xのポート制御、RFC 2865の属性基本、RFC 3579のEAP運搬、RFC 5176の動的認可変更と併読する。

## 7. 設計レビューで説明すべきこと
1. この規格が責任を持つ境界と、責任を持たない境界はどこか。
2. 端末とサーバーのうち、どちらが値を生成し、どちらが検証するか。
3. 資格情報が不正、メッセージが欠落、通信が途絶したときに閉じた状態（fail closed）になるか。
4. 取得したパケットキャプチャとログから規格上の判定理由を説明できるか。
5. 採用OSSの版で対応しているかを公式設定例と実測で確認したか。

---
**参照**: 上記原典。規格の本文を置換するものではない。セキュリティ実装では関連する更新RFC・Errata・ベンダーの実装ガイドも確認する。



## 深掘りA：RADIUSは何を認証し、何を認可するのか

802.1XのPeerはMACを持つ端末だが、RADIUSが信頼性の強いIDとして扱うべきものはEAP-TLSの認証結果・証明書ポリシーである。`Calling-Station-Id` に入る端末MACは機器識別やログ検索に役立つ一方、MACは詐称できるため証明書の代わりにできない。`NAS-Port` はNAS内部でのポート識別であり、通信経路上のTCP/UDPポート番号とは無関係。

### 分離する4つのID

| ID | 生成主体 | 用途 | 混同すると起こる問題 |
|---|---|---|---|
| EAP-Response/Identity | Supplicant | ルーティング・アカウント候補 | Identityだけで強い認証が成立したと誤解 |
| 証明書SAN等 | CAと証明書所持者 | 認証した装置のID | 外部Identityと不整合 |
| Calling-Station-Id | NAS | 接続端末のL2観測 | MACを秘密鍵と同等に信頼 |
| NAS-Identifier/NAS-Port | NAS | 認証箇所の特定 | RADIUSクライアントと端末のIDを混同 |

## 深掘りB：VLAN属性は「命令」ではなく「認可情報」

RADIUSサーバーはAccess-Acceptに `Tunnel-Type=VLAN(13)`、`Tunnel-Medium-Type=IEEE-802(6)`、`Tunnel-Private-Group-ID="100"` といった属性を付けられる。実際にVLANを変更するのはNASまたはスイッチであり、RADIUSサーバーはネットワーク転送処理そのものを持たない。

今回の設計では、`VLAN 100 = Attestation専用`、`VLAN 200 = 通常通信` としたくなる。ただし**VLAN番号はこのRFCが定める値ではなく設計者の割当**である。Access-Acceptを返した時点では端末のブート健全性まで確認できていないなら、まず隔離VLANに入り、Keylime PASS後にCoAやアクセス制御APIで変更する。

### 認証後のタイマー

`Session-Timeout` はRADIUS属性として定義されるが、これを受け取ったNASが再認証を開始するのか、セッションを終えるのかは `Termination-Action` との組合せや機器動作を確認する。`3600` はしばしば設定例として使う秒数であり、普遍的な規格要求ではない。

**根拠を読む場所**：RFC 3580 §3（RADIUS利用）、特に識別子、VLAN/フィルタ、セッション管理の記述。RADIUS Attribute自体の値はRFC 2865・2868等に遡って確認する。本文がInformationalである点にも注意。


## 深掘り演習：設計レビューで必ず説明できるようにする

**この規格の核**：NASの申告する接続元情報、EAPで認証されたID、実際のVLAN/ACLという三つの層を一致させる。MACやNAS-Portは認証結果の暗号的証拠ではない。

### 演習 1 — MACアドレスだけで本人認証する
**観察する現象**：MACアドレスだけで本人認証する。まず問題が起きている層を特定する。端末上のwpa_supplicantログ、NASの802.1X/RADIUSログ、RADIUS/EAPサーバー側の認証結果、実際のフレーム・パケットキャプチャを時刻とセッション識別子で突合する。
**考える順序**：①この規格が直接定める条件と、他規格・製品固有の条件を分離する。②ローカル設定値と通信で実際に送信された値を比較する。③認証、認可、通信経路の開放状態を別々に検証する。④不明な条件で通信を勝手に解放しない。
**具体的な確認**：NASの申告する接続元情報、EAPで認証されたID、実際のVLAN/ACLという三つの層を一致させる。MACやNAS-Portは認証結果の暗号的証拠ではない。 試験環境では正常な要求を1つ保存してから、変更した条件を1つだけ注入し、送信側・受信側・転送装置のログがどのように変わったかを確認する。

### 演習 2 — Tunnel属性を送ったのにVLANが変わらない
**観察する現象**：Tunnel属性を送ったのにVLANが変わらない。まず問題が起きている層を特定する。端末上のwpa_supplicantログ、NASの802.1X/RADIUSログ、RADIUS/EAPサーバー側の認証結果、実際のフレーム・パケットキャプチャを時刻とセッション識別子で突合する。
**考える順序**：①この規格が直接定める条件と、他規格・製品固有の条件を分離する。②ローカル設定値と通信で実際に送信された値を比較する。③認証、認可、通信経路の開放状態を別々に検証する。④不明な条件で通信を勝手に解放しない。
**具体的な確認**：NASの申告する接続元情報、EAPで認証されたID、実際のVLAN/ACLという三つの層を一致させる。MACやNAS-Portは認証結果の暗号的証拠ではない。 試験環境では正常な要求を1つ保存してから、変更した条件を1つだけ注入し、送信側・受信側・転送装置のログがどのように変わったかを確認する。

### 演習 3 — Session-Timeout後に認証が切れる
**観察する現象**：Session-Timeout後に認証が切れる。まず問題が起きている層を特定する。端末上のwpa_supplicantログ、NASの802.1X/RADIUSログ、RADIUS/EAPサーバー側の認証結果、実際のフレーム・パケットキャプチャを時刻とセッション識別子で突合する。
**考える順序**：①この規格が直接定める条件と、他規格・製品固有の条件を分離する。②ローカル設定値と通信で実際に送信された値を比較する。③認証、認可、通信経路の開放状態を別々に検証する。④不明な条件で通信を勝手に解放しない。
**具体的な確認**：NASの申告する接続元情報、EAPで認証されたID、実際のVLAN/ACLという三つの層を一致させる。MACやNAS-Portは認証結果の暗号的証拠ではない。 試験環境では正常な要求を1つ保存してから、変更した条件を1つだけ注入し、送信側・受信側・転送装置のログがどのように変わったかを確認する。

### 演習 4 — 同じ端末が別のNASから認証する
**観察する現象**：同じ端末が別のNASから認証する。まず問題が起きている層を特定する。端末上のwpa_supplicantログ、NASの802.1X/RADIUSログ、RADIUS/EAPサーバー側の認証結果、実際のフレーム・パケットキャプチャを時刻とセッション識別子で突合する。
**考える順序**：①この規格が直接定める条件と、他規格・製品固有の条件を分離する。②ローカル設定値と通信で実際に送信された値を比較する。③認証、認可、通信経路の開放状態を別々に検証する。④不明な条件で通信を勝手に解放しない。
**具体的な確認**：NASの申告する接続元情報、EAPで認証されたID、実際のVLAN/ACLという三つの層を一致させる。MACやNAS-Portは認証結果の暗号的証拠ではない。 試験環境では正常な要求を1つ保存してから、変更した条件を1つだけ注入し、送信側・受信側・転送装置のログがどのように変わったかを確認する。

## 規格文書を読んで照合する方法
以下の順に原文を確認すると、説明を単純な「機能紹介」で終わらせず、規格に根拠のある設計へ落とせる。
1. 冒頭のScope/Introductionでこの規格が定義しない処理を確定する。
2. Protocol Overview/Message Flowを追い、各メッセージの生成者・検証者・次状態を並べる。
3. Packet Format/Attribute/Extensionの定義から、長さ・値域・順序・必須条件を確認する。
4. Security Considerationsから、なりすまし・改ざん・再送・失効・ダウングレードの例外条件を拾う。
5. IANA Considerations、Errata、Updates/Updated byを確認し、古い番号表や旧TLS挙動をそのまま採用しない。
6. OSSの設定名とRFCの用語が一致しない場合、OSSの実装を確認して対応表を作る。

> 注意：本書は原典に基づく**独自の詳細解説**であり、著作権上の原文の全文翻訳ではない。特にIEEEの本文限定情報は推測で補っていない。MUST等の厳密な適合性判定には該当条文とErrataを直接確認する。