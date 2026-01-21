# キル・デス管理システム実装計画

## ゴール
プレイヤーの名前、キル数、デス数を管理し、ゲーム内で確認できるようにする。
BPとC++を併用し、拡張性とパフォーマンスを考慮した設計とする。

## ユーザーレビューが必要な項目
- **クラス構成**: 既存のBlueprintのみの構成から、C++の基底クラス (`PlayerState`, `GameMode`) を導入します。
- **データ保存場所**: 個々のプレイヤーの戦績は `PlayerState` に保存し、サーバーから全クライアントへレプリケーション（同期）させます。

## 変更内容プロポーザル

### C++ Classes (Source/FPS)

#### [NEW] AFPSPlayerState
- `APlayerState` を継承。
- プロパティ:
  - `int32 KillCount` (Replicated)
  - `int32 DeathCount` (Replicated)
- 関数:
  - `GetLifetimeReplicatedProps`: 変数のレプリケーション設定。
  - `AddKill()`: キル数加算（サーバーのみ実行可能）。
  - `AddDeath()`: デス数加算（サーバーのみ実行可能）。

#### [NEW] AFPSGameMode
- `AGameModeBase` を継承。
- 関数:
  - `ProcessPlayerKilled(AController* Killer, AController* Victim)`:
    - キラーがいる場合、キラーの `AFPSPlayerState->AddKill()` を呼ぶ。
    - ビクティムの `AFPSPlayerState->AddDeath()` を呼ぶ。

### Blueprints (Content/FPS/Blueprints)

#### [MODIFY] BP_GameMode (or Create new)
- 親クラスを `AFPSGameMode` に変更。

#### [CREATE] BP_PlayerState
- 親クラスを `AFPSPlayerState` に設定。

#### [MODIFY] BP_FirstPersonCharacter (Player Character)
- 死亡時処理 (`AnyDamage` -> `Health <= 0`) にて、`GameMode` の `ProcessPlayerKilled` を呼び出す処理を追加。

### UI

#### [CREATE] WBP_Scoreboard
- `GameState` の `PlayerArray` を取得し、各プレイヤーの `PlayerName`, `KillCount`, `DeathCount` をリスト表示するウィジェット。

## 検証計画

### 自動テスト
- 現状プロジェクトにテストフレームワークが導入されていないため、手動検証を行う。

### 手動検証
1. **エディタでのマルチプレイ実行**:
   - "Net Mode" を "Play as Listen Server" に設定。
   - "Number of Players" を 2 に設定。
2. **キルログ確認**:
   - Server側でClientをキルする -> ServerのKill増、ClientのDeath増を確認。
   - Client側でServerをキルする -> ClientのKill増、ServerのDeath増を確認。
3. **UI確認**:
   - Tabキー（または指定キー）でスコアボードが表示され、正しい数値が同期されているか確認。
