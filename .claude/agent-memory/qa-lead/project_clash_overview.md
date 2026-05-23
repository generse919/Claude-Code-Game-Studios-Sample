---
name: project-clash-overview
description: CLASH — UE5.7製ローカル1v1格闘ゲームの基本スタック・QA関連制約
metadata:
  type: project
---

CLASHはUE5.7（C++主体）で開発するローカル2プレイヤー格闘ゲーム（PC Windows専用）。

**Why:** ゲームスタジオの48エージェント構成プロジェクト。QAは全システムの受け入れ基準とテスト証跡を管理する。

**How to apply:**
- テストフレームワークはUnreal Automation System（FAutomationTestBase）
- ユニットテストは `tests/unit/[system]/` に配置
- 統合テストは `tests/integration/[system]/` に配置
- FName禁止（FGameplayTag必須）— `coding-standards.md` の Forbidden Patterns に記載済み
- 60fps固定タイムステップ（OQ-1未解決）
- ヘッドレステスト環境では `-nullrhi` フラグを使用
