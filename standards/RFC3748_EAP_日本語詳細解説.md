# RFC 3748 — Extensible Authentication Protocol — 日本語詳細解説
> 原典: https://www.rfc-editor.org/rfc/rfc3748.html  
> 文書の位置付け: Standards Track  
> **注意**: 原文の翻訳・転載ではなく、章・概念に対応づけた独自の日本語技術解説。条文の正確なMUST/SHOULD/MAY、更新関係、Errataは原典で最終確認する。IEEE規格は公開資料で確認できる範囲の解説。

## 0. この規格で何を理解するか
EAPは特定のパスワード方式ではなく、複数の認証方式（EAP Method）を共通のメッセージ枠に載せるプロトコル。802.1Xでは端末とAuthenticatorの間をEAPOLで、Authenticatorと認証サーバーの間をRADIUS/EAPで運ぶ。EAPの成功だけで通常通信を許してよいかは、下位レイヤーのアクセス制御の責務。

## 1. 規格の仕組みを順番に理解する
### 1.1 役割：Peer、Authenticator、Backend Authentication Server
Peerは認証される端末。AuthenticatorはEAPメッセージを中継して認証結果をアクセス制御へ反映する。Backend ServerがEAP Methodを終端する場合、Authenticatorは証明書やTLS鍵を検証しない。認証メソッドの終端主体とポートを解放する主体を分けて把握する。

### 1.2 Request/Response/Success/Failureの状態機械
基本はAuthenticatorからのRequestにPeerがResponseを返す。Identifierにより対応を管理する。Success/FailureにはTypeフィールドがない。EAP-Successは暗号化されたアプリ通信の開始メッセージではない。認証失敗時にControlled Portを開けない運用が必要。

### 1.3 IdentityとNotification
Identityは端末を識別するためのEAP Typeであり、それ自体は強い認証ではない。匿名外部Identityと証明書の内部IDを混同しない。Notificationは表示用であり、認証判定の根拠にしない。

### 1.4 EAP Methodの選択
EAPは拡張可能な枠組みで、Type 13はEAP-TLS。サーバーがMethodを提示し、Peerの対応状況と方針に基づき処理する。未対応方式の拒否や切替で弱い方式へダウングレードしないよう、両端の許容方式を限定する。

### 1.5 NakとExpanded Type
Legacy Nakは提案されたEAP Typeに対応できないときに代替方式を知らせる。Expanded Typeはベンダー別・拡張メソッドを識別する。『使えるものなら何でも受け入れる』運用では安全性が下がりうる。

### 1.6 再送、重複、Identifier
パケット損失を考慮し、Authenticatorが要求再送を制御する場合がある。重複Requestと同一Identifierへの応答を取り違えないこと。タイマー数値そのものは実装・環境で変わるため一律の標準値として扱わない。

### 1.7 EAPが保護しないもの
EAPフレームがそのまま暗号化されるわけではない。個々のEAP Methodの性質が重要。EAP-TLSはTLSを内部で利用するが、802.1Xの制御や後続のIPトラフィックが自動的にTLS化されるわけではない。

## 2. フィールド・データ・設定値の読み解き
| 項目 | 値・形式の例 | 意味と実装上の注意 |
|---|---|---|
| Code | 1=Request, 2=Response, 3=Success, 4=Failure | 先頭1 byte。認証メッセージ種別 |
| Identifier | 1 byte | 要求・応答の対応付け。同時セッション識別そのものではない |
| Length | 2 bytes, network byte order | EAPパケット全長。受信バッファ境界チェックが必要 |
| Type | Request/Responseに存在 | Identity=1, Notification=2, Nak=3, TLS=13等 |
| Type-Data | 可変長 | EAP Method固有の情報。TLSレコードを直接理解する必要はない中継実装もある |

### 表の読み方
ここに記載する識別番号・ビット長はプロトコル形式の説明、パス・タイマー・証明書名は設計例またはOSS固有の設定である。RFCが規定しないローカル設定を規格必須と誤解しないこと。

## 3. 通信・処理の具体例
```text
EAP Request (Identity): Code=1 | Identifier=0x07 | Length=5 | Type=1
EAP Response (Identity): Code=2 | Identifier=0x07 | Length=可変 | Type=1 | 端末ID
EAP Request (TLS): Code=1 | Identifier=0x08 | Type=13 | EAP-TLSフラグ | TLS断片
... 中間メッセージ ...
EAP Success: Code=3 | Identifier=一致する値 | Length=4
```
Identityの値が正しくてもEAP-TLS認証が成立したことにはならない。Successを送る条件はメソッドの検証成功に依存する。

