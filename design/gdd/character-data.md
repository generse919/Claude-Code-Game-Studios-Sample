# キャラクターデータ定義

> **Status**: Revised (Pending Re-Review) — 4th pass 2026-05-23
> **Author**: game-designer + systems-designer (agents), user
> **Last Updated**: 2026-05-23
> **Implements Pillar**: Pillar 1「間合いが全て」/ Pillar 4「単純な外観、深い読み合い」

## Overview

キャラクターデータ定義は CLASH の全キャラクターに関するゲームプレイ値の単一ソースとなるデータレイヤーである。キャラクタースタッツ（体力上限・移動速度・ジャンプ高度）と各技のプロパティ（始動フレーム・攻撃フレーム・硬直フレーム・ヒットボックスサイズ・ダメージ量・リーチ）を UDataAsset として定義し、実装コードから分離する。このシステム自体はゲームプレイロジックを持たず、体力システム・ヒット判定・パリィシステム・ラウンド判定システムが参照する不変の真実として機能する。新キャラクターの追加・技のバランス調整はすべてこのデータ定義の変更によって行い、コード修正を不要とする。技の識別には `FName` ではなく `FGameplayTag` を使用する（UE5.5 の FName 挙動変更による）。

## Player Fantasy

このシステムは、プレイヤーが直接触れることのない**沈黙のインフラ**である。キャラクターデータ定義それ自体は感触を持たず、画面に何も映さない。しかしこのデータ層が存在しなければ、CLASH のあらゆる体験は語られ得ない。「間合いが全て」を成立させる移動速度・リーチ・ヒットボックスの数値も、「読み勝ちへの報酬」を支えるパリィ受付フレームと技の発生フレームも、「逆転可能な緊張感」を可能にする体力と被ダメージ補正も、すべてここで定義された数値の上に乗っている。プレイヤーは「キャラクターデータ」を体験するのではなく、**このデータが正しく設計された結果としてのゲーム**を体験する。

したがって本システムが目指すファンタジーは、データそのものの提示ではなく、**「このゲームのすべての駆け引きが、信頼できる数値の土台の上で起きている」という静かな保証**である。フレームデータが一貫していること、ダメージ計算が再現可能であること、キャラクター間のスタッツが意図通りの非対称性を持っていること——これらが守られているとき、プレイヤーは初めて「俺はお前の全てを見切った」という万能感に到達できる。読み合いの深さ（Pillar 4）は、数値の透明性と一貫性の上にしか成立しない。本ドキュメントは、その土台を逸脱なく定義するための契約書である。

## Detailed Design

### Core Rules

**Non-Goal（G1修正 — スコープ境界の明示）**:

> このGDDはキャラクターの**静的データスキーマ**を定義する。以下はこのGDDのスコープ外とする:
> - **コンボルートとキャンセル受付**: 技Aがどのフレームからどのカテゴリにキャンセルできるかの定義は **キャラクター状態機械 GDD（設計順 #3）** が責任を持つ。このGDDが提供する `StartupFrames`・`RecoveryFrames`・`OnHitAdvantage` はそのシステムが消費するデータである。
> - **ガード状態の遷移ロジック**: `MoveLevel` の照合（上記「MoveLevel Contract」）はデータ定義のみ。状態遷移（立ちガード→しゃがみガードへの切り替えコスト等）は状態機械 GDD で定義。
> - **キャラクター固有のバランス数値**: 2キャラクターの具体的スタッツ非対称性は → OQ-2・キャラクターデザインドキュメントで管理。

**1. データアーキテクチャ原則**

- 各キャラクターは 1 つの `UCharacterDataAsset`（`UPrimaryDataAsset` 継承）を持つ。
- 全ゲームプレイ値はこのアセット内にのみ定義する。コード内にマジックナンバーを持たない。
- アセットはマッチ開始時に一度だけロードし、以降はランタイムで**読み取り専用**として参照する。書き込みは絶対に行わない。
- 技の識別には `FGameplayTag` を使用する（`FName` 禁止 — UE5.5 以降の挙動変更への対応）。
- フレームデータはすべて **60fps 基準（1 フレーム = 1/60 秒）** で表現する。固定タイムステップの実装方式は別途 ADR で決定する。

**2. `UCharacterDataAsset` クラスフィールド**

`UPrimaryDataAsset` を継承するルートクラスのフィールド一覧。このクラスが Primary Asset Manager によってロードされる単位。

| フィールド名 | 型 | 説明 |
|---|---|---|
| `Stats` | `FCharacterStats` | キャラクタースタッツ（後述） |
| `MoveList` | `TArray<FMoveData>` | 全技データの配列（後述）。空はロード失敗 |

---

**3. キャラクタースタッツ（`FCharacterStats`）**

| フィールド名 | 型 | 値域 | 説明 |
|---|---|---|---|
| `CharacterTag` | `FGameplayTag` | `Character.*` | キャラクター識別子 |
| `DisplayName` | `FText` | — | 表示名 |
| `PrimaryColor` | `FLinearColor` | — | テーマカラー（HUD・キャラクター選択画面で使用。P1=青、P2=赤がデフォルト） |
| `MaxHealth` | `int32` | 500–2000 | HP 上限。デフォルト値: **1000** |
| `WalkSpeed` | `float` | 200–600 cm/s | 前進速度 |
| `BackWalkSpeedMultiplier` | `float` | 0.5–1.0 | 後退速度 = WalkSpeed × 係数 |
| `JumpHeight` | `float` | 300–700 cm | ジャンプ到達高度 |
| `JumpForwardDistance` | `float` | 200–600 cm | 前方ジャンプ時の水平移動量 |
| `StandHurtboxHalfExtent` | `FVector` | X/Y: 20–40, Z: 40–100 cm | 立ち状態ハートボックス半サイズ（ボックス中心 = キャラ原点から Z=HalfExtent.Z の高さ） |
| `CrouchHurtboxHalfExtent` | `FVector` | X/Y: 20–40, Z: 20–60 cm | しゃがみ状態ハートボックス半サイズ（ボックス中心 = キャラ原点から Z=HalfExtent.Z の高さ。立ちと同じ高さ基準） |
| `PushboxHalfExtent` | `FVector` | X/Y: 30–60, Z: 40–100 cm | キャラクター同士が重ならないための押し出しボックス |
| `ParryWindow` | `int32` | 3–8 フレーム | パリィ受付フレーム数。キャラごとに設定可能（MVP 粒度。VS+ で技単位オーバーライドを追加予定） |
| `ParryCounterAdvantage` | `int32` | **12–20 フレーム** | パリィ成立後の有利フレーム数（MVP 粒度。VS+ で技単位オーバーライドを追加予定）。下限 12f は軽攻撃（StartupFrames ≤ 8f）および重攻撃（**StartupFrames < 12f、すなわち ≤ 11f**）への確定反撃を保証するための最低値（確定反撃条件: `ParryCounterAdvantage > 相手技の StartupFrames`。等値は確定保証にならない） |

**4. 技カテゴリ（`EMoveCategory`）**

```
LightAttack  — 軽攻撃（軽パンチ・軽キック）：発生が早い、ダメージ低
HeavyAttack  — 重攻撃（重パンチ・重キック）：発生が遅い、ダメージ高、ノックバック大
Special      — 必殺技（MVP: 1ボタン）：高リターン高リスク
Parry        — パリィ（防御技）：入力後パリィ判定に入る
Throw        — 投げ技（**スキーマ予約 — MVP実装はVS+以降**）：ガード不可。MoveLevel判定を適用しない。Hitboxesは掴み判定として使用する
```

> **Throw（投げ技）のフィールド特例（B-3追加）**: `MoveCategory == Throw` の技は、ガード不可のため以下の通常制約を適用しない。①`BlockstunFrames` は 0 を許容（通常の `ClampMin=5` 例外）。② D-2 不変条件（`HitstunFrames > RecoveryFrames`）の適用外とする（ダウン遷移フレームとして扱う）。③ `MoveLevel` はヒット判定システムが無視する（投げはガード規則の対象外）。`IsDataValid()` は `MoveCategory == Throw` を検出してこれらの例外を適用すること。**MVP では Throw カテゴリのデータ定義のみ可能。投げ実装は VS+ で行う**。

**5. 攻撃の段（`EMoveLevel`）**

