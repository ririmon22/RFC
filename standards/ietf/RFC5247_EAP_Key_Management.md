# RFC 5247 — EAP鍵管理とMACsecへの受け渡し

原典は [RFC 5247](https://www.rfc-editor.org/rfc/rfc5247.html)（Standards Track、RFC 3748を更新）。特に §1.4、§2、§3、§4、§5。EAP-TLS 1.3の鍵導出は [RFC 9190 §2.3](https://www.rfc-editor.org/rfc/rfc9190.html#section-2.3)、IEEE側への接続は [802.1X教材](../ieee/README.md) を参照する。

## 1. 鍵を誰が持つか

| 材料 | 生成・保有主体 | スコープと廃棄 |
|---|---|---|
| TLS長期秘密鍵 | PeerとEAP Serverが各自保有 | 証明書の本人性確認。失効時は証明書と台帳を更新 |
| MSK | PeerとEAP Serverがメソッドで独立導出 | 最低64オクテット。独立したAuthenticator構成では必要な鍵材料を認証済みAuthenticatorへ配布 |
| EMSK | PeerとEAP Serverが独立導出 | 最低64オクテット。Authenticatorへの配布対象外 |
| CAK / CKN | 802.1Xの選択した方式でMKA参加者が得る | CAKは秘密、CKNはCAKを選択する名前。リンク／参加者と認証セッションに対応付ける |
| SAK | MKA Key Serverが生成し保護して配布 | MACsec SAで使用。切替時は受信準備と送信切替、廃棄を管理 |

EAP-TLS 1.3では exporter 出力128オクテットの前半64をMSK、後半64をEMSKとする。EAPメソッドが生成する Session-Id と Backend RADIUSの Identifier、NASの Acct-Session-Id は用途も寿命も異なる。認証成功応答が到達したこと、MSKを受け取ったこと、MKA鍵が導入されたことは、別々の観測点で判定できる。

## 2. 通信フォーマットと正常シーケンス

RFC 5247はフレーム形式を新設せず、鍵の生成・輸送・使用条件を定める。典型的なEAP-TLS/RADIUS構成では、EAP交換は端末–NASでEAPOL、NAS–サーバーで EAP-Message 属性。旧来のRADIUS鍵受け渡しでは [RFC 2548 §2.4](https://www.rfc-editor.org/rfc/rfc2548.html#section-2.4) の Microsoft Vendor-Specific Attribute（Vendor-ID 311、Send-Key 16／Recv-Key 17）が現れる。Saltは2オクテット。鍵属性の暗号化とバックエンド全体の輸送保護は異なる機能で、採用したRADIUSプロファイルに従う。ダンプには属性の有無のみ残し、値と共有秘密を収録しない。

1. 端末とEAP ServerがTLSの本人性を確認し、同じセッションのMSKをそれぞれ導出する。
2. EAP ServerがAccess-Acceptで認証結果と必要な鍵材料を当該Authenticatorへ返す。
3. NASはレスポンスを対応する要求・ポート・端末に結び付ける。端末とNASは802.1Xの規則でMKAの入力を取得する。
4. MKAがピアを確認し、Key ServerがSAKを配布する。SecYが送受信SAを導入して暗号化通信が始まる。
5. リンク切断、セッション失効、再認証失敗で旧認可を解除し、旧鍵をスコープに従い廃棄する。

## 3. 実装値と検証

| 設定・選択 | 例 | 変更で観測するもの |
|---|---|---|
| メソッド | wpa_supplicantの `eap=TLS` | 証明書認証、バックエンドでの鍵属性生成 |
| Backend経路 | 管理ネットワーク上の対応TLSプロファイル | NASの真正性、TLSの接続先、方式不一致時の拒否 |
| EAP再認証 | ラボでは300秒を目安にNASで明示設定 | 旧認可の終了時刻、新セッションへの対応付け |
| MKA | 動的CAK方式を明示。比較試験でのみ事前共有CAK | EAP失敗後に静的鍵で通常通信が開かないこと |

FreeRADIUSはEAP Server、wpa_supplicantはPeer。Authenticatorの鍵受領とMKA起動は採用実装／ビルドで確認する。Linuxの `ip macsec` はSA表示・設定に使える。EAPやMSK輸送の起動は別途必要である。

| テスト | 合格条件 |
|---|---|
| EAP成功・正しい鍵輸送 | 同一NASセッションにAcceptとMKA完了が対応し、両方向の保護カウンタが増える |
| Acceptのみ／鍵輸送不可 | 認証ログに成功が出ても暗号化必須のデータ経路は閉じる |
| Aの受け渡しとBの応答を取り違える設定 | Bの通常通信が開かず、セッションの不一致が監査に残る |
| 再認証失敗 | 旧SAが残っても通常通信はポリシーの期限までに停止し、解除条件をログで追える |

### 設計演習

A/Bを同時認証させてからAのみ再認証する。MSK、CAK、SAK、RADIUS Session-Id相当、NASのポートIDについて「更新主体・寿命・観測点」を記述し、B側の認可が維持される証跡を示す。MSKとEMSKが同じ箇所へ渡る設計や、EAP成功だけをMACsec成立とみなす合格判定は修正する。
