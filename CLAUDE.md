# CLAUDE.md — 玉子湯旅館 スタッフアプリ

このファイルはClaudeがセッション開始時に自動で読む設計メモです。
作業前に必ず目を通してください。

---

## プロジェクト概要

- **リポジトリ**: https://github.com/TMA-BOT/tamagoyu
- **公開URL**: https://tma-bot.github.io/tamagoyu/
- **メインファイル**: `index.html`（約9800行超のシングルファイルアプリ）
- **ホスティング**: GitHub Pages（mainブランチへのpushで自動反映）
- **バックエンド**: Firebase Realtime Database + Firebase Authentication

## スタッフ構成・権限

- `front`（フロント）
- `ryoin`（料飲スタッフ）
- `ryoin_lead`（料飲リーダー）
- `seiso`（清掃）
- `admin`（管理者）

---

## 主な機能一覧

### フロント系
- チェックイン・チェックアウト管理
- 部屋状況トラッキング

### 料飲系
- テーブル管理（ドリンク注文・サイン収集）
- 配膳トラッカー（個人別・一括チェック対応のカウンター方式）
- **ランチ管理**（後述）

### シフト管理系
- フロントスタッフのシフト表作成・編集
- 入込数（宿泊人数）との連携・人員不足警告
- 予約CSVからの入込数自動取込
- 公休カウンター（9日制）
- **別タブ表示・印刷**機能（`openShiftNewTab()` / `_buildShiftStandalonePage()`）
- メンバー管理（追加・インライン編集・並び替え・削除）

### 清掃系
- 清掃状況管理

### その他
- `number_counter.html`：車のナンバープレート地域集計ツール

---

## ランチ管理の詳細（重要）

### frontStatusの3値
ランチの支払い状態は `frontStatus` フィールドで管理：
- `'paid'` — ランチ代 会計済
- `'unpaid'` — ランチ代 未会計
- `'room'` — 部屋付け（チェックアウト時に精算）

### 部屋付けの判定ロジック（重要・バグ修正済み）
```javascript
// 現在の正しい判定（全箇所統一）
const fs = lu.frontStatus || (lu.roomNo ? 'room' : (lu.frontPaid!==false ? 'paid' : 'unpaid'));
```

**なぜこの形か：**
古いデータ（`frontStatus`フィールド保存前）は `frontStatus` が `undefined`。
その場合でも `roomNo`（部屋番号）が設定されていれば部屋付けとみなす。
さらに `roomNo` もなければ `frontPaid` で判定する3段フォールバック。

### ランチ集計パネルの構成
1. **通常集計**（部屋付けを除外）
   - 総人数・総売上・伝票数・会計済件数・メニュー別数量
   - 合計表示に「（🏠部屋付け除く）」の注記
2. **🏠 部屋付け一覧セクション**（青枠ボックス）
   - 部屋番号・メニュー内容・金額を一覧表示
   - 部屋付けが0件の場合は非表示

### 関連コード
- `buildLunchList()` 関数 — index.html 約3410行〜
- セーブコード — index.html 約4034行（`frontStatus:frontStatus` で保存）

---

## シフト管理の詳細（重要）

### データ構造
- `SHIFT_MEMBERS`：`[{id, name, role, kyukei}]`
  - `kyukei`：公休9日カウント対象かどうか（boolean）
- `SHIFT_DATA`：`{year_month: {memberId: {day: code}}}` 例：`{'2025_6': {'m1': {1:'早', 2:'休'}}}`
- `SHIFT_OCUPANCY`：`{year_month: {day: {pax:n, manual:bool}}}`
  - `manual:true` のとき手動上書き、`false` は Firebase dates からの自動取得値

### 入込数の取得ロジック
1. Firebase `dates/YYYY-MM-DD` から宿泊人数を自動集計（`_loadOcupancyFromDates()`）
2. 手動上書き（`manual:true`）がある日はスキップ
3. 0に戻したら自動取得に戻る設計