```
High      — 上段（しゃがんでいる相手には当たらない——ガードなしで自動回避可能。立ち姿勢の相手に当たる。立ちガードで防げる）
Mid       — 中段（立ち・しゃがみどちらの姿勢でも当たる——ガードが必要。立ち・しゃがみいずれのガードでも防げる）
Low       — 下段（しゃがみガードのみ防げる。立ち姿勢には当たる）
Overhead  — 頭上段（しゃがみガードでは防げない。立ちガードのみ防げる）
```

> **High と Mid の違い（B-2補完）**: High は「しゃがむだけで自動回避できる攻撃」（Street Fighter・Tekken の業界標準定義）。Mid は「しゃがんでいても当たる攻撃（ガード必須）」。ヒット判定システムは受撃側の `bIsCrouching` フラグと攻撃側の `MoveLevel` を照合して当否を決定する。**MVP では `High` と `Low` の 2 段階で実装する。`Mid`・`Overhead` は Vertical Slice 以降で追加する。**

> **MoveLevel と受撃判定の照合契約（G4修正 — ヒット判定システムへの実装契約）**:
> ヒット判定システムは受撃側の `bIsCrouching` フラグと攻撃側の `FMoveData.MoveLevel` を照合して当否を決定する。
>
> | MoveLevel | 受撃側が立ち（bIsCrouching=false） | 受撃側がしゃがみ（bIsCrouching=true） |
> |-----------|----------------------------------|--------------------------------------|
> | **High** | 当たる（立ちガードで防げる） | **当たらない**（ガード不要 — しゃがみで自動回避） |
> | **Low** | 当たる（しゃがみガードのみ防げる） | 当たる（しゃがみガードのみ防げる） |
> | Mid（VS+） | 当たる（どちらのガードでも防げる） | 当たる（どちらのガードでも防げる） |
> | Overhead（VS+） | 当たる（立ちガードのみ防げる） | 当たる（立ちガードのみ防げる） |
>
> **ガード判定の補足**: 立ちガード (`bIsGuarding=true, bIsCrouching=false`) は High・Mid を防げる。しゃがみガード (`bIsGuarding=true, bIsCrouching=true`) は Low・Mid を防げる。High 攻撃に対してしゃがみ状態（ガードの有無に関わらず）は自動回避。この照合ルールの実装責任はヒット判定システム GDD（設計順 #6）にある。`MoveLevel` はあくまでデータの単一権威であり、解決ロジックはデータ層に属さない。

**6. 技データ（`FMoveData`）**

| フィールド名 | 型 | 値域 | 説明 |
|---|---|---|---|
| `MoveTag` | `FGameplayTag` | `Move.*` | 技識別子（例: `Move.LightPunch`） |
| `MoveCategory` | `EMoveCategory` | enum | 技カテゴリ |
| `MoveLevel` | `EMoveLevel` | enum | 攻撃の段（High/Low が MVP 必須。Mid/Overhead は VS+）。**当たり判定の単一権威**（B-2） |
| `RangeTag` | `FGameplayTag` | `Move.Range.{Close\|Mid\|Far}` **必須** | 技の間合いカテゴリ（Pillar 1「間合いが全て」のデータフック）。ヒット判定システムおよびキャラクター AI が間合い判断に参照する。**`Validate()` でこのフィールドが空または `Move.Range.*` 配下でない場合はエラー**（B-5）。境界距離定義は下記「RangeTag セマンティクス」を参照 |
| `InputAction` | `TSoftObjectPtr<UInputAction>` | — | 対応する Enhanced Input アクション（Asset Bundle に含めて事前ロード） |
| `StartupFrames` | `int32` | 1–30 f | 入力から攻撃判定発生まで |
| `ActiveFrames` | `int32` | 1–15 f | ヒットボックスが有効な期間 |
| `RecoveryFrames` | `int32` | 2–60 f | 攻撃後の硬直（技終了まで） |
| `BlockstunFrames` | `int32` | 5–25 f | ガードされた時、相手の硬直フレーム数 |
| `HitstunFrames` | `int32` | 10–40 f | ヒット時、相手の硬直フレーム数 |
| `BaseDamage` | `int32` | 50–300 | 基礎ダメージ値（ClampMin=50 — D-3 カテゴリ下限と整合） |
| `KnockbackDistance` | `float` | 0–500 cm | ヒット時の相手水平押し込み量 |
| `KnockbackDirection` | `FVector2D` | 正規化 | ノックバック方向（**正規化必須。ただし `KnockbackDistance=0` のときはスキップ**）。座標系: X = 水平（正 = 攻撃者から見た前方）、Y = 垂直（正 = 上）。例: `(1,0)` = 真横、`(0,1)` = 真上（ランチャー技）、`(0.707, 0.707)` = 45° 斜め上。`Validate()` で `KnockbackDistance > 0` の場合のみ長さが `1.0 ± 0.01` を確認する（B-9） |
| `HitEffect` | `TSoftObjectPtr<UNiagaraSystem>` | — | ヒット時エフェクト参照（Asset Bundle に含めて事前ロード） |
| `HitSoundCue` | `TSoftObjectPtr<USoundBase>` | — | ヒット音参照（Asset Bundle に含めて事前ロード）。`USoundBase` は `USoundCue` と `UMetaSoundSource` の共通基底クラス。MetaSounds 移行時も型変更不要 |
| `BlockSoundCue` | `TSoftObjectPtr<USoundBase>` | — | ガード音参照（Asset Bundle に含めて事前ロード）。同上 |
| `Hitboxes` | `TArray<FHitboxData>` | 要素数 1–4 | この技のヒットボックス定義（後述） |
| `ParryWindowOverride` | `int32` | 0–8 f | この技がパリィされる受付ウィンドウ（**0 = キャラ基礎値 `ParryWindow` を使用**。VS+ で技単位調整に使用） |
| `OnParriedCounterAdvantage` | `int32` | 0–20 f | この技がパリィされた際の相手の有利フレーム（**0 = キャラ基礎値 `ParryCounterAdvantage` を使用**。必殺技は大きく設定推奨） |

> **RangeTag セマンティクス（G3修正 — 境界距離定義）**:
> ヒット判定システムおよびキャラクター AI（設計順 #18）が間合い判断に使用する境界距離。
>
> | タグ | 境界距離（ヒットボックス最大リーチ基準） | 想定技種 |
> |------|---------------------------------------|--------|
> | `Move.Range.Close` | ≤ 80cm | 密着・投げ射程想定の接近戦技 |
> | `Move.Range.Mid` | 80–200cm | 通常打撃（立ちパンチ・蹴り）の主射程 |
> | `Move.Range.Far` | > 200cm | リーチ重視・前進置き技 |
>
> 「ヒットボックス最大リーチ」は当該技の `Hitboxes` 内の `HalfExtent.X + |LocalOffset.X|` の最大値で算出する。RangeTag と実際のヒットボックスリーチが境界値を超えている場合は `Validate()` でエディタ警告（保存許可）を発生させること（整合性ヒント）。

**7. ヒットボックスデータ（`FHitboxData`）**

`FBox` は Blueprint 非対応のため `FVector + HalfExtent` 分解形式を使用する。

**フレームカウント基準（絶対フレーム基準）**:
- 技入力を受け付けたフレームを **Frame 0（絶対フレーム 0）** とする
- `StartupFrames=6` の技は Frame 0–5 がスタートアップ期間、Frame 6 が最初のアクティブフレーム
- `ActiveFrameStart=6`、`ActiveFrameEnd=8` であれば Frame 6・7・8 でヒットボックスが有効
- `ActiveFrameStart` の有効値 = `StartupFrames` 以上が設計原則（`ActiveFrameStart < StartupFrames` はエディタ警告）
- `ActiveFrameEnd` の有効値 = `StartupFrames + ActiveFrames - 1` 以下が設計原則（超過はエディタ警告）

| フィールド名 | 型 | 値域 | 説明 |
|---|---|---|---|
| `HitboxTag` | `FGameplayTag` | `Hitbox.*` | 同一技で複数ヒットボックスを区別する識別子 |
| `HalfExtent` | `FVector` | X/Y/Z: 5–150 cm | ボックス半サイズ（AABB） |
| `LocalOffset` | `FVector` | X: -200–+200, Z: -100–+200 cm | キャラクター原点からのオフセット（Z=0 がキャラクター足元原点） |
| `ActiveFrameStart` | `int32` | 1–90 f | このヒットボックスが有効になる絶対フレーム番号（Frame 0 = 入力フレーム） |
| `ActiveFrameEnd` | `int32` | 1–90 f | このヒットボックスが無効になる絶対フレーム番号（この値のフレームも有効） |

