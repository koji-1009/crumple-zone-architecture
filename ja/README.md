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

[skill/crz.md](../skill/crz/skills/crz/SKILL.md) — Astro 実装スキル。自己完結。開発時にAIエージェントに渡す。

[skill/sieve.md](../skill/sieve/skills/sieve/SKILL.md) — CSS 実装スキル。crz.md のコンパニオン。スタイリングを扱うタスクでは crz.md と併せて渡す。

[architecture.md](architecture.md) — スキルのルールの背景にある設計原則。AIが生成したコードのレビューや、判断に迷うケースで追加コンテキストとして使う。

[extensions.md](extensions.md) — MPA + Islands を超えるパターン: リアルタイム更新、キャッシュ層、楽観的な更新、セキュリティ上の考慮事項。

[references/api-maturity.md](references/api-maturity.md) — Baseline による採用規則が誤った答えを出すブラウザAPIと、その代替。architecture.md 5.4 から参照される。

[operations.md](operations.md) — AI駆動開発におけるCRZの展開方法: ドキュメントの役割、2エージェント分離、スキルプロンプトの作成。

## セットアップ

### Claude Code（プラグイン）

このリポジトリは Claude Code のプラグインマーケットプレイスである。追加したうえで crz と、スタイリングを扱う場合は sieve をインストールする:

```bash
claude plugin marketplace add koji-1009/crumple-zone-architecture
claude plugin install crz@crumple-zone-architecture
claude plugin install sieve@crumple-zone-architecture
```

既定ではユーザースコープにインストールされる。`--scope project` または `--scope local` で変更できる。スキルは `/crz:crz` と `/sieve:sieve` で呼び出す。

サードパーティのマーケットプレイスは自動更新が既定で無効である。`/plugin` → Marketplaces で有効にするか、手動で更新する:

```bash
claude plugin marketplace update crumple-zone-architecture
claude plugin update crz@crumple-zone-architecture
claude plugin update sieve@crumple-zone-architecture
```

### Codex（プラグイン）

Codex も同じマーケットプレイスを読む。追加したうえで crz と、スタイリングを扱う場合は sieve をインストールする:

```bash
codex plugin marketplace add koji-1009/crumple-zone-architecture
codex plugin add crz@crumple-zone-architecture
codex plugin add sieve@crumple-zone-architecture
```

Claude Code と Codex では、タスクが frontmatter の description に合致すると自動でロードされる — crz は Astro 実装、sieve はスタイリングで。Claude Code では `/crz:crz` と `/sieve:sieve` による明示的な呼び出しも引き続き使える — こちらが決定論的な経路である。コマンドの打ち忘れはサイレント故障だが、自動ロードがそれを回復に変える。スキルの実体は `skill/<name>/skills/<name>/SKILL.md` の1ファイルのみで、Claude Code と Codex が共通で読むプラグインとして配置している。`skill/crz.md` / `skill/sieve.md` はそこへのシンボリックリンクである。

## License

MIT
