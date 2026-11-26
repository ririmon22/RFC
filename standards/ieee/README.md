# IEEE 802.1X / 802.1AR / 802.1AE — 日本語技術教材集

初版：2026-11-26。原典の内容を踏まえた**独自の日本語技術解説**であり、公認翻訳・原本の逐語訳ではない。原典本文を入手していない部分や条文の適合確認を要する部分は、それぞれの資料で注意書きした。

## 読む順序

1. [802.1AR：DevIDと証明書](02_IEEE8021AR_DevIDと証明書_詳細教材.md) — IDevID、LDevID、鍵・CA・ライフサイクル
2. [802.1X：ポート認証とMKA](01_IEEE8021X_ポート認証とMKA_詳細教材.md) — EAPOL、Controlled Port、RADIUS、MKA
3. [802.1AE：MACsec](03_IEEE8021AE_MACsec_詳細教材.md) — フレーム・SecTAG・SAK・ICV・PN・リプレイ保護
4. [統合シーケンス・設計](04_IEEE8021X_AR_AE_統合シーケンスと設計_詳細教材.md) — 初期登録から802.1X、MKA、MACsec、TPMまで

## 根拠と限界

- IEEEの公開公式ページと原本・原文目次の確認できた範囲を参照。
- 802.1AR-2018は公開原文からClause 6–9等の記述を確認。802.1X-2020は原文の目次・序文と公式説明を確認。
- 802.1AEの原本の逐条照合は全項目完了していないため、すべてのフィールドの規範値を保証する資料ではない。実装時に原本・追補・訂正を照合すること。
- 802.1AR-2018に対する2026年のML-DSA追補、および802.1X-2020/Cor 1-2026、802.1AEの改訂・追加規格を区別して扱う。
- TPM Remote Attestationの統合・Policy Controllerによる許可制御は**今回のシステム向け設計例**でありIEEE 802.1X/AR/AEのみが規定する機能ではない。

## 原典

- https://1.ieee802.org/security/802-1x/
- https://1.ieee802.org/security/802-1ar/
- https://1.ieee802.org/security/802-1ae/
- https://www.rfc-editor.org/