> **MoveLevel 権威の原則（B-2 修正）**: `FMoveData.MoveLevel` を当たり判定の唯一の権威とする。ヒット判定システムは `MoveLevel` のみを参照して「この技がしゃがみ / 立ちどちらに当たるか」を決定する。`CanHitLow` / `CanHitHigh` は C++ 側でヘルパーとして派生計算する（例: `bool GetCanHitLow() const { return MoveLevel == EMoveLevel::Low; }`）。FHitboxData にこれらのフィールドは存在しない。

> **座標系と Facing Direction 変換契約（G5修正 — 消費システムへの実装契約）**:
> `FHitboxData.LocalOffset` および `FMoveData.KnockbackDirection` は**攻撃者ローカル座標系**で定義される。
>
> - `LocalOffset.X` 正方向 = 攻撃者が向いている前方向
> - `KnockbackDirection.X` 正方向 = 攻撃者から見た前方向
>
> **消費者（ヒット判定システム・基本攻撃システム）の変換責任**: ワールド座標への変換時に攻撃者の `FacingSign`（+1 = 右向き、-1 = 左向き）を `LocalOffset.X` および `KnockbackDirection.X` に乗算すること。`Y`・`Z` 成分（垂直）は FacingSign の影響を受けない。
>
> 例: 攻撃者が左向き（FacingSign=-1）の場合、`LocalOffset=(100, 0, 50)` → ワールド水平オフセット = `-100cm`（攻撃者の前方 = 画面左）。`KnockbackDirection=(1, 0)` → ワールドノックバック方向 = `-X`（攻撃者前方 = 画面左）。この変換契約を両システムが同一実装することで P1/P2 の左右対称性を保証する。

> **実装注記（→ ADR-001）**: Animation Notify はヒット判定に使用しない。判定は固定タイムステップ内のフレームカウントで行う（フレーム精度の実装方式は ADR-001「固定タイムステップとフレーム権威」で決定）。Notify はエフェクト・音の発火のみに使用する。`ActiveFrameStart` と `ActiveFrameEnd` は **絶対フレーム基準**（技入力 = Frame 0）で解釈する。

### States and Transitions

キャラクターデータアセット自体は状態を持たない静的なデータ層である。ライフサイクルのみ定義する。

| フェーズ | 状態 | 説明 |
|---------|------|------|
| エディタ時 | 編集可能 | データアセットをエディタで直接編集する |
| ゲーム起動 | ロード待ち | Primary Asset Manager がアセットをロードするまで参照不可 |
| マッチ実行中 | **読み取り専用** | ロード後はいかなるシステムも値を書き換えてはならない |
| マッチ終了 | アンロード候補 | Asset Manager の裁量でアンロード可（同セッション内では通常キャッシュ保持） |

> **`IsDataValid()` のランタイム非実行について（B-R3 修正、B11修正）**: `IsDataValid(FDataValidationContext&)` は**エディタ専用のバリデーション関数**であり、パッケージ（製品）ビルドのロード時には自動実行されない。エッジケースマトリクスで「ロード時 (IsDataValid()) → ロード失敗」と記述している箇所は、エディタ環境（プロジェクト開発中）での挙動を指す。
>
> **製品ビルドでの検証戦略（B11修正: アーキテクチャ欠陥の修正）**:
> - **スカラー値域検証**（`MaxHealth`・`StartupFrames` 等）: 消費側システムの `PostLoad()` オーバーライドで検証可能
> - **ソフト参照（nested TSoftObjectPtr）の検証**: `PostLoad()` は **ソフト参照が解決される前**に呼ばれるため `InputAction`・`HitEffect`・`HitSoundCue`・`BlockSoundCue` の有効性を `PostLoad()` では確認できない。**Asset Bundle Load 完了コールバック**内での検証が必要（`UPrimaryAssetManager::LoadPrimaryAssets()` の完了後）。
>
> この設計判断の詳細は → **ADR-002「Asset Manager 統合戦略」** で確定すること。ADR-002 は少なくとも「製品ビルドでの soft ref 検証タイミング（a: Bundle Load コールバック内検証 / b: 初回アクセス時検証 / c: エディタのみで容認しリリース前チェックリストで担保）」の3択を検討し、選択された方式を実装契約として記述すること。

**二次ロード戦略（ネストされた `TSoftObjectPtr` の解決）**:

`FMoveData` 内の `InputAction`・`HitEffect`・`HitSoundCue`・`BlockSoundCue` はソフト参照であり、`UCharacterDataAsset` 本体のロードでは自動解決されない。

採用方式: **Primary Asset Bundle に含める**。`UCharacterDataAsset` の Asset Bundle に `UInputAction`・`UNiagaraSystem`・`USoundCue` を宣言し、Asset Manager が本体ロード時に一括ロードする。マッチ開始前にすべての参照が解決済みとなることを保証する。`InputAction` が未解決の場合は技の発動が不可能になるため、事前ロードは必須。

**Asset Manager 統合の必須実装要件（B-7追加）**:

1. **`GetPrimaryAssetId()` のオーバーライド（必須）**: `UCharacterDataAsset` は `GetPrimaryAssetId()` をオーバーライドし `FPrimaryAssetId(TEXT("CharacterData"), GetFName())` を返すこと。**重要**: この戻り値の AssetType 文字列（`TEXT("CharacterData")`）と `DefaultGame.ini` の `AssetType` 設定値は**完全一致が必須**。食い違った場合 Asset Manager はアセットを静黙に無視する（コンパイルエラーなし）。

2. **`DefaultGame.ini` スキャンパス登録（必須）**: プロジェクトの `DefaultGame.ini` の `[/Script/Engine.AssetManagerSettings]` セクションに `PrimaryAssetTypesToScan` エントリを追加し、`AssetType=CharacterData`・`AssetBaseClass=UCharacterDataAsset`・`Directories=((Path="/Game/Characters"))` を設定すること。`AssetType` の文字列が要件1の `GetPrimaryAssetId()` 戻り値と一致していることを確認する。

3. **`TArray<FMoveData>` 内 `TSoftObjectPtr` の Asset Bundle 収集（必須）**: `UPROPERTY(meta=(AssetBundles="..."))` タグは 2 段ネストの soft ptr を自動収集しない。`UCharacterDataAsset::UpdateAssetBundleData()` を C++ でオーバーライドし、全 `MoveList` 要素の `InputAction`・`HitEffect`・`HitSoundCue`・`BlockSoundCue` を手動で `AssetBundleData.AddBundleAsset()` で登録すること。**実装時注意（B7修正: UE5.7 要確認）**: UE5.2 以降 `AddBundleAsset()` の第2引数は `FTopLevelAssetPath`（UE5.1 以前の `FSoftObjectPath` から変更）。ソフト参照からの変換式 `SoftPtr.GetUniqueID().GetAssetPath()` は UE5.7 での動作が **LLMの知識範囲外（5.3止まり）のため検証未確認**。サイレント失敗モード（コンパイルエラーなし）のため実装前にエンジンソース（`AssetBundleData.h`）で API シグネチャを必ず確認すること。正しい変換の候補: `SoftPtr.ToSoftObjectPath().GetAssetPath()`（→ ADR-002「Asset Manager 統合戦略」で詳細確定・エンジンソース確認タスクを発行すること）。

### Interactions with Other Systems

