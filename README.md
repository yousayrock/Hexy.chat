# kaleido.chat

**Discord風の見た目でMatrixを使う、動作が軽いクライアント。**

Rust（Leptos）で書かれた Web (PWA) の Matrix クライアントです。Electron を使いません。

---

## ⚠️ 現在の状態：設計段階

**コードはまだ1行も書かれていません。** このリポジトリにあるのは設計文書だけです。

次にやることは機能の実装ではなく、**技術的な生存確認（Spike 0）**です。
依存する matrix-rust-sdk のブラウザ（WASM）対応が[まだ追跡中の課題](https://github.com/matrix-org/matrix-rust-sdk/issues/35)であり、
これが動かなければ技術選定ごと見直す必要があるためです。詳細は [DESIGN §8](docs/DESIGN.md#8-spike-0--実装前の必須検証)。

---

## 何を目指しているか

Discord から Matrix へ移行したい人が、**操作を覚え直さずに、軽くて日本語で使えるクライアント**を手に入れること。

| # | 目指すこと | どう実現するか |
|---|---|---|
| 1 | 日本語対応 | 最初から日本語をベースに設計する |
| 2 | バグが少ない | v0.1 のスコープを極端に絞る |
| 3 | 非Electron | Web (PWA) |
| 4 | Rust製 | Leptos + matrix-rust-sdk |
| 5 | Discord に近い操作感 | 4ペイン構成と Discord の語彙 |

見た目は Discord のクローンではなく、[Commet](https://github.com/commetchat/commet) のような**簡素でクリーンな質感**を目指します。

---

## 技術構成（予定）

| 項目 | 選定 |
|---|---|
| 配布形態 | Web (PWA) のみ |
| UI | [Leptos](https://leptos.dev/) |
| 状態管理 | signal |
| ビルド | [Trunk](https://trunkrs.dev/)（CSR） |
| CSS | Tailwind CSS |
| Matrix SDK | [matrix-rust-sdk](https://github.com/matrix-org/matrix-rust-sdk)（wasm32） |
| 同期 | Sliding Sync（MSC4186） |
| 認証 | OIDC ＋ パスワード |
| ストレージ | IndexedDB |
| 接続先 | matrix.org（v0.1 では固定） |

---

## v0.1 でできること

- ログイン（パスワード / OIDC）
- チャンネル一覧の表示
- テキストメッセージの送受信
- 暗号化されたチャンネルの表示
- オフライン表示と再接続
- ログアウト

**ボイスチャットは v0.2 です。** v0.1 ではボイスチャンネルを表示だけして、通話は公式の [Element Call](https://call.element.io) へ外部リンクで逃がします（[VC_SPEC](docs/VC_SPEC.md)）。

---

## セキュリティについて正直に書いておくこと

kaleido.chat は Web アプリなので、セッションと暗号鍵は **IndexedDB** に保存されます。

**IndexedDB は OS のキーチェーン相当の保護を提供しません。** 同じオリジンで動く他のスクリプトや、端末を物理的に触れる第三者に対して、ネイティブアプリほど強くありません。

暗号化そのものは matrix-rust-sdk に任せ、独自の暗号実装は行いません。ただし、**「ネイティブアプリより安全」とは主張しません。**

---

## 開発環境のセットアップ

**Windows の場合。** Rust がまだ入っていない前提の手順です。

```powershell
# 1. C++ ビルドツール（MSVCリンカ。Rustのビルドに必須。数GB）
winget install --id Microsoft.VisualStudio.2022.BuildTools -e `
  --override "--quiet --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended"

# 2. Rust
winget install --id Rustlang.Rustup -e

# --- ここで一度シェルを開き直す（PATH反映のため） ---

# 3. WASMターゲット
rustup target add wasm32-unknown-unknown

# 4. Trunk
cargo install trunk
```

Tailwind CLI は、Node.js への依存を避けるため[スタンドアロンバイナリ](https://github.com/tailwindlabs/tailwindcss/releases)をプロジェクトルートへ置く方式を推奨します。

**必要な容量の目安**：VS Build Tools と Rust ツールチェーンで合計 5〜8GB 程度。

### 起動（Spike 0 以降）

```bash
trunk serve
```

---

## 文書

| 文書 | 内容 |
|---|---|
| [DESIGN](docs/DESIGN.md) | **設計の最上位。まずこれを読んでください** |
| [ROADMAP](docs/ROADMAP.md) | 段階的な開発計画と判断の記録 |
| [CORE_SPEC](docs/CORE_SPEC.md) | core層の公開API、データモデル、エラー、禁止事項 |
| [UX_SPEC](docs/UX_SPEC.md) | 画面ごとのUX規範、日本語文言、アクセシビリティ |
| [DESIGN_LANGUAGE](docs/DESIGN_LANGUAGE.md) | 色・フォント・角丸・コンポーネント規約 |
| [PERFORMANCE_BUDGET](docs/PERFORMANCE_BUDGET.md) | 「軽い」の測定項目と目標値 |
| [FEATURE_REFERENCE](docs/FEATURE_REFERENCE.md) | 機能の棚卸しと仕分け |
| [VC_SPEC](docs/VC_SPEC.md) | ボイスチャットの設計 |

---

## ライセンス

[MIT](LICENSE)

---

## 参考にしたプロジェクト

**構造は Discord、見た目は Commet、機能は Sable、実装は独自。**

| プロジェクト | 参考にしたもの |
|---|---|
| Discord | レイアウト構造、情報密度、語彙。**配色・フォント・ロゴは使っていません** |
| [Commet](https://github.com/commetchat/commet) | 見た目の方向性。AGPL-3.0 かつ Flutter 製のため、**コードは使えませんし使っていません** |
| [Sable](https://github.com/SableClient/Sable) / [Cinny](https://github.com/cinnyapp/cinny) | **機能一覧のみ。** AGPL-3.0 のため、コードは一切流用していません |
| [Element Call](https://github.com/element-hq/element-call) | ボイスチャットの連携先（v0.2） |

---

## ホスティング

将来的に `kaleido.chat` ドメインでの配信を想定しています（未取得）。
