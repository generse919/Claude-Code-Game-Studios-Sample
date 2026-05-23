---
name: project-character-data-ac-review
description: character-data.md のAC批判的レビュー結果 — 4thパス完了、BLOCKING 8件、RECOMMENDED 12件、NICE-TO-HAVE 1件（2026-05-23実施）
metadata:
  type: project
---

character-data GDD（`design/gdd/character-data.md`）のAC全文を4回レビュー。Status: Revised (Pending Re-Review) — 4thパス完了。

**Why:** QA Leadとしてシフトレフトテストの観点でAC品質を評価。実装前に修正が必要なブロッカーを特定。

**3rdパスから解消されたブロッカー（4thパス時点）:**
- 前パスから解消されたものはなし。BLK-1〜4は全て未解消で4thパスに継続。

**4thパス BLOCKING問題 (8件) — Finding番号対応:**
1. Finding-1 (AC-1.1): "フレーム0"の定義が技フレーム基準（技入力=Frame0）とマッチフレーム基準で混在。LoadCompleteログのカテゴリ・レベル・フォーマット未定義。= 旧REC-1
2. Finding-3 (AC-1.3): gate levelがConfig/Data(ADVISORY)に誤分類。grep操作はCI gateとすべき。= 旧BLK-4の一部
3. Finding-4 (AC-2.2+AC-1.3): CI grepパターンが完全未定義。パターンなしのCI gateはto-doに過ぎない。= 旧BLK-1
4. Finding-5 (AC-3.3): C++フィールド非存在のネガティブテストはランタイムで実行不可。static_assertかコードレビューに変更必須。= 旧BLK-2
5. Finding-7 (AC-5.3): 制約コメントが論理逆転（"ActiveFrameEnd > ActiveFrameStart の制約違反"は実装者を逆方向に誘導する）。= 新規発見
6. Finding-8 (AC-6 note): either/or テスト方法でBLOCKINGゲートに非保護エスケープハッチが存在。= 旧BLK-3
7. Finding-21 (分類表): AC-1.1とAC-1.2がConfig/Data(ADVISORY)に誤分類。マッチライフサイクル統合テストはIntegration(BLOCKING)相当。= 旧BLK-4

**4thパス RECOMMENDED問題 (12件):**
- Finding-2 (AC-1.2): "全スカラーフィールド"が非網羅的、ラウンド終了コールバック名未定義
- Finding-6 (AC-4 B-3): ADVISORY/BLOCKINGの二重カバレッジの区別が未説明
- Finding-9 (AC-7): 下流GDDへのAC移管を強制するメカニズムがない
- Finding-10 (AC-5.2): ParryWindow=1,2が未テスト（ClampMin=3の境界値網羅不足）
- Finding-11 (AC-5 B-5): 非空かつMove.Range.*外のRangeTagが未テスト
- Finding-12 (AC-3.1/3.2): 計算関数シグネチャ未定義
- Finding-13 (AC-4 bullet3+4): WHEN句なし、B-3正常系ケースなし
- Finding-14 (E-2): OnBlockAdvantage>+5f警告のACなし
- Finding-15 (E-3): Hitboxes>4警告のACなし
- Finding-16 (E-4): HitboxTag未登録警告路のValid返却確認ACなし
- Finding-17 (E-5/B-9): KnockbackDistance=0+非正規化方向の正常系ACなし
- Finding-18 (D-3a): BaseDamage_effective按分式のテストカバレッジが全くない
- Finding-19 (Throw例外): Throw例外の正常系ACなし（両exemptionとも未カバー）

**4thパス NICE-TO-HAVE (1件):**
- Finding-20 (AC-6): 最小有効アセットの正常系baseline testなし

**How to apply:** 4thパスでBLK-1〜4は解消されずさらに4件の新規BLOCKINGが追加。次のGDD修正対応ではFinding-5(AC-5.3コメント逆転)とFinding-7(Finding-7=AC-5.3)を最優先。D-3a按分式のテスト欠落（Finding-18）は体力システムGDD設計前に確認する。