| 下流システム | 参照するフィールド | データの流れ |
|------------|-----------------|------------|
| **体力システム** | `MaxHealth` | キャラクター開始時に HP 上限をセット |
| **キャラクター移動システム** | `WalkSpeed`, `BackWalkSpeedMultiplier`, `JumpHeight`, `JumpForwardDistance`, `PushboxHalfExtent` | 毎フレームの移動量計算に使用 |
| **ヒット判定システム** | `Hitboxes`, `ActiveFrameStart/End`, `MoveLevel`（単一権威）, `StandHurtboxHalfExtent`, `CrouchHurtboxHalfExtent` | フレームカウントでヒットボックス有効期間を判断し、`MoveLevel` と受撃側ガード姿勢を照合して当たり判定を実行 |
| **キャラクター状態機械** | `MoveList`（`MoveTag`, `MoveCategory`, `StartupFrames`, `ActiveFrames`, `RecoveryFrames`） | 技入力時に状態遷移の有効性とタイミングを参照 |
| **基本攻撃システム** | `BaseDamage`, `HitstunFrames`, `BlockstunFrames`, `KnockbackDistance`, `KnockbackDirection` | ヒット時のダメージと状態付与の計算 |
| **パリィシステム** | `ParryWindow`, `ParryCounterAdvantage` | パリィ受付判定のウィンドウと成立後有利フレームの取得 |
| **視覚フィードバックシステム** | `HitEffect`, `HitSoundCue`, `BlockSoundCue` | ヒット/ガード時のエフェクト・音を参照してスポーン |
| **音響システム** | `HitSoundCue`, `BlockSoundCue` | 音キューの参照（視覚FBシステムと共用） |

## Formulas

### D-1: OnBlockAdvantage（ガード時フレーム有利不利）

```
OnBlockAdvantage = BlockstunFrames - RecoveryFrames
```

| シンボル | 型 | 値域 | 説明 |
|---|---|---|---|
| `BlockstunFrames` | int32 | 5–25 f | ガードされた相手に発生する硬直フレーム数 |
| `RecoveryFrames` | int32 | 2–60 f | 攻撃した側の技終了後硬直フレーム数 |
| `OnBlockAdvantage` | int32 | -55–+23 f | 正値=攻撃側有利・負値=防御側有利 |

**出力範囲**: 設計推奨値 -30〜+5 f。派生計算値のためデータアセットには保存しない（`BlockstunFrames` と `RecoveryFrames` から随時算出）。

**例（軽パンチ）**: BlockstunFrames=12, RecoveryFrames=14 → OnBlockAdvantage = **-2** f（攻撃側2フレーム不利、相手に反撃機会あり）

---

### D-2: OnHitAdvantage（ヒット時フレーム有利不利）

```
OnHitAdvantage = HitstunFrames - RecoveryFrames
```

| シンボル | 型 | 値域 | 説明 |
|---|---|---|---|
| `HitstunFrames` | int32 | 10–40 f | ヒットした相手に発生する硬直フレーム数 |
| `RecoveryFrames` | int32 | 2–60 f | 攻撃した側の硬直フレーム数（D-1 と同変数） |
| `OnHitAdvantage` | int32 | **+1〜+38 f**（LightAttack/HeavyAttack/Parry は負値禁止。Special/Throw は負値許容） | CLASH では LightAttack・HeavyAttack は常に正値（ヒットした側が有利）が設計原則。Special/Throw は特例として負値を許容する（B3修正） |

**出力範囲（通常技）**: 設計推奨値 +3〜+25 f。最低保証値 +1 f（ヒット後に攻撃側が必ず有利）。  
**出力範囲（Special 特例）**: HitstunFrames 上限 40f / RecoveryFrames 上限 60f により理論的最小値 = 40 - 60 = **-20f**（ガードされたら大差不利な必殺技の設計スペース）。

**不変条件1**: `OnHitAdvantage > OnBlockAdvantage` は必須。ヒットよりガードの方が有利になる技は CLASH に存在しない。  
**不変条件2（B-3追加）**: `OnHitAdvantage ≥ 1`（= `HitstunFrames > RecoveryFrames`）は必須。`Validate()` で `HitstunFrames ≤ RecoveryFrames` の場合は Invalid を返す（保存ブロック）。負値は Pillar 2「読み勝ちへの報酬」と根本矛盾する。**ただし `MoveCategory == Throw` および `MoveCategory == Special` の場合は正常として許容する**（Special 特例 — B3修正: Special は「ガードされたら大差不利」という設計スペースを確保するため。Throw 特例と同様に IsDataValid() で MoveCategory を検出して適用する）。

**例（軽パンチ）**: HitstunFrames=20, RecoveryFrames=14 → OnHitAdvantage = **+6** f（次の軽パンチの StartupFrames が6f以内ならコンボ成立）

---

### D-3: BaseDamage 値域根拠と KO 計算式

**D-3a: 一般 KO 計算式**

```
KO_hits = ceil(MaxHealth / BaseDamage_effective)
```

MaxHealth 変動時に TTK（Time-to-Kill）比を維持するための按分式（**ランタイム自動適用**）:

```
BaseDamage_effective = (float)BaseDamage_base * MaxHealth / 1000.0f
```

> ⚠️ **整数除算バグ防止（B1修正）**: `BaseDamage_base * (MaxHealth / 1000)` と書いた場合、`int32` 同士の整数除算により MaxHealth=500 → 500/1000=**0** となり BaseDamage_effective=0 → KO計算でゼロ除算クラッシュ。MaxHealth=1200 → 1倍（期待値1.2倍）など 1000・2000 以外で完全に誤動作する。必ず `float` キャストを使用すること。

**適用主体**: `BaseDamage_effective` はデータアセットには保存しない。データアセットには **`MaxHealth = 1000` 基準の `BaseDamage_base`** のみを定義する。ランタイムでのスケール適用は**体力システムが担当**し、ダメージ計算時に `(float)BaseDamage_base * MaxHealth / 1000.0f` を計算する（詳細は体力システム GDD で定義 — AC-7.1 参照）。この方式により `MaxHealth` を変更したキャラクターでも `BaseDamage_base` の値を変更せず TTK を維持できる。`ClampMin=50` は `BaseDamage_base`（MH=1000 基準）に対して適用する。

**D-3b: BaseDamage 設計推奨値域（MaxHealth=1000 基準）**

以下の KO 打数は `MaxHealth=1000` のときのみ成立する。MaxHealth を変更する場合は D-3a の按分式で再算出すること。

| 攻撃カテゴリ | BaseDamage 推奨値域 | KO 打数（MH=1000） | 設計方針 |
|---|---|---|---|
| LightAttack | 50–80 | 13〜20 発 | 地ならしツール。多段プレッシャーが主目的 |
| HeavyAttack | 120–180 | 6〜9 発 | 読み勝ちの報酬。軽攻撃の約 2.5 倍 |
| Special | 200–300 | 4〜5 発 | ハイリスク逆転ツール。ガードされると大きな隙を生む |

**実戦的 KO 構成例（MH=1000）**: 軽攻撃 5 発（300）+ 重攻撃 3 発（450）+ 必殺技 1 発（250）= 1000。3 カテゴリが均等に機能するバランス。

> ダメージ補正（コンボスケーリング）は**体力システム GDD で定義**する。BaseDamage はコンボ補正前の素の数値。

## Edge Cases

### E-1: フレームデータの境界値

> **ClampMin/ClampMax はエディタ UI 制約（B-R4 修正）**: 以下の「ClampMin により〜を拒否する」は、エディタの Details パネルにおけるスピナー入力制限を指す。プログラム的なデータ構築（C++ テストコード・Python スクリプト等）では `ClampMin` は無効。**実質的な保存ブロックゲートは `IsDataValid(FDataValidationContext&)` の実装**（AC-5）。`IsDataValid()` 内で同等の値域チェックを必ず実装すること。

- **If `StartupFrames` = 0**: `ClampMin="1"` によりエディタ UI が入力を制限する（`IsDataValid()` でも同等チェックを実施）。0 は「入力と同時に攻撃判定が発生する」を意味し、固定タイムステップモデルでは実装不能。
- **If `ActiveFrames` = 0**: `ClampMin="1"` によりエディタ UI が入力を制限する（`IsDataValid()` で同等チェック）。有効フレームが存在しない技はヒット判定を持てない。
- **If `RecoveryFrames` ≤ 1**: `ClampMin="2"` によりエディタ UI が入力を制限する（`IsDataValid()` で同等チェック）。RecoveryFrames = 0 は「即次行動可能」を意味し技キャンセルとの区別が消失する。
- **If `StartupFrames` + `ActiveFrames` + `RecoveryFrames` > 120**: エディタ警告のみ（保存許可）。120f = 2 秒超の技は将来の特殊技への拡張余地として禁止しないが、レビュー推奨。
- **If `ParryWindow` = 0**: `ClampMin="3"` によりエディタ UI が入力を制限する（`IsDataValid()` で同等チェック）。受付 0f ではパリィ操作が入力不可能となりパリィシステムが無効化される。

