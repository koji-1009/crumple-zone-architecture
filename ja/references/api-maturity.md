# ブラウザAPIの例外

architecture.md 5.4 が採用の規則を定める。Baseline Newly Available では、採用はプロジェクトのサポート下限に対する判断であり、Widely Available では到達率は問いでなくなる。ほとんどのAPIはこの規則以上のものを必要とせず、ここには何も記録しない。

このファイルが扱うのは例外 — 規則が誤った答えを出すAPIと、代わりに取るべき手段である。Drag and Drop は Widely Available でありながら回避すべきである。`<datalist>` は機能検出を通過したうえでエンジンごとに異なる挙動をする。項目がここに載る条件は、規則が予測できない挙動を、取るべき代替とともに名指しすることである。サポート状況のデータは、それを保守している MDN と Baseline に属する。判断がそれに左右される場合に限り、ステータスをここに書く。

最終レビュー: 2026-08

## 概要

| API | 規則が誤る理由 | 判断 | 再評価 |
| --- | --- | --- | --- |
| Drag and Drop | Widely Available だが、仕様がマウスイベントの上に構築されている | 回避。操作が必須要件なら隔離 | Pointer Events の上に再構築された時 |
| `<select>` のスタイリング | 有効化しているのは1エンジンのみで、どのJS代替も残りのエンジンを待つより悪い | プログレッシブエンハンスメントとして隔離 | すべてのエンジンが有効化して出荷した時 |
| 日付・時刻入力 | 全エンジンにあるが、ピッカーはスタイルできず、表示形式はロケールに縛られる | 制約付きで使用。基本的な日付入力を超えるなら隔離 | スタイリングのフックが日付入力に及ぶか、Temporal が全エンジンに届いた時 |
| File System Access | 単一ベンダーで標準化トラック外のため、規則が始まらない | 回避 — `<input type="file">`、blob ダウンロード、サーバーサイド | 2つ目のエンジンと標準化トラック |
| `<datalist>` | 全エンジンにあるが、フィルタリングと描画が意図的に未定義 | combobox には回避。些細な候補表示なら許容 | 仕様がフィルタリングとスタイリングのフックを定義した時 |
| `<input type="month/week">` | どこでも仕様化されているが、2つのエンジンが素のテキスト欄に劣化させる | 回避 — `<select>` に分解する | それらのエンジンがピッカーを出荷した時 |
| CSS anchor positioning | モジュールが改訂中のまま全エンジンが出荷した | 確定したサブセットを使い、`position-anchor` を明示的に宣言する | モジュールが確定した時 |
| Speculation Rules | 有効化して出荷しているのは1エンジン。もう1つは prefetch が設定の背後にあり、prerender は単一エンジン | 任意の強化に留め、決して前提にしない | 2つ目のエンジンが有効化して出荷した時 |
| Interest invokers（`interestfor`） | 1エンジンのみで、WebKit が設計への反対を表明している | 回避 | ベンダー間の合意 |
| CloseWatcher | 全エンジンにはなく、`<dialog>` と popover が既に必要を満たしている | 回避 | -- |

---

## 1. HTML Drag and Drop API

故障パターン: 構造的設計欠陥

### 成熟度評価

| 軸 | 評価 | 詳細 |
| --- | --- | --- |
| クロスプラットフォーム互換性 | 不合格 | マウスイベントに基づく仕様。ほとんどのモバイルブラウザは実装していない。タッチイベントの polyfill は API を実装するのではなく模倣する |
| 合成可能性 | 不合格 | ドロップを有効にするには `dragover` で `preventDefault()` が必要 — 非自明なイベントモデルの反転。`dataTransfer` は DOM 操作ではなく OS クリップボード向けに設計された MIME タイプのインターフェースを使う。ドラッグ画像のカスタマイズは大きく制限される |
| 障害モードの透明性 | 不合格 | モバイルでは何も起きない。エラーもフォールバックもなく、`draggable` 属性は無視される |
| 仕様の安定性 | 合格 | HTML5 以来 WHATWG Living Standard で安定 |

### CRZ の戦略: 回避（推奨）または隔離

- 管理画面の並べ替え可能なリスト: 明示的な並べ替えコントロール（上下移動ボタン、位置番号の入力）を提供する。よりアクセシブルで、どこでも動作する
- カンバン形式のボード: ドラッグ＆ドロップが必須要件なら、`@dnd-kit`（内部で Pointer Events を使用）を使う単一の island コンポーネントに隔離する。island は `onCardMove(cardId, fromColumn, toColumn, position)` を公開し、DnD API はこの境界の外に漏れない
- ファイルアップロードのドロップゾーン: 唯一の正当なユースケース。`dataTransfer.files` を使う `drop` イベントはデスクトップで確実に動作する。`<input type="file">` を主たる操作として提供し、ドロップゾーンをプログレッシブエンハンスメントとする island で包む

再評価の条件: モバイルファーストのセマンティクスを持つ、Pointer Events 上に再構築された将来の仕様改訂。

---

## 2. `<select>` のスタイリング（`appearance: base-select`）

故障パターン: 実装の遅れ

### 成熟度評価

