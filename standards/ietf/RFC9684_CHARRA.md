# RFC 9684 — CHARRAによるTPM証跡のYANG取得

[公式原典](https://www.rfc-editor.org/rfc/rfc9684.html)（Standards Track）。§2のYANGツリーとRPC・YANGモジュール、Security Considerationsを照合する。ラボではTPM 2.0の経路に絞る。

## 1. 論理フォーマットと輸送

| RPC／要素 | 方向 | フィールドと注意 |
|---|---|---|
| tpm20-challenge-response-attestation | Verifier → YANG Server | TPM 2.0用のnonce、バンクとPCR indexの選択、複数TPMを扱う場合の証明書名指定 |
| tpm20-attestation-response | YANG Server → Verifier | certificate-name、TPMS_QUOTE_INFO、条件に応じたquote-signature等。型と階層は原典を優先 |
| log-retrieval | Verifier ↔ YANG Server | log-type、log-selectorを指定。取得済み範囲と今回の範囲を記録 |
| NETCONFまたはRESTCONF | 管理接続 | 選択した輸送に応じてXMLやJSONを使い、YANGのbinary型では規定の符号化を使う |

RFC 9684はYANGモデルでRPCの意味を定義する。YANG定義、RESTCONF/NETCONFシリアライズ、binaryに入るTPMの構造体は三つの層。キャプチャがTLSで暗号化されている場合はアプリログでRPC名、選択、応答長を確認する。採取ファイルにはTPM署名の公開検証に必要なデータのみ収録する。

## 2. 正常系と設定

1. Verifierが意思決定に必要なバンクとPCR範囲を選択し、1回分のnonceを生成する。
2. 認可済み管理クライアントがTPM 2.0 RPCを呼ぶ。YANG Serverは指定のTPM、証明書／鍵、PCRを選択し、TSS経由でQuoteを取得する。
3. Verifierは返却された署名対象データを復号ではなくデコードし、登録済みAKで署名を検証する。要求と返却のnonce、PCR選択を照合する。
4. 必要なログ範囲を取得し、再計算・参考値評価を行う。応答とログの時点差も扱う。

［ラボ設定］共用サーバーからA/Bの隔離アドレスへの管理接続のみ許可。管理サービスのTLS／SSH認証鍵とQuote用AKを分離し、管理ユーザーには対象RPCと関連読み取り権限を割り当てる。信頼台帳には端末ID、AK、certificate-name、バンクの組を置く。nonceは32バイトのラボ値を試し、要求とTPM候補の制約を確認して採用する。

## 3. OSS対応と観測

YANGのスキーマ検査は [pyang](https://github.com/mbj4668/pyang) 等、TPM呼出しは [tpm2-tss](https://github.com/tpm2-software/tpm2-tss) が候補。NETCONF Serverを選んでもCHARRA RPCのTPMバックエンドは別途必要で、対応リリースとモジュールを確認する。Keylimeの通常のAgent APIはCHARRAの相互運用試験と別ケースにする。相互運用の合否は、スキーマ読み込み、RPC呼出し、TPM生成、独立した署名・ログ検証の順に記録する。

## 4. 異常系と設計演習

| 条件 | 採点基準 |
|---|---|
| 指定バンクやTPMがない | エラーが可視化され、評価器が空応答をPASSに変換しない |
| 管理クライアントにRPC権限がない | Quoteが発行されず、認証／認可エラーを記録 |
| binary型を誤ってデコードする | 構造体や署名検証を通過せず、ログ再生に進まない |
| ログの増分要求に連続性がない | 既知のチェックポイントから再取得し、不確かな範囲では隔離を維持 |
| Aの証明書名や応答をB向けに誤設定 | 管理接続先、登録AK、要求nonceのうち少なくとも一つの照合で拒否 |

演習としてAにのみTPM 2.0機能を有効化し、Bと異なるバンク構成を与える。YANG機能発見からエラーの分類までを図示し、AKの証明書を更新したとき旧certificate-nameへの参照をどう失効させるかを書く。TLS通信の成功、RPC成功、Evidence受理、通常通信の4つを個別に採点する。

## 5. 相互運用を切り分ける読み方

原典 §2.1.1.3.2 のYANGツリーとXML例を並べて、`ietf-tpm-remote-attestation` のモジュールnamespace、RPC名、TPM 2.0機能、応答の型を確認する。ログ取得は §2.1.1.3.3 の `log-retrieval` を確認する。クライアントの選択は「SHA-256バンクのPCR 0〜7」のようにバンクと範囲を組で記録し、文字列から暗黙に推測しない。

- NETCONFで管理する場合、メッセージIDでRPC応答を対応付け、nonceではQuoteの鮮度を検証する。両者の一致は別々に確認する。
- `certificate-name` は参照名。信頼済み公開鍵に対応付ける処理と、管理アクセス制御（NACM等）がRPCアクセスを許可する処理を別々に確認する。
- ログ取得を分割する場合、取得済み範囲・次位置・Quoteの時点を記録する。端末再起動後は前回取得位置を引き継ぐ前に連続性を評価する。
