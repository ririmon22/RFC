# TCG TPM 2.0 Library — PCR・Quote・登録鍵を追う

[TCG公式仕様一覧](https://www.trustedcomputinggroup.org/resources/documents/tpm-library-specification)。本教材では Family 2.0、Revision 01.83 を参照版とする。Part 1はアーキテクチャ、Part 2は構造体、Part 3はコマンド。実装の改訂・Errataを台帳に記録する。

## 1. PCRと鍵

`PCR_after = Hash_bank(PCR_before || event_digest)`。同じイベント集合でも順序、対象PCR、バンク、初期値が違うと結果が変わる。PCR値は測定履歴の集約、イベントログはその説明。アプリケーションはLinux TPM driver／TPM resource manager、TSS、TPMコマンド境界を経由する。

| 鍵・オブジェクト | 用途 | 管理 |
|---|---|---|
| EK | TPMの登録／由来の評価 | EK証明書やラボの登録根拠に照合。プロビジョニングで扱う |
| AK | Quote等の署名 | Verifierが登録済み公開鍵と対応付ける。再生成・廃止を端末ライフサイクルに反映 |
| EAP-TLS端末鍵 | 802.1Xでの本人性 | 別の証明書用途。TPMに配置する場合もAKとの対応を別途台帳化 |
| PCR | 計測ダイジェストの集約 | 選択とアルゴリズムをプロファイル化。複数バンクが存在しうる |

EKのみを登録して端末の任意の署名鍵をAKとして扱うとTPM由来の根拠が失われる。採用プロファイルに応じてMakeCredential／ActivateCredential等を用い、AKのNameと、同一TPM内で保護される関係を評価する。資格情報発行元、登録の認可、失効のタイミングまで決める。

## 2. ワイヤ形式と署名範囲

| 層 | 形式 |
|---|---|
| コマンドヘッダー | tag 2、commandSize 4、commandCode 4オクテット。TPMの多バイト値は原典のマーシャリングに従う |
| TPM2_Quote入力 | signHandle、qualifyingData、inScheme、PCRselect。セッション領域を使うコマンド形式ではその領域も存在 |
| Quote結果 | TPM2B_ATTESTで運ぶ署名対象とTPMT_SIGNATURE。外側サイズやAPI固有の出力ファイルを署名対象と混同しない |
| TPMS_ATTEST | magic、type、qualifiedSigner、extraData、clockInfo、firmwareVersion、型固有領域 |
| TPMS_QUOTE_INFO | PCR選択とpcrDigest。要求した選択、順序、ダイジェスト計算方式を確認 |

`TPM2_Quote` のCommand Codeは0x00000158。VerifierはQuote type、nonceに対応するextraData、AK、署名、PCR selectionとpcrDigestを照合する。構造体をCのメモリレイアウトのまま送受信せず、可変長とバイトオーダーを意識してTSSで扱う。

## 3. 正常系と手元検証

1. PCRを測定プロファイルに従ってExtendし、ログを保存する。
2. 登録済みAKで、チャレンジを qualifyingData として Quote を作る。
3. Verifierが署名・nonce・バンク/PCRを検証し、受け取ったPCR値やログ再計算を署名済みpcrDigestに結び付ける。
4. 測定ポリシーを評価する。署名成功、ログ整合、期待値適合は別の判定を記録する。

ラボでは [tpm2-tools](https://github.com/tpm2-software/tpm2-tools) の `tpm2_pcrread`、`tpm2_quote`、`tpm2_checkquote` と [tpm2-tss](https://github.com/tpm2-software/tpm2-tss) を利用する。 `TPM2TOOLS_TCTI` で使うデバイス／ソケットを固定し、A/Bでswtpm状態を別ディレクトリにする。例として `sha256:0,2,4,7` を選び、対象環境のログと計測目的に応じて増減する。出力形式は固定したツール版のマニュアルで確認する。

## 4. 異常系と設計演習

| 条件 | 期待 |
|---|---|
| wrong AK、署名の一部違い | 登録鍵／署名検証で失敗 |
| 同じQuoteを新しい要求に再利用 | nonce不一致で失敗 |
| バンクやindexを変更 | 要求したPCR selectionとの照合で失敗 |
| PCR値照会とQuoteの間で測定が増える | 整合失敗。再要求で原因を判定 |
| swtpm状態の置換 | AKの変化または旧チャレンジとの不整合を検出。仮想TPMの信頼境界はホストを含む |

演習ではAとBに別のAKを登録して、同じ測定ダイジェストを持つ構成でも結果を取り違えないことを示す。続いてPCRを一つ追加し、Quote構造体のどのフィールドとVerifierのどの設定が変わるかを説明する。

## 5. Quoteを独立に検証する実験

端末側では、登録済み `ak.ctx` とVerifierが発行した `CHALLENGE_HEX` を使用する。長期鍵を別システムへコピーする操作は含まない。

```bash
tpm2_quote -c ak.ctx -l sha256:0,2,4,7 -g sha256 \
    -q "${CHALLENGE_HEX}" -m quote.msg -s quote.sig -o quote.pcrs
```

Verifier側の `akpub.pem` は登録経路から取得する。到着した応答の公開鍵でその応答自身を信頼する設計は避ける。

```bash
tpm2_checkquote -u akpub.pem -g sha256 -q "${CHALLENGE_HEX}" \
    -m quote.msg -s quote.sig -f quote.pcrs
```

[ツール公式例](https://tpm2-tools.readthedocs.io/en/latest/man/tpm2_checkquote.1/)を参照する。署名・PCR値の照合を通した後、要求したPCR selectionが過不足なく返ったか、計測ログを独立に再生できるか、そのログと参考値が適合するかを別工程で検査する。nonceが同じかのテストでは、Verifierのチャレンジだけを変更して同じ証跡を評価し直す。失敗を観測したら別nonceで新規Quoteを作り、成功へ戻ることを確認する。

### データの切り分け

- `quote.msg` はTPMの署名対象、`quote.sig` は署名表現、`quote.pcrs` は使用ツールの出力形式。ネットワークで運ぶ際に形式とサイズ上限をプロファイル化する。
- `TPML_PCR_SELECTION` には選択集合が入り、要素ごとにバンクのアルゴリズムとPCRを指定するビット列がある。PCR 7の選択とPCR 7のダイジェストは別データ。
- Quoteの `clockInfo` が示す情報とサーバー側の発行／失効時刻は時間源が異なる。チャレンジ照合を省略する根拠にはしない。