### E-2: D-2 不変条件違反

- **If `HitstunFrames` ≤ `BlockstunFrames`（同一 `FMoveData` 内）**:
  - エディタ保存時: `IsDataValid()` でエラーを返し保存をブロックする（`PostEditChangeProperty` はブロック不可 — 警告ログと値修正のみ）。
  - ロード時: `IsDataValid()` / `Validate()` でも同一チェックを実施し、違反データはロード失敗とする（エディタ前の古い `.uasset` や直接書き込まれたアセットへの対策）。
  - 理由: 「ヒットよりガードの方が有利になる技」は Pillar 4 に根本矛盾するため設定ミスとして扱う。
- **If `HitstunFrames` ≤ `RecoveryFrames`（B-3追加）**:
  - `IsDataValid()` でエラーを返し保存をブロックする。ロード時も同一チェック。**ただし `MoveCategory == Throw` および `MoveCategory == Special` の場合は正常として許容する**（Throw/Special 特例 — B3修正: Special は「ガードされたら大差不利な必殺技（RecoveryFrames > HitstunFrames 上限40f）」という設計スペースを確保するため免除）。
  - 理由: `OnHitAdvantage = HitstunFrames - RecoveryFrames ≤ 0` はヒット後に攻撃側が不利または同値になることを意味し、「常に正値」の設計原則および Pillar 2「読み勝ちへの報酬」に根本矛盾する。Special 特例は高リスク必殺技の設計自由度のためのみ認める。
- **If `OnHitAdvantage` < +3 f（推奨下限未満だが ≥ +1 を満たす）**: エディタ警告のみ（保存許可）。弱い有利フレームの技は設計として許容する。
- **If `OnBlockAdvantage` > +5 f（推奨上限超過）**: エディタ警告のみ（保存許可）。強いガード有利技は設計判断として存在しうるが、バランス審査が必要。

### E-3: データ不整合

- **If `MoveList` が空**: `Validate()` でエラーを返し Primary Asset Manager のロードを失敗させる。技を持たないキャラクターはゲームプレイを成立させられない。
- **If 同一キャラクターの `MoveList` 内に重複する `MoveTag` が存在**: `Validate()` でエラー、ロード失敗。同一タグを持つ技が 2 つ存在するとタグ検索で返却される技が不定になる。
- **If 異なるキャラクター間で同じ `MoveTag` が存在**: 正常。`Move.LightPunch` を複数キャラが持つことは想定済み。キャラクター識別は `CharacterTag` で行う。
- **If `Hitboxes` が空**（`MoveCategory != Parry` の場合）: `Validate()` でエラー、ロード失敗。ヒットボックスを持たない攻撃技は当たり判定が発生しない。
- **If `Hitboxes` が空**（`MoveCategory = Parry` の場合）: **正常**。パリィ技はヒットボックスを持たない。空を許容する。
- **If `Hitboxes` の要素数が 4 を超える**: エディタ警告のみ（保存許可）。5 個以上はパフォーマンス審査が必要であることをログに記録する。
- **If `FHitboxData.ActiveFrameEnd` < `FHitboxData.ActiveFrameStart`**: `IsDataValid()` でエラーを返し保存をブロックする（B-6、B2修正: `≤` から `<` に変更）。`ActiveFrameEnd == ActiveFrameStart` は「1フレームヒットボックス（単発フレームのみ有効）」として正当。有効区間が負（End < Start）のヒットボックスのみ無効。
- **If `FHitboxData.ActiveFrameEnd` > `StartupFrames` + `ActiveFrames` - 1**: エディタ警告のみ（保存許可）。技の有効期間外にヒットボックスが残る設計は意図的な可能性があるため禁止しない。（B5修正: 従来の記述 `> StartupFrames + ActiveFrames` は1フレームずれており、マトリクスの `> StartupFrames + ActiveFrames - 1` が正しい基準に修正）

### E-4: 未登録 GameplayTag

> **前提実装要件（B-8追加、B6修正）**: `FGameplayTag` および `UInputAction` を使用するために以下の設定が必須。
> ①プロジェクト `Build.cs` の `PublicDependencyModuleNames` に以下を追加:
> - `"GameplayTags"` — `FGameplayTag` 型のため（必須）
> - `"EnhancedInput"` — `UInputAction` 型のため（`EnhancedInput` モジュールに属す。未追加だとコンパイルエラー）
> - `"Niagara"` — `UNiagaraSystem` 型のため（`TSoftObjectPtr<UNiagaraSystem>` の型解決に必要。**B6修正: 以前の要件から欠落していた**）
>
> ②`IsDataValid(FDataValidationContext&)` を使用するために `Build.cs` に以下を追加（**B6修正: 以前の要件から欠落していた**）:
> ```csharp
> if (Target.bBuildEditor) {
>     PrivateDependencyModuleNames.Add("DataValidation");
> }
> ```
> `FDataValidationContext` と `EDataValidationResult` は `DataValidation` エディタモジュールに定義されている。未追加だとコンパイルエラー（エラーメッセージが不明瞭なため注意）。
>
> ③ `Config/DefaultGameplayTags.ini`（または `GameplayTags` プラグイン設定経由）に `Character.*`、`Move.*`（`Move.Range.Close`/`Move.Range.Mid`/`Move.Range.Far` を含む）、`Hitbox.*` タグ階層を定義する。タグソースが存在しない状態では E-4 のバリデーションチェック自体が機能しない（→ ADR-003「GameplayTags モジュール統合とタグ命名規約」で全タグ定義を管理）。

- **If `MoveTag` が GameplayTag レジストリに存在しない**: `Validate()` でエラー、ロード失敗。無効タグを持つ技はコンボ判定・パリィカウンター判定が一切機能しない。
- **If `CharacterTag` が GameplayTag レジストリに存在しない**: `Validate()` でエラー、ロード失敗。キャラクター識別が不能になりマッチャーが対戦相手を特定できない。
- **If `HitboxTag` が GameplayTag レジストリに存在しない**: `Validate()` で警告のみ。ヒット判定は機能するがエフェクト・音の分岐ロジックが不定動作になる。QA フラグを立てる。
- **If `InputAction` が `nullptr` または未ロード**: 実行時に「この技は入力トリガーなしに発動不可」として扱い警告をログ出力する。`TSoftObjectPtr` 解決失敗はクラッシュではなくサイレント失敗。

### E-5: 値域外の値

- **If `MaxHealth` ≤ 0**: `ClampMin="500"` によりエディタ UI が制限する（`IsDataValid()` で同等チェック）。0 以下はキャラクターが初期状態でKO。
- **If `WalkSpeed` = 0**: `ClampMin="200"` によりエディタ UI が制限する（`IsDataValid()` で同等チェック）。移動不能キャラクターは間合い管理（Pillar 1）が成立しない。
- **If `BaseDamage` < 50**: `ClampMin="50"` によりエディタ UI が制限する（`IsDataValid()` で同等チェック）。`ClampMin` は `BaseDamage_base`（MH=1000 基準）に対して適用。ダメージ 50 未満は KO 手数が 20 発超となり試合テンポを損なう。
- **If `KnockbackDistance` = 0**: **正常**。ノックバックなしの技（その場ヒット技）は設計として有効。
- **If `BackWalkSpeedMultiplier` = 0**: `ClampMin="0.5"` によりエディタ UI が制限する（`IsDataValid()` で同等チェック）。後退不能はキャラクター非対称性の崩壊。

### エッジケース処理マトリクス