## 4. OSS設定との対応（原理と実装を区別）
wpa_supplicant側は `eap=TLS` を選び、CA・端末証明書・秘密鍵・サーバー名検証ポリシーを持つ。AuthenticatorはEAPのリレーとポート状態を管理し、RADIUS ServerはEAP Methodを終端する。実装ログでは EAP identifier、現在のMethod、最終Success/Failureを対応させる。

## 5. 正常系・異常系で確認すること
- [ ] Identity送信だけではポートを開放しない
- [ ] 不正Lengthや未知Typeでメモリ範囲外を参照しない
- [ ] EAP-TLS以外への意図しない方式変更を許さない
- [ ] 同一Identifierの重複応答を正しく処理
- [ ] EAP-Failure時に通信を遮断
- [ ] 認証状態と実データ経路の制御が一致

## 6. 他規格との接続・読む順序
RFC 3579でRADIUSへのEAPカプセル化、RFC 9190でEAP-TLS 1.3、IEEE 802.1XでEAPOLとControlled Portを学ぶ。

## 7. 設計レビューで説明すべきこと
1. この規格が責任を持つ境界と、責任を持たない境界はどこか。
2. 端末とサーバーのうち、どちらが値を生成し、どちらが検証するか。
3. 資格情報が不正、メッセージが欠落、通信が途絶したときに閉じた状態（fail closed）になるか。
4. 取得したパケットキャプチャとログから規格上の判定理由を説明できるか。
5. 採用OSSの版で対応しているかを公式設定例と実測で確認したか。

---
**参照**: 上記原典。規格の本文を置換するものではない。セキュリティ実装では関連する更新RFC・Errata・ベンダーの実装ガイドも確認する。



## 深掘りA：EAPパケットは「Methodを入れる入れ物」

EAPはTLSそのものでもRADIUSそのものでもない。EAPのヘッダーは基本的に `Code(1) + Identifier(1) + Length(2)` で、Request/Responseだけ `Type` とType固有のDataが続く。RequestとResponseの対応づけはIdentifierを利用する。CodeがSuccess/FailureのときにTypeが存在すると思って5byte以上を必須として処理すると誤解析になる。

```text
01 07 00 05 01          EAP-Request / ID=7 / Length=5 / Type=Identity
02 07 00 0b 01 64 ...    EAP-Response / Identity（例、長さ要照合）
03 09 00 04             EAP-Success / ID=9 / Length=4
04 09 00 04             EAP-Failure / ID=9 / Length=4
```

2行目は先頭構造のイメージであり、正確なパケット長は実際のIdentityバイト列から算出する。16bit Lengthはネットワークバイトオーダー。

### Request/Responseの処理モデル

1. AuthenticatorはEAP-Request/Identityを送る。
2. Peerは同じIdentifierでEAP-Response/Identityを返す。
3. EAPサーバーが認証方式を選択してEAP-Request(TLS=Type13)を送る。
4. Peer/EAPサーバー間でMethod固有の交換が行われる。
5. Methodが成功するとEAP-Successが下位層へ結果を伝える。

Identityを知っていることや、TLS証明書を送ったこと自体は成功条件ではない。TLSハンドシェイク・証明書検証が完了して、EAP結果の整合が取れて初めて許可候補になる。

## 深掘りB：Nak・タイムアウト・下位層

Peerが提示されたMethodを実行できない場合、Nakなどを通じて別Methodを示せる。ただしサーバー側が安全でない方式への変更を受け入れると、意図しないダウングレードが起こる。今回の端末認証ではEAP-TLSだけを許可するポリシーを最初に決める。

EAP自体は通信の再送タイマー・リンク確立をすべて規定するわけではない。802.1XのEAPOL上ではL2フレーム、RADIUS運搬ではRADIUSの再送、EAP-TLS内部ではTLSハンドシェイクが関係する。どの層が再送したのかを分けてログに残す必要がある。

EAPが成立しても、無線LANなら鍵導出・暗号化、EthernetならControlled Portの許可という「後処理」は別にある。EAP-Success = 全IP通信の暗号化完了ではない。

**根拠を読む場所**：RFC 3748 §2（EAPの概念）、§4（Packet Format）、§5（Request/Response Types）、§7（Security Considerations）。


## 深掘り演習：設計レビューで必ず説明できるようにする

**この規格の核**：EAPはMethodに依存しない状態管理。EAPOL・RADIUS・TLSのどこが処理を止めたかの識別が運用で重要になる。

