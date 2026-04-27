# CLAUDE.md — バカレシピ展応援ゲーム 引き継ぎ書

> このファイルは Claude Code（および他の AI コーディングアシスタント）が起動時に
> プロジェクト文脈を把握するためのガイドです。
> 一般的な技術情報やコードの再記載は避け、**この一つのプロジェクト固有の判断と
> 落とし穴**だけを書いています。詳細は `index.html` 内のコメントを直接読むこと。

---

## 1. プロジェクト概要

- **正式名称**: 野島慎一郎のバカレシピ展 応援ゲーム 〜缶バッジペア消し〜
- **目的**: ファンが運営する**非公式の応援コンテンツ**。実イベント（バカレシピ展 / 2026年4月29日〜5月3日 / 清瀬市郷土博物館）の集客と SNS 拡散を後押しする。
- **公開URL**: https://karupasu0920.github.io/baka-recipe-match-game/
- **デプロイ**: GitHub Pages（リポジトリ `karupasu0920/baka-recipe-match-game` の `main` ブランチ直下を静的配信）
- **対象端末**: iPhone Safari (iOS16+) / Android Chrome / PC 主要ブラウザ。**モバイル比率が高い**前提でレイアウト・タップ判定を最適化。

---

## 2. 技術スタック

| 項目 | 採用 | 補足 |
|---|---|---|
| 言語 | HTML / CSS / JavaScript | TypeScript なし。**ビルド工程なし** |
| 構成 | **単一ファイル `index.html`**（約6,700行） | CSS / JS は `<style>` `<script>` で内包。意図的にこの構成を維持 |
| バックエンド | Firebase Firestore | **compat SDK v10.14.0**（modular ではない）。スクリプトタグで読み込み |
| QRコード | [qrcodejs](https://github.com/davidshimjs/qrcodejs) v1.0.0 | CDN（jsdelivr）から ~16KB |
| ビルド | なし | `npm install` 等は不要。**ローカル確認は静的サーバーだけで OK**（README 参照） |
| パッケージマネージャ | なし | `package.json` を作らないこと |

**Firestore コレクション/ドキュメント**:
- `rankings` … 1ドキュメント = 1記録。`{ id, name, time, rank, device, createdAt }`
- `stats/totalChallenges` … `{ count }`。GAME START 押下で atomic increment。**doc 削除＝カウントリセット**仕様（旧パスへの自動フォールバックは意図的に削除済み）。

---

## 3. 主要機能一覧

| 機能 | 概要 |
|---|---|
| ペアマッチゲーム本体 | 22枚の缶バッジを45秒以内にペアで消す |
| コンボシステム | 3連以上でタイム短縮ボーナス。フルコンボ最大短縮2.35秒 |
| 称号判定 | タイムに応じた11段階の称号（README 参照） |
| シークレットバッジ | ひまわり🌻 をマッチで清瀬市特別演出 |
| ランキング | Firestore リアルタイム共有。TOP画面に8/5件、TOP100モーダル別途 |
| Xシェア | 3段階フォールバック（後述） |
| QRシェア | qrcodejs で `GAME_URL` の QR を生成。会場で対面シェア用 |
| 累計プレイ数表示 | `stats/totalChallenges` を 30秒キャッシュ越しに表示 |
| バカレシピ展ガイド | `ℹ️` ボタンモーダル |
| クレジット | `©️` ボタンモーダル（楽曲・画像・フォント・キャラの出典） |
| マャーちゃんダンス | ゲーム中に背景レイヤーで踊る（6×5 スプライト） |
| BGM / SE | 野島さん公式楽曲・音声。`Audio.playSE('click')` 等の API |
| 音設定 | BGM/SE をそれぞれ OFF/中/大の3段階。localStorage 保存 |

---

## 4. 重要な設計判断

ここに書いてあるルールを**勝手に変えると壊れる**。変える前に必ず動作確認。

### 4-1. 単一ファイル構成は維持
- `index.html` 一枚に全部入っているのは意図的。フォルダ分割やバンドラ導入は **ユーザーの明示的な依頼があるまで提案しないこと**。
- HTML / CSS / JS の各セクション内では、関連処理をまとめて IIFE で隔離する慣習がある。

### 4-2. Xシェアは「3段階フォールバック」
`openXShare(text, url)` が一手にまとめている。**個別の URL 遷移を直接書かないこと**。
1. クリップボードに先コピー（保険）
2. モバイル: `twitter://post` で X アプリ起動を試行（1.5秒の visibilitychange 監視）
3. 失敗 or PC: `window.open` で `x.com/intent/post` を**新タブ**で開く（必ず `noopener,noreferrer`）
4. それも失敗ならコピートースト表示

**同タブ遷移は禁止**（ゲーム本体が置き換わる）。

### 4-3. ランキング重複削除
- 一意キー = `name + rank(称号) + device`（`getEntryUniqueKey()`）
- 同じキーが複数あればベストタイムだけ残す（`deduplicateRankings()`）
- **ページングはユニーク件数基準**。「名前付き件数」基準ではない（v6 / 2026-04-27 で変更）。
- 連投で1ページが少数ユニークに潰れても、内部 fetch ループが追加で取りに行く（safety 3000件まで）。

### 4-4. 累計プレイ数（`stats/totalChallenges`）
- **doc 削除＝リセットを成立させる**ため、旧パスへのフォールバックは意図的に削除済み。
- 30秒キャッシュ。Firestore コスト削減用。
- 自分が押した直後はキャッシュを +1 して即時反映する仕掛けあり。

### 4-5. iOS Safari / レイアウト問題
- 100vh は使わず `--vh` カスタムプロパティ（`window.visualViewport.height` ベース）。
- bfcache 復元時の缶バッジ重なり対策として `pageshow.persisted` で再散布。
- `--mfp-h`（マッチパネル高さ）は **常に最大値で固定**。シークレット時に縮小しないこと（過去のガタつきの原因）。

### 4-6. プレイ動機・記録動線
- **「記録する」を押した人だけ Firestore に登録**される設計。
- 名前未入力で「記録する」を押すと `nameWarningModal` を表示。
- ランキング登録名は「`名無しのプレイヤー`」**および空文字列**を集計から除外（`isNamedEntry()`）。

### 4-7. UI ガード
- 右下の `ℹ️ / ©️` ボタンと左下の `💫 ゲームを紹介` は、**モーダル表示中とゲーム中は非表示**。
- 制御は `body[data-modal-open]`、`body[data-screen="game"]`、および `body:has(.overlay.active)` の3経路。`.overlay` クラスを使わない新モーダル（QR / シェア選択）は z-index 9000 で上に被せる戦略。

### 4-8. コメント慣習
- 大きめの変更には `// v6 (2026-04-27)：…` のように **バージョンタグ＋日付** を付けてある。
- 新しい修正を入れたら同様に `vN (YYYY-MM-DD)` で動機・トレードオフを残す（次の AI / 開発者のために）。

---

## 5. 既知の課題 / TODO

- [ ] **記録誘導**: 累計プレイ数 vs ランキング登録数のギャップ（プレイした人の大半が「記録する」を押さない）。記録ボタンの強調 / クリア後の自動誘導アニメは未実装（修正パターン2の案だけ用意してある）。
- [ ] **連投プレイヤーの占有**: v6で「ユニーク件数基準」のページングに変更したが、表示文言の改善余地あり。「これまでのチャレンジ X回 / ランキング登録 Y人」のような対比表記は未実装（修正パターン3の案）。
- [ ] **device 値の扱い**: `'pc' | 'mobile' | undefined` の3パターン。古いデータは `undefined`。重複削除キーが `'unknown'` で旧データ同士はまとまるが、PC/モバイル両方でプレイした同一人物は別行に残る（仕様）。
- [ ] **F12 診断ヘルパ `__debugRankings()`**: 開発・サポート用。本番では誤実行に注意（全件 get するので Firestore コストが増える）。
- [ ] **音量設定のセーブ場所**: localStorage。複数端末で同期しない（仕様）。

---

## 6. やってはいけないこと（重要）

### Git
- ❌ `git push --force` / `--force-with-lease`（main への force push は特に厳禁）
- ❌ `git reset --hard`、`git rebase -i`、`git filter-branch`（履歴改変系全般）
- ❌ `git commit --amend` を**push済みコミット**に対して行う
- ❌ `git config` の変更
- ❌ `--no-verify` でフックを飛ばす
- ❌ コミットなしの暴走（変更を都度コミットせず大量に積み上げない）
- ✅ コミットメッセージは英語タイトル + 詳細箇条書き、過去の慣習に合わせる（直近: `Add QR share modal and improve ranking dedup/pagination` のスタイル）

### コード
- ❌ `index.html` を分割しない（明示依頼があるまで）
- ❌ TypeScript / バンドラ / npm の導入提案
- ❌ Firebase modular SDK への置き換え（compat 前提のコードが多い）
- ❌ `openXShare` を経由しないアドホックな X URL 遷移
- ❌ `--mfp-h` をシークレット時だけ縮小するような変動
- ❌ 同タブ遷移（`location.href = '...'` で X / 外部に飛ばす）
- ❌ `stats/totalChallenges` の旧パス（`stats/global.totalPlays` 等）への自動フォールバック復活
- ❌ Audio API の直接呼び出し（必ず `Audio.playSE(...)` / `Audio.startBGM(...)` 経由）

### Firestore
- ❌ セキュリティルールを緩めない（コミット前に必ず本人確認）
- ❌ `rankings` を一括 delete するスクリプトを安易に走らせない（ベストタイム履歴が消える）
- ✅ `stats/totalChallenges` の `count` を意図的にリセットしたいときは Firebase Console で doc を削除する。コードは「無ければ 0 」として正しく振る舞う

### アセット
- ❌ 楽曲・効果音・画像の差し替え（**野島さん本人許諾済み**のものだけ使用）。新規追加するときは出典・許諾を必ず確認
- ❌ `assets/badges/` の缶バッジ画像のリネーム（`menu-list.json` の `badgeId` と紐づく）

---

## 7. ファイル構成

詳細は `README.md` に記載。要点だけ:

```
my-app/
├── index.html              # 全部入り（HTML/CSS/JS、約6,700行）
├── README.md               # ユーザー向け説明
├── CLAUDE.md               # ← このファイル
├── .gitignore
├── assets/
│   ├── background*.png     # PC / モバイル背景
│   ├── Title.png, bakaten-banner.png, start-banner.svg
│   ├── audio/              # BGM (mp3)
│   ├── badges/             # 缶バッジ11種
│   ├── dance/maya-sprite.png  # 6×5 スプライト
│   └── menu/               # 料理画像 + menu-list.json
└── tunes/                  # 効果音 (mp3)
```

`menu-list.json` は `index.html` の `loadMenuData()` から fetch される。**badgeId / name / url / isSecret** が主要フィールド。シークレットは `isSecret: true`。

---

## 8. 重要な過去の修正履歴

| コミット | 内容 |
|---|---|
| `404735d` | 初期コミット |
| `c508940` | モバイルページング修正（リスト＆「もっと見る」を共通スクロール領域に。iOS Safari の sticky フッタ対策） |
| `8c222f8` | ランキング・ページング改善（先読み済みデータが cap で隠れる問題）。`hasNextRankingPage` 追加 |
| `8dfa2f3` | **QRシェアモーダル追加** + **ランキング重複削除（name+rank+device）** + **ページング基準をユニーク件数へ刷新** + 診断ヘルパ `__debugRankings()` 追加 |

### 設計の流れ（時系列）
1. 当初は localStorage のみのランキング → Firestore 化
2. Xシェアは普通の `<a href="https://twitter.com/intent/...">` だった → アプリ起動・タブ遷移・コピーの3段階フォールバックに統合（v4 / 2026-04）
3. `stats/global.totalPlays` から `stats/totalChallenges` へ移行（doc 削除＝リセットを意図的に成立させるため、旧パスフォールバックは削除）
4. 名前付き100件ベースのページング → ユニーク100件ベース（v6 / 2026-04-27）
5. シェア導線を「Xに飛ばす」だけだった → QR選択モーダル経由に（v5 / 2026-04-27）

---

## 9. 開発者へのお願い（AI 含む）

- 大きな変更を加える前に**動作仮説を1〜2文で書く**こと（コミットメッセージ詳細欄でも、コードコメントの `vN (YYYY-MM-DD)` でも OK）。
- iOS Safari / Android Chrome / PC の**3経路で目視確認**してから push するのが理想。最低でも仕様の影響範囲を明記。
- このファイル（CLAUDE.md）も**仕様や禁止事項が変わったら更新**すること。とくに「やってはいけないこと」のリストは口伝にせず、ここに明文化する。