| 違反種別 | 検出タイミング | 処理 |
|---------|------------|------|
| フレームデータ値域外 | エディタ保存時 (ClampMin/Max) | 保存ブロック |
| HitstunFrames ≤ BlockstunFrames | エディタ保存時 (IsDataValid()) | 保存ブロック（※PostEditChangeProperty は警告のみ） |
| HitstunFrames ≤ BlockstunFrames | ロード時 (IsDataValid()) | ロード失敗 |
| HitstunFrames ≤ RecoveryFrames（B-3追加） | エディタ保存時 (IsDataValid()) | 保存ブロック（Throw/Special 特例あり） |
| HitstunFrames ≤ RecoveryFrames（B-3追加） | ロード時 (IsDataValid()) | ロード失敗（Throw/Special 特例あり） |
| MoveCategory=Special の技の HitstunFrames ≤ RecoveryFrames | エディタ保存時 / ロード時 | **正常**（Special 特例 — B3修正）。リスクの高い必殺技の「ガードされたら大差不利」設計スペースを確保するため免除。IsDataValid() は MoveCategory==Special を検出してこの条件を許容する |
| ActiveFrameEnd < ActiveFrameStart（厳密な未満のみ）| エディタ保存時 (IsDataValid()) | 保存ブロック（等値=1フレームヒットボックスは正常） |
| ActiveFrameStart < StartupFrames | エディタ保存時 | 警告のみ（意図的な早期ヒットボックスとして許容） |
| ActiveFrameEnd > StartupFrames + ActiveFrames - 1 | エディタ保存時 | 警告のみ（意図的な遅延ヒットボックスとして許容） |
| MoveCategory=Throw の技の `BlockstunFrames=0` | エディタ保存時 / ロード時 | **正常**（Throw 特例）。ガード不可技は BlockstunFrames が 0 でよい。IsDataValid() は MoveCategory==Throw を検出してこの値を許容する |
| MoveCategory=Throw の技の `HitstunFrames ≤ RecoveryFrames` | エディタ保存時 / ロード時 | **正常**（Throw 特例）。D-2 不変条件（HitstunFrames > RecoveryFrames）は Throw には適用しない |
| MoveList 空 / 重複 MoveTag / Hitboxes 空 (非Parry) / HitstunFrames 不変条件違反 | ロード時 (IsDataValid()) | ロード失敗 |
| 無効 GameplayTag (MoveTag / CharacterTag) | ロード時 (IsDataValid()) | ロード失敗 |
| RangeTag が空または Move.Range.* 配下でない（B-5追加） | エディタ保存時 + ロード時 (IsDataValid()) | 保存ブロック / ロード失敗 |
| KnockbackDirection 非正規化（長さ ≠ 1.0 ± 0.01）**かつ KnockbackDistance > 0**（B-9修正） | ロード時 (IsDataValid()) | ロード失敗 |
| HitboxTag 未登録 | ロード時 (Validate()) | 警告のみ |
| Hitboxes 要素数 > 4 | エディタ保存時 | 警告のみ |
| OnBlockAdvantage 推奨範囲外 | エディタ保存時 | 警告のみ |

## Dependencies

**上流依存（このシステムが依存するシステム）**: なし
> Foundation レイヤーのデータシステム。他のどのシステムにも依存しない。

**下流依存（このシステムに依存するシステム）**:

| 依存システム | 依存の種類 | 参照するフィールド | インターフェース |
|------------|-----------|-----------------|--------------|
| 体力システム | **ハード**（MaxHealth なしで体力システムは機能不可） | `MaxHealth` | マッチ開始時にキャラアセットから `MaxHealth` を取得し初期HPをセット |
| キャラクター移動システム | **ハード** | `WalkSpeed`, `BackWalkSpeedMultiplier`, `JumpHeight`, `JumpForwardDistance`, `PushboxHalfExtent` | 毎フレームの移動量計算に参照 |
| ヒット判定システム | **ハード** | `Hitboxes[]`, `MoveLevel`（当たり判定権威）, `StandHurtboxHalfExtent`, `CrouchHurtboxHalfExtent` | フレームカウントでヒットボックス有効期間を判断し、MoveLevel で攻撃の段を解決して衝突検出 |
| キャラクター状態機械 | **ハード** | `MoveList`（`MoveTag`, `MoveCategory`, `StartupFrames`, `ActiveFrames`, `RecoveryFrames`） | 技入力時の状態遷移とタイミング管理 |
| 基本攻撃システム | **ハード** | `BaseDamage`, `HitstunFrames`, `BlockstunFrames`, `KnockbackDistance`, `KnockbackDirection` | ヒット時のダメージ計算と状態付与 |
| パリィシステム | **ハード** | `ParryWindow`, `ParryCounterAdvantage` | パリィ受付判定と成立後有利フレームの取得 |
| 視覚フィードバックシステム | **ソフト**（参照が null なら効果なし） | `HitEffect`, `HitSoundCue`, `BlockSoundCue` | ヒット/ガード時にエフェクト・音をスポーン |
| 音響システム | **ソフト** | `HitSoundCue`, `BlockSoundCue` | 音キューの参照 |

> **双方向注記**: 上記の各下流システムの Dependencies セクションには「キャラクターデータ定義に依存する」と記載すること。

## Tuning Knobs

| チューニングノブ | 安全範囲 | 高すぎると | 低すぎると | 影響するゲームプレイ |
|---|---|---|---|---|
| `MaxHealth` | 700–1500 | 試合が長くなりすぎ、逆転機会が増えすぎる | 必殺技1〜2発でKO。リスクとリターンのバランスが崩壊 | 全試合のテンポ・TTK |
| `WalkSpeed` | 300–500 cm/s | 間合いが瞬時に変わり、距離管理の意味が薄れる | 移動が遅すぎて間合いの引き出しが少なくなる | Pillar 1「間合いが全て」の体感 |
| `BackWalkSpeedMultiplier` | 0.6–0.85 | 逃げが容易になりすぎ、攻めにくい試合になる | 逃げられず追い詰めが簡単すぎる | ガードvs攻めの選択バランス |
| `ParryWindow` | 4–7 f | パリィが容易すぎて全攻撃を弾ける。攻め側のリターンが消滅 | 習得困難で実戦不使用になる。Pillar 3「逆転可能な緊張感」が失われる | **最敏感ノブ。1f単位で調整** |
| `ParryCounterAdvantage` | **12–18 f** | パリィ成功後の反撃が確定しすぎる（大ダメージが保証される） | パリィ成功しても反撃できない。報酬感が薄れる | パリィのリスク・リターン比 |
| `StartupFrames`（軽攻撃） | 4–8 f | 遅すぎて牽制として機能しない | 発生が速すぎてすかし困難。間合い管理の意味が薄れる | Pillar 1 の攻防テンポ |
| `StartupFrames`（重攻撃） | **8–11 f**（B8修正: 旧 8–18f） | 重攻撃が軽攻撃と差別化できない（発生差が縮まる） | 重攻撃が実戦で使えなくなる | ParryCounterAdvantage=12f の確定反撃保証と整合させるための制約。重攻撃の「重さ」は StartupFrames ではなくダメージ・ノックバックで表現する |
| `HitstunFrames` | **`max(RecoveryFrames + 1, StartupFrames + 3)` 以上**（形式制約: `RecoveryFrames + 1` / コンボ設計推奨: `StartupFrames + 3`、大きい方が実質下限）。重攻撃・特殊技では `RecoveryFrames` が大きくなるため形式制約が支配的になることに注意。※ `MoveCategory == Throw` および `MoveCategory == Special` の技はこの制約を適用しない（B3修正: Special は大きな RecoveryFrames を持てるため HitstunFrames 上限 40f との制約を免除） | コンボが長くなりすぎ、受け身感が消える | コンボが成立せず、ヒットの報酬感が薄い | コンボの深さと読み勝ちの報酬 |
| `BaseDamage`（軽攻撃） | 50–80 | 軽攻撃連打が強すぎる。重攻撃の存在意義が薄れる | 軽攻撃に意味がなく、全員重攻撃待ち | 攻撃種の役割分担 |
| `BaseDamage`（必殺技） | 200–280 | 必殺技が当たると逆転不可能。パリィの価値が下がる | 必殺技のリスクに見合わず使われない | 逆転劇の頻度と強さ |

**相互作用する組み合わせ（変更時は両方確認すること）**:

| 組み合わせ | 相互作用 |
|-----------|---------|
| `MaxHealth` ↔ `BaseDamage` | MaxHealth を上げた場合は BaseDamage も按分調整。TTK を一定に保つために比率を維持する |
| `ParryWindow` ↔ `ParryCounterAdvantage` | パリィウィンドウを広げるなら有利フレームを減らしてリスク・リターン比を保つ |
| `HitstunFrames` ↔ `StartupFrames`（次の技） | コンボが成立するかは `OnHitAdvantage > 次技のStartupFrames` で決まる。両方変えるとコンボルートが変わる |
| `WalkSpeed` ↔ ヒットボックスリーチ（`HalfExtent.X` + `LocalOffset.X`） | 移動速度が速くなるほど、技の有効リーチ内に相手が入りやすくなる。間合いの「駆け引き距離」が変わる |

