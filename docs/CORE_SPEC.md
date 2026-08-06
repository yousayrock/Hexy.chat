# CORE SPEC

core層とSDKラッパ層の仕様。Matrix通信とアプリ内部処理を担当し、**画面描画は担当しません**。

**通信・同期・メモリ管理・暗号化は、すべて Rust 側でこの層が担います。**
UI層は表示と入力だけを扱い、性能に効く処理を持ちません。

---

## 1. 責務

core層とSDKラッパ層は以下を担当します。

- Matrix Client 初期化
- ログイン（パスワード / OIDC）
- セッション復元
- ログアウト
- ルーム一覧取得
- タイムライン取得
- メッセージ送信・受信
- 同期管理（Sliding Sync）
- 暗号化
- ローカル保存
- 再接続
- エラー変換
- イベント通知

**画面描画は担当しません。**

---

## 2. 技術構成

```text
Language:    Rust
Matrix SDK:  matrix-rust-sdk（wasm32-unknown-unknown）
同期:        Sliding Sync（MSC4186）
Storage:     IndexedDB（matrix-sdk-indexeddb）
Serialize:   serde
Error:       thiserror
Logging:     tracing
```

ライブラリは必要最小限にします。

---

## 3. ディレクトリ構成

```text
src/
├── ui/                 Leptos コンポーネント（この文書の対象外）
├── core/
│   ├── mod.rs
│   ├── models/         User / Room / Message / Session / AppEvent
│   ├── errors/         AppError
│   └── config/         AppConfig
└── matrix/
    ├── mod.rs
    ├── client.rs       初期化
    ├── auth.rs         ログイン・セッション
    ├── sync.rs         Sliding Sync
    ├── rooms.rs        ルーム一覧
    ├── messages.rs     送受信
    ├── encryption.rs   E2EE
    └── storage.rs      IndexedDB
```

---

## 4. 公開API

初期バージョンでは以下に限定します。APIを増やす場合は1機能ずつ追加します。

```rust
pub async fn initialize_app(config: AppConfig) -> Result<(), AppError>;

pub async fn login_with_password(
    username: String,
    password: String,
) -> Result<SessionInfo, AppError>;

pub async fn login_with_oidc() -> Result<SessionInfo, AppError>;

pub async fn restore_session() -> Result<SessionInfo, AppError>;

pub async fn logout() -> Result<(), AppError>;

pub async fn get_rooms() -> Result<Vec<RoomSummary>, AppError>;

pub async fn get_messages(
    room_id: String,
    limit: u32,
) -> Result<Vec<MessageItem>, AppError>;

pub async fn send_message(
    room_id: String,
    body: String,
) -> Result<MessageItem, AppError>;

pub async fn start_sync() -> Result<(), AppError>;

pub async fn stop_sync() -> Result<(), AppError>;
```

**Matrix SDK 固有型を公開APIへ直接出してはいけません。** 必ず下記の専用モデルへ変換します。

---

## 5. AppConfig

```rust
pub struct AppConfig {
    pub default_homeserver: String,  // v0.1 では "https://matrix.org" 固定
    pub log_level: String,
}
```

---

## 6. SessionInfo

```rust
pub struct SessionInfo {
    pub homeserver: String,
    pub user_id: String,
    pub device_id: String,
    pub display_name: Option<String>,
    pub avatar_url: Option<String>,
}
```

**access token を UI層へ返してはいけません。**

---

## 7. RoomSummary

```rust
pub struct RoomSummary {
    pub room_id: String,
    pub name: String,
    pub avatar_url: Option<String>,
    pub last_message: Option<String>,
    pub last_message_timestamp: Option<i64>,
    pub unread_count: u32,
    pub is_encrypted: bool,
    pub is_direct: bool,
}
```

---

## 8. MessageItem

```rust
pub struct MessageItem {
    pub event_id: String,
    pub transaction_id: Option<String>,
    pub room_id: String,
    pub sender_id: String,
    pub sender_name: String,
    pub sender_avatar_url: Option<String>,
    pub body: String,
    pub timestamp: i64,
    pub is_own: bool,
    pub status: MessageStatus,
}

pub enum MessageStatus {
    Sending,
    Sent,
    Failed,
}
```

Matrix では配達済み・既読の扱いが一般的なSNSと異なるため、v0.1 では過剰な状態表示をしません。

---

## 9. AppEvent

core層からUI層へ通知するイベント。ストリームとして渡します。

```rust
pub enum AppEvent {
    LoginSucceeded(SessionInfo),
    LoginFailed(AppError),
    SessionRestored(SessionInfo),
    SessionExpired,
    SyncStarted,
    SyncStopped,
    RoomListUpdated(Vec<RoomSummary>),
    MessageReceived(MessageItem),
    MessageUpdated(MessageItem),
    MessageSendFailed {
        transaction_id: String,
        error: AppError,
    },
    ConnectionLost,
    ConnectionRestored,
}
```

