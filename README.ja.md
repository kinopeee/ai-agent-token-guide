# AIエージェントのトークン消費を最小化する実践ガイド

日本語 | [English](./README.md)

![AIエージェントのトークン消費を最小化する実践ガイド](assets/x-card.ja.png)

AIエージェントの入力、コンテキスト、ツール実行、検証、モデル選択を見直し、無駄なトークン消費を減らすための実践ガイドです。目標は、必要な品質と完了時間を満たしながら、検証に合格したタスクあたりの総コストを下げることです。

全文は [ai-agent-token-guide.ja.md](./ai-agent-token-guide.ja.md) です。

## 8つの原則

1. [入力を減らす](./ai-agent-token-guide.ja.md#原則1入力を減らす)
2. [コンテキストの寿命を管理する](./ai-agent-token-guide.ja.md#原則2コンテキストの寿命を管理する)
3. [往復を減らす](./ai-agent-token-guide.ja.md#原則3往復を減らす)
4. [やり直しを早く止める（タスク設計と検証器）](./ai-agent-token-guide.ja.md#原則4やり直しを早く止めるタスク設計と検証器)
5. [出力を減らす](./ai-agent-token-guide.ja.md#原則5出力を減らす)
6. [プロンプトキャッシュを壊さない](./ai-agent-token-guide.ja.md#原則6プロンプトキャッシュを壊さない)
7. [モデルと推論量を使い分ける](./ai-agent-token-guide.ja.md#原則7モデルと推論量を使い分ける)
8. [計測してから最適化する](./ai-agent-token-guide.ja.md#原則8計測してから最適化する)

何から試すか迷ったら、[導入の優先順位チェックリスト](./ai-agent-token-guide.ja.md#導入の優先順位チェックリスト)の「[今日できること](./ai-agent-token-guide.ja.md#今日できること設定変更なし)」から始めてください。設定を変えずに試せる項目です。

間違っている点などがあったら、PRを送ってください。

## ライセンス

文章と画像は [CC BY 4.0](./LICENSE)、ガイド内のコード例は [MIT](./LICENSE-CODE) で公開しています。

2026年10月25日 · [Kinopee](https://x.com/kinopee_ai)