## Visual/Audio Requirements

このシステムはデータ定義層（インフラ）であり、直接的なビジュアル/オーディオ要件を持たない。
ビジュアル/オーディオ表現は下流システムが担当し、このシステムはそのデータ参照先（`HitEffect`、`HitSoundCue`、`BlockSoundCue`）を定義するのみである。

関連するビジュアル要件は各依存システムの GDD を参照:
- ヒットエフェクトの視覚仕様 → 視覚フィードバックシステム GDD
- 打撃音・パリィ音の音響仕様 → 音響システム GDD

## UI Requirements

このシステムはデータ定義層であり、直接的な UI 要件を持たない。
UI は `CharacterTag`・`DisplayName`・`PrimaryColor` を読み取るが、表示方法は HUD システム GDD と キャラクター選択画面 GDD で定義する。

## Acceptance Criteria

### AC-1: データロードとライフサイクル

- **GIVEN** 有効な `UCharacterDataAsset` が Primary Asset Manager に登録されており、マッチ開始イベントが発火する、**WHEN** マッチ初期化シーケンスが完了する、**THEN** 両キャラクター分のアセットがいずれも非 `nullptr` で参照可能であり、マッチ初期化シーケンス完了コールバック（`AClashGameMode::OnMatchInitialized()`）が呼ばれた時点でロードが完了していること（ログに `[CLASH][AssetManager] CharacterData loaded: CharacterTag=%s` の形式で 2 件記録される）（B9修正: 「フレーム 0」の定義を `OnMatchInitialized()` コールバックに明確化、ログフォーマットを指定）
- **GIVEN** マッチ実行中に `UCharacterDataAsset` がロード済みである、**WHEN** マッチ開始フレームに `GetStats()` と `GetMoveList()` の全戻り値を記録し、ラウンド終了コールバック直前にも同じアクセサを呼び出す、**THEN** 対象フィールド（`FCharacterStats` の全スカラーフィールド + `MoveList` 全要素の `BaseDamage`・`StartupFrames`・`HitstunFrames`・`BlockstunFrames`・`ParryWindow`）の値がマッチ開始時と等値であること（読み取り専用の保証）
- **GIVEN** プロジェクトの C++ ソースを検索する、**WHEN** `MaxHealth`、`BaseDamage`、`StartupFrames` 等のゲームプレイ値を表すリテラル数値を探す、**THEN** いずれもハードコードされた数値リテラルが存在せず、必ず `UCharacterDataAsset` 経由で参照していること（B10修正: このACはCI静的解析ゲートとして分類する — 下記テスト証跡分類参照）

### AC-2: FGameplayTag 使用

- **GIVEN** `FMoveData` 構造体の定義を参照する、**WHEN** `MoveTag` フィールドの型を確認する、**THEN** 型が `FGameplayTag` であり `FName` または `FString` でないこと
- **GIVEN** C++ ソースの技・キャラクター識別コードを検索する、**WHEN** `FName` による比較・検索コードを探す、**THEN** 存在しないこと（すべて `FGameplayTag` を使用）

### AC-3: OnBlockAdvantage / OnHitAdvantage 計算（自動ユニットテスト必須）

- **GIVEN** `BlockstunFrames=12`、`RecoveryFrames=14` の技データがある、**WHEN** OnBlockAdvantage を算出する、**THEN** 結果が `-2` であること
- **GIVEN** `HitstunFrames=20`、`RecoveryFrames=14` の技データがある、**WHEN** OnHitAdvantage を算出する、**THEN** 結果が `+6` であること
- **GIVEN** `UCharacterDataAsset` の構造体定義を確認する、**WHEN** フィールド一覧を参照する、**THEN** `OnBlockAdvantage`・`OnHitAdvantage` という名前のフィールドが存在しないこと（派生計算値のため保存不要）（B10修正: この検証はコンパイル時チェックとして `static_assert(!std::is_member_pointer<decltype(&FMoveData::OnBlockAdvantage)>::value, ...)` の形で実装するか、または CI grep `grep -r "OnBlockAdvantage" --include="*.h"` で当該フィールド名が struct 宣言に含まれていないことを確認するCIゲートとして実装すること。実行時ユニットテストでの検証は不可）

### AC-4: HitstunFrames 不変条件（エディタ手動確認 — ADVISORY）

> **B-6修正**: `PostEditChangeProperty` は UE5 において保存をブロックできない（警告ログ出力・値修正のみ可能）。保存ブロックの正しいゲートは `IsDataValid(FDataValidationContext&)` である。エディタ UI での保存ブロック確認は **手動確認（ADVISORY）** とし、自動ユニットテストは AC-5 の `IsDataValid()` 呼び出しで担保する。

- **GIVEN** `HitstunFrames=10`、`BlockstunFrames=15` の `FMoveData` を含むアセットに対して Data Validation を実行する、**WHEN** エディタ「Asset → Validate Data」を実行する、**THEN** エラーが表示されること（手動確認）
- **GIVEN** `HitstunFrames=12`、`BlockstunFrames=12` の場合も同様にエラーが表示されること（等値は禁止）（手動確認）
- **GIVEN** `HitstunFrames=15`、`BlockstunFrames=12` の場合はエラーなし（手動確認）
- **追加（B-3）**: `HitstunFrames=10`、`RecoveryFrames=12` の場合（OnHitAdvantage=-2）も同様にエラーが表示されること

### AC-5: バリデーション — 保存ブロック（自動ユニットテスト必須）

テスト方法: C++ テストコードで `NewObject<UCharacterDataAsset>(TestPackage)` でアセットを生成し（outer = `UPackage` 必須で GC 保護する）、`IsDataValid(FDataValidationContext& Context)` を呼び出す。戻り値 `EDataValidationResult::Invalid` であることを確認する（B-6修正: UE5.1+ の正式 API は `IsDataValid(FDataValidationContext&)`。ClampMin はエディタ UI 制約であり、コードから直接構造体を構築した場合は動作しないため、`IsDataValid()` の実装で同等チェックを行うこと）。エラーメッセージフォーマット規約: `[ClashData][Error][FieldName]: Reason` の形式とする（R-5）。

- **GIVEN** `StartupFrames=0` の `FMoveData` を含む `UCharacterDataAsset` を構築する、**WHEN** `IsDataValid()` を呼び出す、**THEN** 戻り値が `EDataValidationResult::Invalid` であり、Context のエラーメッセージに `"StartupFrames"` が含まれること
- **GIVEN** `ParryWindow=0` の `FCharacterStats` を含む `UCharacterDataAsset` を構築する、**WHEN** `IsDataValid()` を呼び出す、**THEN** 戻り値が `EDataValidationResult::Invalid` であり、エラーメッセージに `"ParryWindow"` が含まれること
- **GIVEN** `ActiveFrameEnd=4`、`ActiveFrameStart=5` の `FHitboxData` を含む `FMoveData` を構築する（終了フレームが開始フレームより前）、**WHEN** `IsDataValid()` を呼び出す、**THEN** 戻り値が `EDataValidationResult::Invalid` であること（`ActiveFrameEnd < ActiveFrameStart` の制約違反 — B2修正: 旧テストケースは等値 `End=Start=5` をInvalidとしていたが、等値は1フレームヒットボックスとして正常。テストケースを `End=4 < Start=5` の真の無効ケースに変更）
- **GIVEN** `ActiveFrameEnd=5`、`ActiveFrameStart=5` の `FHitboxData` を含む `FMoveData` を構築する（1フレームヒットボックス）、**WHEN** `IsDataValid()` を呼び出す、**THEN** 戻り値が `EDataValidationResult::Valid` であること（等値は1フレームヒットボックスとして許容される正常ケース）
- **追加（B-3）**: `HitstunFrames=10`、`RecoveryFrames=15`、`MoveCategory=LightAttack` の `FMoveData` を構築する、**WHEN** `IsDataValid()` を呼び出す、**THEN** 戻り値が `EDataValidationResult::Invalid` であること（OnHitAdvantage = -5 = 負値禁止。LightAttack は Special/Throw 特例の対象外）
- **追加（B-3 Special 特例）**: `HitstunFrames=10`、`RecoveryFrames=50`、`MoveCategory=Special` の `FMoveData` を構築する（リスクの高い必殺技）、**WHEN** `IsDataValid()` を呼び出す、**THEN** 戻り値が `EDataValidationResult::Valid` であること（Special 特例: OnHitAdvantage=-40 でも許容）
- **追加（B-5）**: `RangeTag` が空の `FMoveData` を構築する、**WHEN** `IsDataValid()` を呼び出す、**THEN** 戻り値が `EDataValidationResult::Invalid` であること（Move.Range.* タグ必須）