---

## 10. エラー設計

```rust
pub enum AppError {
    InvalidLogin,
    InvalidHomeserver,
    NetworkUnavailable,
    ServerUnavailable,
    SessionExpired,
    PermissionDenied,
    EncryptionError,
    StorageError,
    InvalidInput,
    NotInitialized,
    Unknown,
}
```

**Matrix SDK の内部エラーをそのままUI層へ返してはいけません。**
内部ログには詳細を残し、UIには安全で分かりやすいエラー種別だけを返します。

日本語文言への変換表は [UX_SPEC](UX_SPEC.md)。

---

## 11. ログイン処理

```text
入力検証
↓
Matrix Client 生成
↓
ログイン（パスワード または OIDC）
↓
暗号化初期化
↓
セッション保存
↓
ユーザー情報取得
↓
SessionInfo を返す
```

**パスワードは保存しません。**

---

## 12. セッション保存

保存対象:

- homeserver
- user ID
- device ID
- access token
- refresh token（存在する場合）

保存先は **IndexedDB** です。

> **Web版の保護レベルの限界。**
> IndexedDB は OSキーチェーン相当の保護を提供しません。同一オリジンのスクリプトや、端末を物理的に触れる第三者に対して、ネイティブアプリほど強くありません。
> **この限界を隠さず、README とアプリ内でユーザーへ明示します。**
> 鍵の生成・保護には Web Crypto API などブラウザネイティブの機能を使い、**独自の暗号実装は行いません**。

---

## 13. 同期管理

Sliding Sync を core層で一元管理します。

- 二重起動を防止する
- ログイン後に開始する
- ログアウト時に停止する
- ネットワーク切断時に再接続する（指数バックオフ）
- session expired を検出する
- ルーム更新をイベント通知する
- メッセージ受信をイベント通知する

**UIのライフサイクルだけで同期を停止しません。**

---

## 14. メッセージ送信

```text
空文字チェック
↓
transaction ID 生成
↓
Sending 状態をUIへ通知
↓
Matrixへ送信
↓
成功時に event ID 確定
↓
Sent 状態を通知
```

失敗時は transaction ID を保持し、手動再送に備えます。v0.1 では自動再送しません。

---

## 15. メッセージ受信

Matrix イベントからアプリ専用モデルへ変換します。

v0.1 で対応するのは**テキストメッセージのみ**です。
未対応イベントは無視せず、ログへ記録します。UIではクラッシュさせず、簡易表示または非表示にします。

---

## 16. 暗号化

**暗号化は matrix-rust-sdk に任せます。**

禁止事項:

- 独自暗号化
- 独自鍵交換
- 秘密鍵の平文保存
- access token のログ出力
- 復号済み内容の無制限ログ出力

暗号化未準備時は UI へ `EncryptionError` を返します。

**復号は常に有効です。** ユーザーが選べるのは「自分が作る新規ルームを暗号化するか」だけです（理由は [DESIGN](DESIGN.md) §4）。

---

## 17. ローカルデータ

v0.1 の保存対象:

- セッション
- SDKが必要とする暗号化データ（鍵）
- ルーム一覧キャッシュ
- メッセージキャッシュ
- 最終同期情報

独自DBを増やさず、可能な限り SDK の推奨保存方式（`matrix-sdk-indexeddb`）を使います。

---

## 18. ログ設計

```text
ERROR / WARN / INFO / DEBUG / TRACE
```

本番では INFO 以下を基本とします。

**ログへ出してはいけない情報:**

- パスワード
- access token
- refresh token
- 暗号鍵
- メッセージ本文の全文
- 個人情報

---

## 19. テスト

最低限、以下をテストします。

- 不正ログイン
- 正常ログイン
- セッション復元
- ログアウト
- 空メッセージ拒否
- メッセージモデル変換
- エラー変換
- 同期の二重起動防止
- `stop_sync`

実サーバーが必要な統合テストと、モック可能な単体テストを分けます。

---

## 20. 実装順序

```text
1. Cargoプロジェクト作成
2. AppConfig
3. AppError
4. Matrix Client 初期化
5. パスワードログイン
6. セッション保存（IndexedDB）
7. セッション復元
8. ルーム一覧（Sliding Sync）
9. メッセージ取得
10. メッセージ送信
11. 同期
12. 暗号化（復号）
13. OIDCログイン
```

**この順序に入る前に、[DESIGN](DESIGN.md) §8 の Spike 0 を必ず通してください。**

---

## 21. 完成条件（v0.1）

```text
Matrixへログインできる（パスワード / OIDC）
セッションをIndexedDBへ保存できる
リロード後にセッションを復元できる
Sliding Sync でルーム一覧を取得できる
メッセージ履歴を取得できる
テキストメッセージを送信できる
同期で新着メッセージを受信できる
暗号化ルームのメッセージを復号して表示できる
切断と再接続を表示できる
ログアウトできる
```
