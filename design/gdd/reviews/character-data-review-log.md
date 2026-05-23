# Review Log: キャラクターデータ定義

> **Source GDD**: `design/gdd/character-data.md`
> **System**: キャラクターデータ定義（設計順 #1 / Foundation Layer）

---

## Review — 2026-05-20 — Verdict: MAJOR REVISION NEEDED

**Scope signal**: L
**Specialists**: game-designer, systems-designer, qa-lead, unreal-specialist, creative-director (senior)
**Blocking items**: 10 | **Recommended**: 8 | **Nice-to-have**: 3
**Prior verdict resolved**: N/A — 初回レビュー

**Summary (creative-director)**:
Foundation Layer スキーマとして下流8システムすべての前提となるGDDだが、スキーマの構造的欠落が3件（`ActiveFrameStart/End` のゼロ点未定義・`UCharacterDataAsset.MoveList` フィールド未宣言・`FCharacterStats.PrimaryColor` 未定義）と、バリデーション不備2件（`HitstunFrames` 不変条件のロード時未検証・ネストソフト参照の二次ロード戦略未定義）が発見された。複数スペシャリストが独立に確認した矛盾が存在するため MAJOR REVISION NEEDED と判定。受け入れ基準（AC）も全体的に再設計が必要（10件のブロッカー中3件がAC品質問題）。ゲームジャムスコープ考慮で ParryWindow の粒度はキャラ単位維持を裁定したが、FMoveData にオーバーライドフィールド追加を推奨。

**Top Blockers (要修正)**:
1. `FHitboxData.ActiveFrameStart/End` のゼロ点（フレーム基準点）未定義
2. `UCharacterDataAsset.MoveList: TArray<FMoveData>` がスキーマ未宣言
3. `FCharacterStats.PrimaryColor` が UI Requirements で参照されているが未定義
4. 攻撃の段（`EMoveLevel` High/Mid/Low/Overhead）がスキーマに不在
5. `HitstunFrames > BlockstunFrames` 不変条件が `Validate()` に未追加
6. `TSoftObjectPtr` ネスト参照の二次ロード戦略未定義
7. D-3 `BaseDamage` スキーマ下限（30）と D-3 計算基準（50）の不一致 + MaxHealth変動時KO計算式なし
8. AC-7 が未構築依存システムに依存（条件付き BLOCKING へ再分類必要）
9. AC-5 が ClampMin 実装詳細を検証（Validate() 戻り値ベースに書き直し必要）
10. AC-1「差分ゼロ」比較が曖昧で実行不能

**Revision applied (2026-05-22)**: 全10件のブロッカーを同セッション内で解決。Status → `Revised (Pending Re-Review)`。次回 /design-review でクリーンな判定を推奨。

---

## Review — 2026-05-22 — Verdict: MAJOR REVISION NEEDED（2nd pass）

**Scope signal**: L  
**Specialists**: game-designer, systems-designer, qa-lead, unreal-specialist, creative-director (senior)  
**Blocking items**: 9 | **Recommended**: 7  
**Prior verdict resolved**: Partial — 前回10件は解消されたが、新規9件が発見された

**Summary (creative-director)**:
前回の MAJOR REVISION NEEDED から明確な改善はあったものの、より根深い問題が露呈した。Overhead 定義の業界標準逆転（B-1）、OnHitAdvantage 負値の設計原則未実装（B-3）、パリィ三択のリスク > リターン（B-4）の3件は Pillar 2/3/4 を直接損なう。Foundation Layer としてのスキーマ権威の二重定義（B-2: MoveLevel vs CanHitLow/CanHitHigh）と Pillar 1 間合いフックの欠如（B-5）は後続 GDD 全体に波及する。UE5 技術不適合（B-6: PostEditChangeProperty 誤用、B-7: Asset Manager 必須実装未記載、B-8: GameplayTags モジュール設定未記載）および B-9（KnockbackDirection Y軸未定義）も BLOCKING。