### AC-6: バリデーション — ロード失敗（統合テスト必須）

> **テスト方法（B-6/B-7修正、B10修正）**: このテストは Asset Manager の完全初期化が必要なため、Unreal Editor の **Editor Automation Test**（`IMPLEMENT_COMPLEX_AUTOMATION_TEST`）として実装すること（`-nullrhi` ヘッドレスの Unit Test では Asset Manager が完全に初期化されないため不可）。
>
> **B10修正: エスケープハッチを廃止** — 「`NewObject<>()` + `IsDataValid()` の直接呼び出しで代替可能」という従来の記述を削除する。AC-5 のバリデーションロジックテストと AC-6 のロードパステストは異なる目的を持ち、後者を前者で代替することは AC-6 の BLOCKING 要件を無効化するため不可。Asset Manager の実際のロードパス検証が技術的に困難な場合は、このテストを **BLOCKING → ADVISORY（手動 PlayTest エビデンス）に格下げ** し、`production/qa/evidence/ac6-load-test-[date].md` に手動確認記録を残すことで要件を満たすこと。格下げを選択する場合は AC テスト証跡分類テーブルを更新すること。
>
> **AC-6.2 フィクスチャ（B-8修正）**: 未登録 GameplayTag のテストは「`DefaultGameplayTags.ini` に存在しないタグ文字列を `UGameplayTagsManager::Get().RequestGameplayTag(Name, false)` で取得し `IsValid()=false` を確認」する方式で実装する。

- **GIVEN** `MoveList` が空配列の `UCharacterDataAsset` に対して `IsDataValid()` を呼び出す、**WHEN** バリデーションを実行する、**THEN** `EDataValidationResult::Invalid` が返されること
- **GIVEN** 未登録 GameplayTag を `MoveTag` に持つアセットの `IsDataValid()` を呼び出す、**WHEN** バリデーションを実行する、**THEN** `Invalid` が返されること（未登録タグは `RequestGameplayTag(Name, false).IsValid()=false` で検出）
- **GIVEN** `MoveCategory=LightAttack` かつ `Hitboxes` が空の技を含むアセットの `IsDataValid()` を呼び出す、**WHEN** バリデーションを実行する、**THEN** `Invalid` が返されること
- **GIVEN** `MoveCategory=Parry` かつ `Hitboxes` が空の技を含む（他フィールドは有効な）アセットの `IsDataValid()` を呼び出す、**WHEN** バリデーションを実行する、**THEN** `EDataValidationResult::Valid` が返されること

### AC-7: 下流システムへのデータインターフェース（統合テスト必須 — **条件付き BLOCKING**）

> **ゲート条件（R-6修正）**: 以下のテストは依存システムの GDD が承認された時点でアクティブ化される。各システムの GDD 承認時に、当該 GDD の Acceptance Criteria にこの AC を統合テストとして転記すること（責任移管先ファイルを明示）。
> - AC-7.1 → 体力システム GDD 承認後（移管先: `design/gdd/health-system.md` の AC セクション）
> - AC-7.2 → キャラクター移動システム GDD 承認後（移管先: `design/gdd/character-movement.md` の AC セクション）
> - AC-7.3 → ヒット判定システム GDD 承認後（移管先: `design/gdd/hit-detection.md` の AC セクション）
> - AC-7.4 → パリィシステム GDD 承認後（移管先: `design/gdd/parry-system.md` の AC セクション）

- **GIVEN** `MaxHealth=1000` のアセットがロードされた状態でマッチを開始する、**WHEN** 体力システムが初期 HP をセットする、**THEN** キャラクターの初期 HP が `1000` であること
- **GIVEN** `WalkSpeed=350`、`BackWalkSpeedMultiplier=0.7` のアセットがロードされた状態で後退入力する、**WHEN** 移動システムが後退速度を計算する、**THEN** 後退速度が `245 cm/s`（= 350 × 0.7）であること
- **GIVEN** `StartupFrames=6` の技が入力フレーム 0 から発動する、**WHEN** フレーム 5 のヒット判定を実行する、**THEN** ヒットボックスが非アクティブであること；**WHEN** フレーム 6 のヒット判定を実行する、**THEN** ヒットボックスがアクティブであること
- **GIVEN** `ParryWindow=5` のアセットがロードされた状態でパリィ入力する、**WHEN** 入力から 5 フレーム以内に攻撃が接触する、**THEN** パリィが成立すること；**WHEN** 6 フレーム目以降に攻撃が接触する、**THEN** パリィが成立しないこと

### テスト証跡分類

| AC グループ | ストーリータイプ | 必要な証跡 | ゲートレベル |
|---|---|---|---|
| AC-3, AC-5 | Logic（フォーミュラ・バリデーション） | `tests/unit/character-data/` 自動ユニットテスト | **BLOCKING** |
| AC-4 | エディタ手動確認（IsDataValid UI） | `production/qa/evidence/` 手動確認ログ | ADVISORY（B-6修正） |
| AC-6 | Integration（Asset Manager ロード検証） | `tests/integration/character-data/` Editor Automation Test（`IMPLEMENT_COMPLEX_AUTOMATION_TEST`）。技術的困難な場合は ADVISORY に格下げ可（B10修正: エスケープハッチ廃止、ただし格下げオプションあり） | **BLOCKING**（または ADVISORY — AC-6 テスト方法注記参照） |
| AC-7 | Integration（下流システム連携） | `tests/integration/[system]/` — 依存 GDD 承認後に各システムのストーリーで実装 | **条件付き BLOCKING** |
| AC-1.1, AC-1.2 | Integration（ロードライフサイクル・不変性） | `tests/integration/character-data/` またはスモークチェック | ADVISORY |
| AC-1.3（ハードコード禁止）、AC-2.2（FName 禁止） | 静的解析 | CI grep ルール（`coding-standards.md` の Forbidden Patterns に追加）（B10修正: AC-1.3 を Config/Data ADVISORY から CI ゲートに再分類） | **CI ゲート** |
| AC-2.1 | Config/Data | スモークチェック通過 | ADVISORY |

## Open Questions

| # | 質問 | オーナー | 解決時期 |
|---|------|---------|---------|
| OQ-1 | **固定タイムステップの実装方式**: 格闘ゲームに必要な 60fps フレーム精度を UE5.7 で実現するための実装アプローチ（`UFightingGameSubsystem` の蓄積型ループ vs. Project Settings の Fixed Frame Rate）。→ **ADR として記録すること** | 開発者 / エンジンエンジニア | 入力システム GDD 設計前 |
| OQ-2 | **2キャラクターの具体的なスタッツ差**: Brawler と Striker（仮称）のスタッツをどのように非対称にするか（Pillar 1 の間合いゲームを豊かにするため）。→ バランスシートはこの GDD ではなくキャラクターデザインドキュメントで管理 | ゲームデザイナー | Alpha ティア |
| OQ-3 | **必殺技のコマンド入力版への拡張**: `FMoveData.InputAction` は MVP では 1 ボタンアクションを指すが、Vertical Slice では `InputSequence`（コマンド入力）に変わる。このフィールドの型変更が発生するか、それとも別フィールドを追加するか。→ コマンド入力シーケンス検出 GDD 設計時に解決 | システムデザイナー | VS ティア |
| OQ-4 | **無敵フレーム・アーマー状態のスキーマ追加（VS+スコープ）**: 特殊技の切り返し（リバーサル）に必要な無敵フレーム（`InvincibilityFrameStart/End`）と重攻撃の打ち合い強化に必要なアーマー状態（`ArmorStartFrame/EndFrame`）は VS+ スコープとして延期。FMoveData を拡張し、ヒット判定システム GDD との整合が必要（MoveCategory==Throw の IsBlockable 判定と同様の特例処理が必要）。これらが未実装の MVP では Pillar 3「逆転可能な緊張感」は主にパリィシステムのみで担保する。 | ゲームデザイナー / システムデザイナー | VS ティア |
