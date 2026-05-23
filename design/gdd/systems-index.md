# Systems Index: CLASH

> **Status**: Draft
> **Created**: 2026-05-20
> **Last Updated**: 2026-05-20
> **Source Concept**: design/gdd/game-concept.md

---

## Overview

CLASH は間合い管理とパリィを核とした 1v1 ローカル格闘ゲーム（UE5.7 / PC Windows）。
全システムは「読み合いと間合いで勝敗を決める」という 30 秒コアループを支えるために存在する。
ゲームプレイの基盤となる入力・状態機械・ヒット判定を最優先で設計し、
その上にパリィ（逆転の切り札）・ラウンド判定・視覚フィードバックが積み重なる。
プリミティブジオメトリ一本のアートパイプラインにより、コンテンツ負荷は最小限。
設計の中心はシステムの精度（フレームパーフェクト判定）と読み合いの深度にある。

---

## Systems Enumeration

| # | システム名 | カテゴリ | 優先度 | ステータス | 設計ドキュメント | 依存先 |
|---|-----------|---------|--------|-----------|----------------|--------|
| 1 | キャラクターデータ定義 | Core | MVP | Pending Final Verification (4th pass revised) | [character-data.md](character-data.md) | なし |
| 2 | 入力システム | Core | MVP | Not Started | — | なし |
| 3 | キャラクター状態機械 | Core | MVP | Not Started | — | 入力システム、キャラクターデータ定義 |
| 4 | キャラクター移動システム | Gameplay | MVP | Not Started | — | 入力システム、キャラクター状態機械 |
| 5 | 体力システム | Gameplay | MVP | Not Started | — | キャラクターデータ定義 |
| 6 | ヒット判定システム | Gameplay | MVP | Not Started | — | キャラクターデータ定義、キャラクター状態機械 |
| 7 | 基本攻撃システム | Gameplay | MVP | Not Started | — | 入力システム、キャラクター状態機械、ヒット判定システム、体力システム |
| 8 | パリィシステム | Gameplay | MVP | Not Started | — | 入力システム、キャラクター状態機械、ヒット判定システム |
| 9 | 必殺技システム | Gameplay | MVP | Not Started | — | 入力システム、キャラクター状態機械、ヒット判定システム、体力システム |
| 10 | ラウンド・マッチシステム | Gameplay | MVP | Not Started | — | 体力システム、キャラクター状態機械 |
| 11 | 視覚フィードバックシステム | UI | MVP | Not Started | — | 基本攻撃システム、パリィシステム、キャラクター状態機械 |
| 12 | HUD システム | UI | MVP | Not Started | — | 体力システム、ラウンド・マッチシステム |
| 13 | マッチフロー UI | UI | MVP | Not Started | — | ラウンド・マッチシステム |
| 14 | 音響システム | Audio | MVP | Not Started | — | 基本攻撃システム、パリィシステム、必殺技システム、ラウンド・マッチシステム |
| 15 | コマンド入力シーケンス検出 | Core | Vertical Slice | Not Started | — | 入力システム |
| 16 | キャラクター選択画面 | UI | Vertical Slice | Not Started | — | キャラクターデータ定義、マッチフロー UI |
| 17 | 設定画面 | Meta | Alpha | Not Started | — | 入力システム |
| 18 | CPU AI システム | Gameplay | Full Vision | Not Started | — | 入力システム、キャラクター状態機械、ヒット判定システム、基本攻撃システム、パリィシステム |

---

## Categories

| カテゴリ | 説明 | 主なシステム |
|---------|-----|------------|
| **Core** | 全システムが依存する基盤 | 入力、キャラクター状態機械、コマンド検出、データ定義 |
| **Gameplay** | ゲームの面白さを作るシステム | 移動、戦闘、パリィ、体力、ラウンド判定、AI |
| **UI** | プレイヤー向け情報表示 | HUD、マッチフロー、視覚フィードバック、キャラ選択 |
| **Audio** | サウンド・音楽システム | SFX・BGM 統合管理 |
| **Meta** | コアループ外のシステム | 設定画面、キーコンフィグ |

---

## Priority Tiers

| ティア | 定義 | 目標マイルストーン | 設計の緊急度 |
|--------|-----|-----------------|------------|
| **MVP** | コアループが機能するために必須。「これは面白いか？」を検証できる最小構成 | 最初のプレイアブルプロトタイプ（〜2週間） | 最優先で設計 |
| **Vertical Slice** | 1つの完結した体験を示すために必要。コマンド技と簡易キャラ選択を追加 | VS/デモ（〜4週間） | 2番目に設計 |
| **Alpha** | 全機能が粗削りな状態で存在。設定画面・QoL 追加 | Alpha マイルストーン（〜6週間） | 3番目に設計 |
| **Full Vision** | ポリッシュ・CPU AI・コンテンツ完備 | Beta / リリース（〜8週間） | 必要に応じて設計 |

---

## Dependency Map

### Foundation Layer（依存なし）

1. **キャラクターデータ定義** — 技リスト・フレームデータ・ヒットボックス定義のデータアセット。コードより先に設計できる唯一の基盤
2. **入力システム** — 全プレイヤー操作の起点。フレーム精度要件（格闘ゲームの核）はここで決める

### Core Layer（Foundation に依存）

1. **キャラクター状態機械** — depends on: 入力システム、キャラクターデータ定義
2. **キャラクター移動システム** — depends on: 入力システム、キャラクター状態機械
3. **体力システム** — depends on: キャラクターデータ定義

### Feature Layer（Core に依存）

