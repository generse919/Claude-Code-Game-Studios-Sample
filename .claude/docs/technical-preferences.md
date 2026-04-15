# Technical Preferences

<!-- Populated by /setup-engine. Updated as the user makes decisions throughout development. -->
<!-- All agents reference this file for project-specific standards and conventions. -->

## Engine & Language

- **Engine**: Unreal Engine 5.7
- **Language**: C++ (primary), Blueprint (gameplay prototyping)
- **Rendering**: Deferred Rendering, Lumen GI, Nanite (use selectively — primitives won't need Nanite)
- **Physics**: Chaos Physics (UE5 default)

## Input & Platform

<!-- Written by /setup-engine. Read by /ux-design, /ux-review, /test-setup, /team-ui, and /dev-story -->
<!-- to scope interaction specs, test helpers, and implementation to the correct input methods. -->

- **Target Platforms**: PC (Windows)
- **Input Methods**: Keyboard/Mouse, Gamepad
- **Primary Input**: Gamepad
- **Gamepad Support**: Partial
- **Touch Support**: None
- **Platform Notes**: デスクトップ対戦想定。ゲームパッド推奨だがキーボード分割対戦も対応必須。ホバー専用インタラクション禁止。

## Naming Conventions

- **Classes**: Prefixed PascalCase — `A` for Actor, `U` for UObject, `F` for struct, `I` for Interface (例: `APlayerFighter`, `UCombatComponent`, `FAttackData`)
- **Variables**: PascalCase (例: `MoveSpeed`, `CurrentHealth`)
- **Booleans**: `b` prefix (例: `bIsAlive`, `bCanParry`)
- **Functions**: PascalCase (例: `TakeDamage()`, `TriggerParry()`)
- **Files**: Match class name without prefix (例: `PlayerFighter.h`, `CombatComponent.cpp`)
- **Blueprints**: PascalCase with `BP_` prefix (例: `BP_PlayerFighter`, `BP_FightingGameMode`)
- **Constants**: PascalCase or UPPER_SNAKE_CASE

## Performance Budgets

- **Target Framerate**: 60 fps
- **Frame Budget**: 16.6ms
- **Draw Calls**: 1,000 上限（格闘ゲームの単純シーン向け — 2キャラ + プリミティブステージ）
- **Memory Ceiling**: 8GB RAM / 4GB VRAM

## Testing

- **Framework**: Unreal Automation System (FAutomationTestBase)
- **Minimum Coverage**: ゲームロジック（体力計算・ラウンド判定）、コマンド入力検出、パリィ判定タイミング
- **Required Tests**: バランス式（ダメージ計算）、ゲームプレイシステム（ラウンド勝敗）、入力シーケンス検出

## Forbidden Patterns

<!-- Add patterns that should never appear in this project's codebase -->
- [None configured yet — add as architectural decisions are made]

## Allowed Libraries / Addons

<!-- Add approved third-party dependencies here -->
- [None configured yet — add as dependencies are approved]

## Architecture Decisions Log

<!-- Quick reference linking to full ADRs in docs/architecture/ -->
- [No ADRs yet — use /architecture-decision to create one]

## Engine Specialists

<!-- Written by /setup-engine when engine is configured. -->
<!-- Read by /code-review, /architecture-decision, /architecture-review, and team skills -->
<!-- to know which specialist to spawn for engine-specific validation. -->

- **Primary**: unreal-specialist
- **Language/Code Specialist**: ue-blueprint-specialist (Blueprint graphs) または unreal-specialist (C++)
- **Shader Specialist**: unreal-specialist (専用シェーダースペシャリストなし — Primaryがマテリアルをカバー)
- **UI Specialist**: ue-umg-specialist (UMG ウィジェット、CommonUI、入力ルーティング、ウィジェットスタイリング)
- **Additional Specialists**: ue-gas-specialist (Gameplay Ability System、アトリビュート、ゲームプレイエフェクト)、ue-replication-specialist (プロパティレプリケーション、RPC、クライアント予測、ネットコード)
- **Routing Notes**: C++ アーキテクチャと広範なエンジン決定にはPrimaryを使用。Blueprint グラフアーキテクチャとBP/C++境界設計にはBlueprint スペシャリストを使用。GASはすべてのアビリティとアトリビュートコードに使用。レプリケーションスペシャリストはマルチプレイヤーまたはネットワークシステムに使用。UMGスペシャリストはすべてのUI実装に使用。

### File Extension Routing

<!-- Skills use this table to select the right specialist per file type. -->

| File Extension / Type | Specialist to Spawn |
|-----------------------|---------------------|
| Game code (.cpp, .h files) | unreal-specialist |
| Shader / material files (.usf, .ush, Material assets) | unreal-specialist |
| UI / screen files (.umg, UMG Widget Blueprints) | ue-umg-specialist |
| Scene / prefab / level files (.umap, .uasset) | unreal-specialist |
| Native extension / plugin files (Plugin .uplugin, modules) | unreal-specialist |
| Blueprint graphs (.uasset BP classes) | ue-blueprint-specialist |
| General architecture review | unreal-specialist |