| 軸 | 評価 | 詳細 |
| --- | --- | --- |
| クロスプラットフォーム互換性 | 不合格 | 有効化して出荷しているのは1エンジン。2つ目は設定の背後にあり、3つ目はプレビューリリースにある |
| 合成可能性 | 合格（条件付き） | サポートされていれば CSS と自然に統合でき、`<option>` 内のリッチな HTML コンテンツに対応し、標準の `::picker()` 疑似要素を使う |
| 障害モードの透明性 | 合格 | 非対応ブラウザは従来の `<select>` を描画する — 機能するがスタイルされない |
| 仕様の安定性 | 境界的 | WHATWG Stage 2。名称は何度も変わった（`<selectmenu>` -> `<selectlist>` -> `appearance: base-select`）。現在の API 表面は確定したように見える |

### CRZ の戦略: プログレッシブエンハンスメントとして隔離

- `appearance: base-select` は今日から使うが、適切に劣化する見た目のカスタマイズに限る
- 新しい CRZ プロジェクトでは JavaScript ベースの select 置き換え（Select2、React Select、Headless UI Listbox）を使わない
- 選択肢のリッチなコンテンツが必須要件で、今日の時点でクロスブラウザの一貫性が必須なら: サポートを機能検出し、ネイティブの `<select>` にフォールバックする単一の `<RichSelect>` island

再評価の条件: すべてのエンジンが有効化して出荷した時。

---

## 3. 日付・時刻の入力タイプ

故障パターン: 実装の遅れ（部分的）

### 成熟度評価

| 軸 | 評価 | 詳細 |
| --- | --- | --- |
| クロスプラットフォーム互換性 | 境界的 | 主要ブラウザはすべてこのタイプをサポートするが、UX は大きく異なる。Safari は空のとき現在日付をゴースト値として表示する |
| 合成可能性 | 不合格 | ピッカー UI は CSS でスタイルできない。Shadow DOM の構造はブラウザごとに異なる。表示形式はブラウザのロケールに従い、上書きできない。タイムゾーンのサポートがない |
| 障害モードの透明性 | 合格 | テキスト入力にフォールバックする。無効な日付は `:invalid` 疑似クラスを発生させる |
| 仕様の安定性 | 合格 | WHATWG Living Standard で安定 |

### CRZ の戦略: 制約付きで使用し、UX 要件がネイティブの能力を超えるなら隔離

- 基本的な日付入力（生年月日、開始日、期限）: ネイティブの `<input type="date">` を直接使う。UX の不一致は見た目の問題であり、挙動の問題ではない
- 高度な要件（日付範囲、タイムゾーン対応、独自カレンダー）: 専用の island に隔離する。Temporal API が Baseline に届くにつれて検討する
- 日付ピッカーを一から自作しない。アクセシビリティ、キーボード、ロケールの扱いは膨大である。プロジェクト所有のインターフェースで包んだライブラリを使う
- Safari の空値表示: CSS で回避するか、既知の見た目の問題として文書化する。`type="text"` に切り替えない

再評価の条件: `appearance: base-select` 形式のカスタマイズが日付入力に及んだ時。Temporal API が全エンジンに届いた時。

---

## 4. File System Access API

故障パターン: 実装の遅れ（深刻 — 単一ベンダー）

### 成熟度評価

| 軸 | 評価 | 詳細 |
| --- | --- | --- |
| クロスプラットフォーム互換性 | 不合格 | Chrome/Chromium のみ。Firefox と Safari は実装しておらず、懸念を表明している |
| 合成可能性 | 合格（サポート環境では） | Promise ベースで、ストリームと統合できる |
| 障害モードの透明性 | 合格 | 機能検出が明快 |
| 仕様の安定性 | 不合格 | WICG の提案。W3C の標準化トラックにない。ベンダー間の合意がない |

### CRZ の戦略: 回避

- 単一ベンダーの API はクロスプラットフォーム互換性の要件に反する
- ファイルの読み込み: `<input type="file">`
- ファイルの書き出し: Blob URL + download 属性
- 一括操作: BFF 経由でサーバーサイド

再評価の条件: 2つ目のエンジンの実装と、正式な標準化トラック。

---

## 5. `<datalist>` 要素

故障パターン: 仕様の不足

### 成熟度評価

| 軸 | 評価 | 詳細 |
| --- | --- | --- |
| クロスプラットフォーム互換性 | 不合格 | フィルタリングのアルゴリズムが異なる: Chrome は前方一致、Firefox は部分一致。`label` 属性の描画も異なる。選択肢を API で動的に絞り込むと、ブラウザがその上から独自のフィルタリングを再適用し、有効な選択肢を黙って隠す |
| 合成可能性 | 不合格 | スタイルできない。フォントサイズがページのズームを無視する。リッチなコンテンツを表示できない。`autocomplete` 属性との相互作用が一貫しない |
| 障害モードの透明性 | 合格 | 素のテキスト入力にフォールバックする — 候補がないだけで機能する |
| 仕様の安定性 | 境界的 | WHATWG Living Standard にあるが、描画とフィルタリングの挙動は意図的に未定義 |

