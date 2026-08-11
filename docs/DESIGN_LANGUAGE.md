# DESIGN LANGUAGE

Hexy.chat の見た目を決める規約。色・フォント・形状・コンポーネントの作法。

## 参照元の役割分担

| | 参照元 |
|---|---|
| **レイアウト構造**（4ペイン、情報密度、ホバー挙動） | Discord |
| **見た目の質感**（簡素・クリーン・ツールらしさ） | [Commet](https://github.com/commetchat/commet) |
| 配色・フォント・ロゴ | **独自** |

Commet は Discord風UIを持つ Matrix クライアントで、「機能は豊富だがインターフェースは簡素」を掲げています。派手さより整理された佇まいを取る、という方向がここの基準です。

> **どちらのコードも使いません。** Commet は AGPL-3.0 かつ Flutter 製で、ライセンス上も技術上も Leptos へ持ち込めません。Discord は商用クローズドです。
> 参照するのは**画面を見て分かること**だけです。

## 原則

1. **ダークオンリー。** ライトモードは実装しない
2. **丸を使わない。** スクエア基調の角丸で統一する
3. 情報密度は高く。ただし読めなくなるほど詰めない
4. **飾りを置かない。** 機能のないボタンやアイコンを配置しない

> **なぜ Discord の色を使わないか。**
> `#5865f2`（blurple）などの配色、`gg sans` / `Whitney` といったフォントは Discord のブランド資産です。
> Hexy.chat は MIT で一般公開するため、これらをそのまま使うとトレードドレスの問題になり得ます。
> **構造は真似てよく、見た目の記号は独自にします。**

---

## カラートークン — `midnight`

背面から前面へ向かって段階的に明るくすることで、ペインの階層を表現します。真っ黒（`#000000`）は使わず、わずかに青みを含んだミッドナイトトーンで統一します。

| トークン | 値 | 用途 |
|---|---|---|
| `midnight.server` | `#0f1115` | 最背面。スペース（サーバー）バー |
| `midnight.sidebar` | `#14171e` | 中間面。チャンネル一覧、メンバー一覧 |
| `midnight.chat` | `#1a1d24` | 最前面。タイムライン背景 |
| `midnight.input` | `#222630` | 入力フォームの背景 |
| `midnight.accent.DEFAULT` | `#6366f1` | アクセント（Tailwind標準 indigo-500） |
| `midnight.accent.hover` | `#4f46e5` | アクセントのホバー（indigo-600） |
| `midnight.accent.light` | `rgba(99, 102, 241, 0.15)` | 選択中の背景、淡いハイライト |
| `midnight.text.normal` | `#f1f5f9` | 本文（slate-100） |
| `midnight.text.muted` | `#94a3b8` | チャンネル名、タイムスタンプ（slate-400） |

**アクセントに indigo-500 を選んだ理由**：Tailwind 標準色なので商標上完全に安全であり、かつ「青っぽくて格好いい」という求める印象を満たすため。

### 状態色（v0.1 で必要な最小限）

| 用途 | 値 |
|---|---|
| 成功・オンライン | `#22c55e`（green-500） |
| 警告 | `#f59e0b`（amber-500） |
| エラー・送信失敗 | `#ef4444`（red-500） |

---

## フォント

**商標フォントは一切使いません。** 全体を等幅フォントで統一することで、「洗練された開発者向けツール」の印象を出し、既存チャットアプリの模倣感を消します。

```text
JetBrains Mono → Fira Code → ui-monospace → SFMono-Regular → Consolas → Noto Sans JP → monospace
```

日本語は `Noto Sans JP` にフォールバックします。等幅フォントの多くは日本語グリフを持たないため、この順序が重要です。

---

## 形状

| トークン | 値 | 用途 |
|---|---|---|
| `borderRadius.app-icon` | `6px` | アイコン、アバター、ボタン |
| 標準の角丸 | `4px` 〜 `8px` | カード、入力欄 |

**`rounded-full` を使いません。** アバターも含めてすべてスクエア基調にします。これだけで既存クライアントの模倣感が消えます。

---

## Tailwind 構成

Leptos（Rust/WASM）では JavaScript 環境のような自動クラス検出が効きません。**Tailwind CLI に Rust ソースを直接スキャンさせる**設定が必要です。

### `tailwind.config.js`

```javascript
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    // 最重要：すべてのRustソースをスキャン対象にする
    "./src/**/*.rs",
    "./index.html",
  ],
  theme: {
    extend: {
      colors: {
        midnight: {
          server:  '#0f1115',
          sidebar: '#14171e',
          chat:    '#1a1d24',
          input:   '#222630',
          accent: {
            DEFAULT: '#6366f1',
            hover:   '#4f46e5',
            light:   'rgba(99, 102, 241, 0.15)',
          },
          text: {
            normal: '#f1f5f9',
            muted:  '#94a3b8',
          },
        },
      },
      fontFamily: {
        app: ['JetBrains Mono', 'Fira Code', 'ui-monospace', 'SFMono-Regular',
              'Consolas', 'Noto Sans JP', 'monospace'],
      },
      borderRadius: {
        'app-icon': '6px',
      },
    },
  },
  plugins: [],
}
```

### `Trunk.toml`

Trunk の `[[hooks]]` を使い、ビルドの前段で Tailwind CLI を走らせます。

```toml
[build]
target = "index.html"

[[hooks]]
stage = "build"
command = "npx"
command_arguments = [
  "tailwindcss",
  "-i", "./style/input.css",
  "-o", "./style/output.css",
  "--minify"
]
```

**Node.js を入れたくない場合**：Tailwind 公式のスタンドアロン単体実行バイナリをプロジェクトルートへ置き、`command = "./tailwindcss"` にすれば npm なしで動きます。依存を減らせるのでこちらを優先します。

### `style/input.css`

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* カスタムスクロールバー */
::-webkit-scrollbar { width: 8px; }
::-webkit-scrollbar-track { background: #14171e; }
::-webkit-scrollbar-thumb { background: #222630; border-radius: 4px; }
```

### `index.html`

```html
<link data-trunk rel="css" href="style/output.css" />
```

`trunk serve` を叩けば、Rust の保存 → Tailwind 再生成 → WASM コンパイル → ホットリロードが自動で回ります。

> **Spike 0 での確認事項**：この構成が実際に動くこと、特に `[[hooks]]` の実行順序が Trunk のビルドより先であることを確認します。

---

## レスポンシブ実装

ブレークポイントごとの表示ペイン数は [DESIGN](DESIGN.md) §5 を参照。実装上の要点:

- ドロワーは `fixed` + `translate-x` のトランジションで出し入れし、`md:static md:translate-x-0` でデスクトップでは常設に戻す
- ドロワー開閉は signal で管理する
- バックドロップは左右どちらかが開いているときだけ描画し、クリックで両方閉じる
- **各ペインは個別に `overflow-y-auto`、`body` は `overflow-hidden`**

---

## コンポーネント規約

### メッセージ行（v0.1）

Discord 同様、吹き出しで囲わず**縦に流れるストリーム形式**にします。

- 行全体を `group` にし、ホバーで背景をわずかに明るくする
- 左にアバター、右に「送信者名 ＋ タイムスタンプ」→「本文」
- 本文は `whitespace-pre-wrap break-all`
- 自分のメッセージは**色だけでなく配置や記号でも区別する**（[UX_SPEC](UX_SPEC.md) のアクセシビリティ要件）

### アバター（v0.1）

- スクエア基調（`rounded-app-icon`）。**丸にしない**
- 画像が無い場合は表示名の頭文字を `midnight.accent` 背景で描画

### 入力欄（v0.1）

- 画面最下部に貼り付けず、周囲に余白を取って浮かせる
- `midnight.input` 背景
- v0.1 では**添付ボタン・絵文字ピッカー・GIFボタンを置きません**（機能がないため。飾りのボタンは出さない）

### ホバーアクションバー（v0.2）

返信・編集・削除・リアクションは v0.1 のスコープ外です。**v0.1 ではホバーしても何も出しません。**

### リアクション（v0.2）

v0.1 では描画しません。

---

## タイムラインのスクロール

- 新着受信時、**ユーザーが最下部付近にいるときだけ**自動で追従する
- 過去ログを読んでいる最中に勝手に飛ばさない
- Leptos の `NodeRef` で DOM 参照を取り、`scroll_top = scroll_height` で追従させる