### 別タブ表示の注意点
`openShiftNewTab()` はモーダルのHTMLとは独立して `_buildShiftStandalonePage()` でHTMLを生成。
モーダル側のレイアウト変更は**別タブ側に自動反映されない**ため、同じ修正を両方に加える必要がある。

### 勤務コード凡例の配置
- モーダル内：シフト表の**上**（`shift-code-legend` div）
- 別タブ：公休カウンター → 凡例 → シフト表 の順（`_buildShiftStandalonePage()` 内）

### 関連コード位置（目安）
- `_renderShift()` — シフト表のメインレンダリング
- `_renderMemberList()` — メンバー管理UI（インライン編集・並び替え対応）
- `openShiftNewTab()` / `_buildShiftStandalonePage()` — 約9400行〜
- `_loadShiftData()` / `_loadOcupancyFromDates()` — 約9485行〜

---

## Firebase構成

- `window._db` — Firebaseデータベースインスタンス
- `window._currentUser` — 現在のログインユーザー
- `_dbGet(path)` — データ取得ヘルパー
- `_dbRef(path)` — refヘルパー

### 重要な知見（過去のバグから）
- **saveタイムアウト**: base64画像データが大きいため `_saving` タイムアウトは3000ms以上に設定
- **JPEG圧縮**: 白背景fill + JPEG圧縮でサイズ削減
- **localStorageバックアップ**: Firebase保存失敗時の保険
- **manualAssign**: 全save/load関数に必ず含める（過去に抜けて無言バグが発生）
- **Airpay/Airレジ API非公開**: CSVエクスポート/インポートによるファイルベース連携のみ可能

---

## モジュールスコープの注意

`LUNCHES`・`SHIFT_MEMBERS` などの変数はモジュールスコープ（`window.LUNCHES` とは別物）。
ブラウザコンソールでテストする際は `eval()` で書き換えが必要。

---

## コミット履歴（主要）

| コミット | 内容 |
|---------|------|
| fac63c4 | fix: 別タブ表示の勤務コード凡例をシフト表の上に移動 |
| 87b81f8 | fix: 勤務コード凡例をシフト表の上に移動（モーダル側） |
| b8afa7c | feat: メンバー管理にインライン編集・並び替え機能を追加 |
| cf735da | fix: シフトモーダルの凡例・出勤人数行の重なりを修正（sticky bottom解除） |
| 1cf50eb | feat: シフト表に3パターンの注意喚起を追加 |
| 93eac7e | fix: シフト入込数行を日付ヘッダーの直下（thead2行目）に移動 |
| c61246e | feat: ホームのゲスト変更をシフト入込数に自動反映 |
| 01f3466 | feat: シフト管理に予約CSV取込機能を追加 |
| 92bc309 | feat: シフト表に入込数連携と人員不足警告を追加 |
| 698b38b | feat: 祝日データを外部APIから自動取得する仕組みに変更 |
| b438554 | fix: 部屋付けランチの判定ロジックを修正（roomNoフォールバック追加） |
| 36653e5 | ランチ集計：通常集計から部屋付けを除外 |
| 1783bb7 | ランチ集計に部屋付け一覧セクションを追加 |
| c44f932 | アプリ最新版を反映（配膳カウンター・ドリンク注文・メニュー並び替え等） |
| b6e7de5 | 初回アップロード |

---

## 将来の課題・未実装

- **売上照合ツール**: フロントレジ × Airレジ（iPad）の日次照合シート
  - Airメイト CSVのサンプルが入手できたら着手
  - `index.html` の新タブ or 別HTMLファイルで検討中
  - FirebaseにCSVをインポートする設計、重複防止・欠損日検出も予定
- **ツアー部屋割り機能**: 予約班との調整待ち
  - 現状はCSVで一括取り込み→グループ分割で対応
  - 部屋割りが事前確定かどうか未確認

---

## 開発Tips

- GitHub Pagesへの反映は push 後 数分かかる
- テスト時はブラウザの開発者ツールのコンソールを活用
- Firebase権限エラーが出る場合は管理者ログインが必要
- index.html は9800行超のため、編集前に `grep -n` で対象行を特定してから作業する