1. **ヒット判定システム** — depends on: キャラクターデータ定義、キャラクター状態機械
2. **基本攻撃システム** — depends on: 入力システム、キャラクター状態機械、ヒット判定システム、体力システム
3. **パリィシステム** — depends on: 入力システム、キャラクター状態機械、ヒット判定システム
4. **必殺技システム** — depends on: 入力システム、キャラクター状態機械、ヒット判定システム、体力システム
5. **ラウンド・マッチシステム** — depends on: 体力システム、キャラクター状態機械
6. **コマンド入力シーケンス検出** — depends on: 入力システム（VS+で必殺技システムに追加）

### Presentation Layer（Gameplay をラップ）

1. **視覚フィードバックシステム** — depends on: 基本攻撃システム、パリィシステム、キャラクター状態機械
2. **HUD システム** — depends on: 体力システム、ラウンド・マッチシステム
3. **マッチフロー UI** — depends on: ラウンド・マッチシステム
4. **音響システム** — depends on: 基本攻撃システム、パリィシステム、必殺技システム、ラウンド・マッチシステム

### Polish Layer（上記すべてに依存）

1. **キャラクター選択画面** — depends on: キャラクターデータ定義、マッチフロー UI
2. **設定画面** — depends on: 入力システム
3. **CPU AI システム** — depends on: 入力システム、キャラクター状態機械、ヒット判定、基本攻撃、パリィ

---

## Recommended Design Order

| 設計順 | システム | 優先度 | レイヤー | 担当エージェント | 見積もり工数 |
|--------|---------|--------|---------|----------------|------------|
| 1 | キャラクターデータ定義 | MVP | Foundation | game-designer / systems-designer | S |
| 2 | 入力システム | MVP | Foundation | game-designer / systems-designer | M |
| 3 | キャラクター状態機械 | MVP | Core | game-designer / systems-designer | M |
| 4 | キャラクター移動システム | MVP | Core | game-designer | S |
| 5 | 体力システム | MVP | Core | systems-designer | S |
| 6 | ヒット判定システム | MVP | Feature | game-designer / systems-designer | M |
| 7 | 基本攻撃システム | MVP | Feature | game-designer / systems-designer | M |
| 8 | パリィシステム | MVP | Feature | game-designer / systems-designer | L |
| 9 | 必殺技システム | MVP | Feature | game-designer | M |
| 10 | ラウンド・マッチシステム | MVP | Feature | game-designer | S |
| 11 | 視覚フィードバックシステム | MVP | Presentation | art-director / game-designer | M |
| 12 | HUD システム | MVP | Presentation | ue-umg-specialist / ux-designer | M |
| 13 | マッチフロー UI | MVP | Presentation | ue-umg-specialist / ux-designer | S |
| 14 | 音響システム | MVP | Presentation | audio-director / sound-designer | S |
| 15 | コマンド入力シーケンス検出 | VS | Feature | systems-designer / game-designer | M |
| 16 | キャラクター選択画面 | VS | Polish | ue-umg-specialist / ux-designer | S |
| 17 | 設定画面 | Alpha | Polish | ue-umg-specialist | S |
| 18 | CPU AI システム | Full Vision | Polish | ai-programmer / game-designer | L |

> 工数見積もり: S = 1セッション、M = 2〜3セッション、L = 4セッション以上

---

## Circular Dependencies

なし。依存グラフは完全な DAG（有向非巡回グラフ）。

---

## High-Risk Systems

| システム | リスク種別 | リスク内容 | 軽減策 |
|---------|-----------|-----------|--------|
| **キャラクター状態機械** | 設計リスク | 7システムのボトルネック。状態定義が不完全だと下流 GDD が矛盾を抱える | **設計順 #3 に配置**。承認前に全依存システムの状態参照が満たされているか確認 |
| **入力システム** | 技術リスク | UE5 の Enhanced Input がコマンド入力シーケンス（↓→P等）に非対応。フレームパーフェクト精度が必要 | MVP では 1 ボタン入力のみに限定。シーケンス検出は VS で自作設計 |
| **ヒット判定システム** | 技術リスク | 格闘ゲームに必要なフレーム精度と UE5 デフォルト設定（可変タイムステップ）の乖離 | 固定タイムステップ（60fps ロック）と Chaos Physics の衝突チャンネル設定を GDD で確定 |
| **パリィシステム** | 設計リスク | 受付ウィンドウ幅の調整が難しい——狭すぎると「使えない技」、広すぎると「全部パリィできる」 | GDD で 3〜8 フレームの設計範囲を定義し、プロトタイプで反復テスト（/prototype 推奨） |
| **CPU AI システム** | スコープリスク | ゲームジャム公開形式（オンライン vs. ローカル）次第で優先度が大きく変わる | Full Vision に据え置き。公開形式が確定してから着手判断 |

---

## Progress Tracker

| メトリクス | 数 |
|-----------|---|
| 識別されたシステム総数 | 18 |
| 設計ドキュメント着手済み | 1 |
| 設計ドキュメントレビュー済み | 0（Pending Final Verification: 1） |
| 設計ドキュメント承認済み | 0 |
| MVP システム設計完了 | 1 / 14 |
| Vertical Slice システム設計完了 | 0 / 2 |

---

## Next Steps

- [x] `/design-system character-data` — 設計順 #1: キャラクターデータ定義 GDD を作成（Designed — Pending Review）
- [ ] `/design-system input-system` — 設計順 #2: 入力システム GDD を作成
- [ ] `/map-systems next` — 常に設計順の次のシステムに進む
- [ ] `/design-review design/gdd/[system].md` — 各 GDD 完成後にレビュー
- [ ] `/gate-check pre-production` — 全 MVP GDD 完成後にプリプロダクションゲートチェック
- [ ] `/prototype parry` — パリィシステムは GDD 設計と並行して早期プロトタイプを推奨