### 演習 1 — Identityは一致しているがTLSは失敗
**観察する現象**：Identityは一致しているがTLSは失敗。まず問題が起きている層を特定する。端末上のwpa_supplicantログ、NASの802.1X/RADIUSログ、RADIUS/EAPサーバー側の認証結果、実際のフレーム・パケットキャプチャを時刻とセッション識別子で突合する。
**考える順序**：①この規格が直接定める条件と、他規格・製品固有の条件を分離する。②ローカル設定値と通信で実際に送信された値を比較する。③認証、認可、通信経路の開放状態を別々に検証する。④不明な条件で通信を勝手に解放しない。
**具体的な確認**：EAPはMethodに依存しない状態管理。EAPOL・RADIUS・TLSのどこが処理を止めたかの識別が運用で重要になる。 試験環境では正常な要求を1つ保存してから、変更した条件を1つだけ注入し、送信側・受信側・転送装置のログがどのように変わったかを確認する。

### 演習 2 — Method Nakで弱い方式に落ちる
**観察する現象**：Method Nakで弱い方式に落ちる。まず問題が起きている層を特定する。端末上のwpa_supplicantログ、NASの802.1X/RADIUSログ、RADIUS/EAPサーバー側の認証結果、実際のフレーム・パケットキャプチャを時刻とセッション識別子で突合する。
**考える順序**：①この規格が直接定める条件と、他規格・製品固有の条件を分離する。②ローカル設定値と通信で実際に送信された値を比較する。③認証、認可、通信経路の開放状態を別々に検証する。④不明な条件で通信を勝手に解放しない。
**具体的な確認**：EAPはMethodに依存しない状態管理。EAPOL・RADIUS・TLSのどこが処理を止めたかの識別が運用で重要になる。 試験環境では正常な要求を1つ保存してから、変更した条件を1つだけ注入し、送信側・受信側・転送装置のログがどのように変わったかを確認する。

### 演習 3 — EAP-SuccessにTypeを期待した
**観察する現象**：EAP-SuccessにTypeを期待した。まず問題が起きている層を特定する。端末上のwpa_supplicantログ、NASの802.1X/RADIUSログ、RADIUS/EAPサーバー側の認証結果、実際のフレーム・パケットキャプチャを時刻とセッション識別子で突合する。
**考える順序**：①この規格が直接定める条件と、他規格・製品固有の条件を分離する。②ローカル設定値と通信で実際に送信された値を比較する。③認証、認可、通信経路の開放状態を別々に検証する。④不明な条件で通信を勝手に解放しない。
**具体的な確認**：EAPはMethodに依存しない状態管理。EAPOL・RADIUS・TLSのどこが処理を止めたかの識別が運用で重要になる。 試験環境では正常な要求を1つ保存してから、変更した条件を1つだけ注入し、送信側・受信側・転送装置のログがどのように変わったかを確認する。

### 演習 4 — 同じIdentifierの再送が発生
**観察する現象**：同じIdentifierの再送が発生。まず問題が起きている層を特定する。端末上のwpa_supplicantログ、NASの802.1X/RADIUSログ、RADIUS/EAPサーバー側の認証結果、実際のフレーム・パケットキャプチャを時刻とセッション識別子で突合する。
**考える順序**：①この規格が直接定める条件と、他規格・製品固有の条件を分離する。②ローカル設定値と通信で実際に送信された値を比較する。③認証、認可、通信経路の開放状態を別々に検証する。④不明な条件で通信を勝手に解放しない。
**具体的な確認**：EAPはMethodに依存しない状態管理。EAPOL・RADIUS・TLSのどこが処理を止めたかの識別が運用で重要になる。 試験環境では正常な要求を1つ保存してから、変更した条件を1つだけ注入し、送信側・受信側・転送装置のログがどのように変わったかを確認する。

## 規格文書を読んで照合する方法
以下の順に原文を確認すると、説明を単純な「機能紹介」で終わらせず、規格に根拠のある設計へ落とせる。
1. 冒頭のScope/Introductionでこの規格が定義しない処理を確定する。
2. Protocol Overview/Message Flowを追い、各メッセージの生成者・検証者・次状態を並べる。
3. Packet Format/Attribute/Extensionの定義から、長さ・値域・順序・必須条件を確認する。
4. Security Considerationsから、なりすまし・改ざん・再送・失効・ダウングレードの例外条件を拾う。
5. IANA Considerations、Errata、Updates/Updated byを確認し、古い番号表や旧TLS挙動をそのまま採用しない。
6. OSSの設定名とRFCの用語が一致しない場合、OSSの実装を確認して対応表を作る。

> 注意：本書は原典に基づく**独自の詳細解説**であり、著作権上の原文の全文翻訳ではない。特にIEEEの本文限定情報は推測で補っていない。MUST等の厳密な適合性判定には該当条文とErrataを直接確認する。