**Revision applied (2026-05-22)**: 全9件のブロッカーを同セッション内で解決。設計決定: CanHitLow/CanHitHigh 削除 → MoveLevel 単一権威、ParryCounterAdvantage 下限 12f（案α）。Status → `Revised (Pending Re-Review)`（2nd pass）。

---

## Review — 2026-05-23 — Verdict: MAJOR REVISION NEEDED（3rd pass）
Scope signal: XL
Specialists: game-designer, systems-designer, unreal-specialist, creative-director（senior）[qa-lead: セッション限界によりタイムアウト — AC品質の独立検証欠落]
Blocking items: 12 | Recommended: 8
Summary: ゲームデザイン面では投げの欠落（三すくみ未完成・Pillar 2/4破綻）、High/Mid挙動の無定義、HitstunFrames設計ガイダンスと形式制約の矛盾（特殊技では制約を満たせない値域問題）、ParryCounterAdvantage 12f保証のオフバイワンが発見された。システム設計面ではD-3スケーリング式の適用主体未定義とセッション長保証の脆弱性。UE5実装面では`IsDataValid()`がパッケージビルドで実行されないアーキテクチャ欠陥（製品ビルドでの不正データサイレントロード）、`AddBundleAsset()` API変更（UE5.2+: FTopLevelAssetPath）、ClampMinの誤記がBLOCKING。スタル参照（CanHitLow/CanHitHigh）が3回目も残存していたが削除。
Prior verdict resolved: Partial — 前回9件は解消されたが、新規12件が発見された

**Revision applied (2026-05-23)**: 全12件のブロッカーを同セッション内で解決。設計決定: ①EMoveCategory.Throw追加（VS+実装予約、BlockstunFrames=0等の特例を文書化）、②High=しゃがみ自動回避（業界標準）、③D-3スケーリング=体力システムがランタイム自動適用、④ParryCA 12f文言修正（≤11fまでの保証と明記）。RECOMMENDED修正: USoundCue→USoundBase、EnhancedInputモジュール追記。無敵フレーム・アーマーはOQ-4としてVS+延期記録。Status → `Revised (Pending Re-Review)`（3rd pass）。

---

## Review — 2026-05-23 — Verdict: MAJOR REVISION NEEDED（4th pass）
Scope signal: XL
Specialists: game-designer, systems-designer, qa-lead（今回は完全実行）, unreal-specialist, creative-director（senior）
Blocking items: 14 | Recommended: 7 | Scope-boundary rulings: 4
Prior verdict resolved: Partial — 前回12件は解消されたが、新規14件が発見された（rework regression の最終サイクルと診断）

Summary（creative-director）: 算術バグ（D-3b int32除算クラッシュ・ActiveFrameEnd条件逆転・Special技不変条件不可能・entities.yaml誤記・E-3 off-by-one）、Build.cs モジュール欠落（DataValidation/Niagara）、UE5.7 API検証ギャップ（AddBundleAsset変換式）、ParryCA 12f vs 重攻撃 Tuning Knob 8-18f 矛盾の4クラスターに集約。ACテスト矛盾（AC-5.3 条件逆転・AC-6 エスケープハッチ）も解消。スコープ境界4件のうちG1（キャンセルシステム）は状態機械GDDへ適切に延期、G3/G4/G5はこのGDDへの追記で解決。creative-director は「これ以上の full review は収穫逓減」と判断、次は unreal-specialist ターゲット検証 + /consistency-check を推奨。

**Revision applied (2026-05-23)**: 全14件のブロッカーを同セッション内で解決。設計判断3件確定: ①Special カテゴリを D-2 不変条件から免除（リスクの高い必殺技設計スペース確保）、②重攻撃 StartupFrames safe range を 8–11f に制限（ParryCA 12f 保証との整合）、③RangeTag 境界距離: Close≤80cm / Mid 80–200cm / Far>200cm。新規追加: MoveLevel Contract（照合ルール）、RangeTag Semantics（境界定義）、Coordinate Conventions（Facing Direction 変換契約）、Non-goal 宣言（キャンセル/コンボルートは状態機械GDD）。Status → `Revised (Pending Final Verification)`（4th pass）。残タスク: unreal-specialist による UE5.7 API 検証（B7）。