### CRZ の戦略: 些細な候補表示を超える用途では回避。combobox パターンは隔離

- 単純で短い候補リスト（5〜15件の静的な選択肢、API の裏付けなし、ラベルと値の区別なし）: プログレッシブエンハンスメントとして許容
- API で裏付けたオートコンプリート: `<datalist>` を使わない。絞り込み済みの結果をブラウザが不透明に再フィルタリングするため、挙動が予測できない。ARIA パターンか Downshift で combobox の island を作る
- ラベルと値の区別（「Tokyo, Japan」を表示して「TYO」を送信する）: `<datalist>` を使わない。クロスブラウザの描画が一貫しない
- `<datalist>` は危険な中間地帯にある — combobox の用途で動きそうに見えるが、挙動が未定義なため、ユーザーに何が見えるかを予測できない

再評価の条件: WHATWG の仕様がフィルタリングの挙動、ラベルの描画、スタイリングのフックを定義するよう改訂された時。`appearance: base-select` 形式のカスタマイズが datalist に及んだ時も同様。

---

## 6. `<input type="month">`（および `type="week"`）

故障パターン: 実装の遅れ

### 成熟度評価

| 軸 | 評価 | 詳細 |
| --- | --- | --- |
| クロスプラットフォーム互換性 | 不合格 | Firefox とデスクトップ版 Safari はピッカーを提供せず、素のテキスト入力にフォールバックする |
| 合成可能性 | 不合格 | ピッカーがある環境でもスタイルできない。フォールバックのテキスト入力には形式の案内がない |
| 障害モードの透明性 | 境界的 | 手がかりのない `<input type="text">` に劣化する — 技術的には透明だが、体験としては不透明 |
| 仕様の安定性 | 合格 | WHATWG Living Standard で安定。問題は仕様ではなく実装にある |

### CRZ の戦略: 回避 — 信頼できるプリミティブに分解する

- 案A: ネイティブの `<select>` を2つ（年 + 月）。どこでも動き、アクセシブルで、JS 不要
- 案B: 月だけが意味を持つと案内した `<input type="date">`。値はサーバー側で切り詰める
- 案C（リッチな UI が必要な場合）: `onMonthSelect(year, month)` を公開する独自の月ピッカーを持つ island
- `<input type="week">` にも同じ分析が当てはまる

再評価の条件: Firefox と Safari がネイティブの月・週ピッカーを出荷した時。

---

## 7. 短い例外

| API | 規則が誤る理由 | CRZ の戦略 |
| --- | --- | --- |
| CSS anchor positioning | 2026-01 に全エンジンがコアを出荷したため規則は確定とみなすが、モジュールはまだ改訂中である — `position-anchor` の初期値は `normal` に落ち着くまでに3度変わったため、各エンジンの過去のリリースは現在のものと挙動が異なる | 確定したサブセットを使う。`position-anchor` は初期値に頼らず明示的に宣言し、下限より古いブラウザでも要素が使える位置に置かれるよう静的なフォールバック位置を宣言する。差を埋めるために JS の配置ライブラリを導入しない |
| Speculation Rules | MPA のナビゲーション向けの prefetch と prerender で、`<script type="speculationrules">` ブロックとして宣言する。有効化して出荷しているのは1エンジン。2つ目は prefetch が設定の背後にあり、prerender は1エンジンのみ | 任意の強化に留める。状態を変える URL（サインアウト、カート、言語切り替え）は除外し、`Sec-Purpose: prefetch` はサーバー側で扱う。体感速度をこれに依存させない |
| Interest invokers（`interestfor`） | ホバー、フォーカス、長押しで popover を起動する宣言的なトリガー。1エンジンで出荷され、WebKit が設計への反対を表明している | 回避。クリックで起動する `command` / `popovertarget` を使う。入力モダリティごとの対応物を必要としない |
| CloseWatcher | 独自 UI に対する Esc、Android の戻る操作、ジェスチャーによる閉じる操作を統一する。規則はいずれ採用を認める | 回避。`<dialog>` と popover は既に close request を受け取る。CloseWatcher が必要になるのは、たいていそのどちらかであるべき独自のオーバーレイである |

---

## 参考

- [Fifty problems with standard web APIs in 2025](https://zerotrickpony.com/articles/browser-bugs/)
- [DragDropTouch polyfill](https://github.com/drag-drop-touch-js/dragdroptouch) — DnD 仕様がマウスイベントに基づくことを記録している
- [Chrome Blog: A customizable select](https://developer.chrome.com/blog/a-customizable-select)
- [Open UI: Customizable Select Element Explainer](https://open-ui.org/components/customizableselect/)
- [MDN: `<input type="date">`](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/input/date)
- [WHATWG Issue #9986: `<datalist>` behavior inconsistencies](https://github.com/whatwg/html/issues/9986)
- [MDN browser-compat-data Issue #25723: datalist meta-issue](https://github.com/mdn/browser-compat-data/issues/25723)
- [MDN: `<input type="month">`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/month)
- [MDN: Speculation Rules API](https://developer.mozilla.org/en-US/docs/Web/API/Speculation_Rules_API)
