# Crumple Zone Architecture

ブラウザを信頼し、健全なWebアプリケーションを作るためのアーキテクチャ。「壊れ方」から設計を考える。

正しさとメンテナンス性が重要なWebアプリケーションを対象とする。MPA + Islands を基盤とする。

提供するもの:

* 不具合の低減 — クライアント状態を構造的に最小化する。状態が少なければ壊れる箇所も少ない
* 標準でセキュア — ルールに従って実装するだけで、セキュアな構成になる
* メンテナンス性 — フロントエンドは必要に応じて作り直せる薄さを保つ
* AIエージェント互換 — スキルファイルがAIに一貫した実装基準を与える

原則:

リロードすれば正しい状態が再構築される。ユーザー体験をブラウザと一緒に作り込む。
設計原則の詳細は [architecture.md](architecture.md) を参照。

英語版: [../](../)

## ドキュメント

[**skill/crz.md**](../skill/crz.md) — Astro 実装スキル。自己完結。開発時にAIエージェントに渡す。

[**architecture.md**](architecture.md) — スキルのルールの背景にある設計原則。AIが生成したコードのレビューや、判断に迷うケースで追加コンテキストとして使う。

[**extensions.md**](extensions.md) — MPA + Islands を超えるパターン: リアルタイム更新、キャッシュ層、楽観的な更新、セキュリティ上の考慮事項。

[**operations.md**](operations.md) — AI駆動開発におけるCRZの展開方法: ドキュメントの役割、2エージェント分離、スキルプロンプトの作成。

## セットアップ

スキルファイルをAIエージェントの設定にコピーする:

### Claude Code（プロジェクト）

```bash
mkdir -p .claude/skills/crz && curl -o .claude/skills/crz/SKILL.md https://raw.githubusercontent.com/koji-1009/crumple-zone-architecture/main/skill/crz.md
```

### Claude Code（グローバル）

```bash
mkdir -p ~/.claude/skills/crz && curl -o ~/.claude/skills/crz/SKILL.md https://raw.githubusercontent.com/koji-1009/crumple-zone-architecture/main/skill/crz.md
```

### Cursor

```bash
mkdir -p .cursor/rules && curl -o .cursor/rules/crz.md https://raw.githubusercontent.com/koji-1009/crumple-zone-architecture/main/skill/crz.md
```

### Codex

```bash
curl -o AGENTS.md https://raw.githubusercontent.com/koji-1009/crumple-zone-architecture/main/skill/crz.md
```

Claude Code では、タスクがスキルの description に合致すると自動でロードされる。`/crz` による明示的な呼び出しも引き続き使える — こちらが決定論的な経路である。コマンドの打ち忘れはサイレント故障だが、スキルの description がそれを自動回復に変える。

## License

MIT
