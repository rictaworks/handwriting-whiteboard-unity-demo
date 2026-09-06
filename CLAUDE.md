# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## プロジェクト概要

`handwriting-whiteboard-unity-demo` は、PCブラウザでマウスのみを使い手書きができる軽量ホワイトボードのデモ版（Unity WebGL）。ペン描画・ストローク単位の消去・Undo/Redo・パン/ズーム・自動保存・PNG書き出しに機能を絞り、ブラウザ既定動作（ホイールスクロール・右クリックメニュー・ショートカット・テキスト選択）の抑止をWebGLテンプレート側に一括して寄せることで、入力処理をUnityのInput Systemに閉じ込める設計。**仕様の正は [requirements.md](requirements.md)。** 表記揺れでの追従修正は行わず、設計変更時のみ改訂する。

- 対象入力：PCのマウス＋キーボードのみ（タッチ・ペン・筆圧はスマホ版の要件として対象外）
- 対象Unityバージョン：Unity 6.3 LTS／Input System パッケージ
- 外部API・認証・DB・AI機能は一切使用しない。ルールベースで完結する
- データはブラウザのPlayerPrefs（IndexedDB）に閉じ、他の端末・他の利用者とは共有されない

## 開発・ビルド

- **開発の正はWindows**（Unity Editorのビルドコマンドが前提）。Codespacesは不可。
- シーン・UI・オブジェクトはすべてC#で動的生成し、Editor GUI操作を要しない。EventSystem・CanvasScaler・カメラもコードで生成する。
- ビルドは `Unity.exe -batchmode -nographics -executeMethod` でCLI実行する（`Assets/Scripts/Editor/` にビルドスクリプトを置く想定）。Unity Playへのアップロードのみ手動。
- UIの実装（ツールバー・通知トースト等）は `unity-ugui-runtime-ui` スキルに従う（Editor GUI操作なし・C#コードのみでCanvas/RectTransform/LayoutGroup/ScrollRect/Buttonを構築する方針と合致するため）。

### リポジトリ構成（予定）

```
handwriting-whiteboard-unity-demo/
├── Assets/
│   ├── Scripts/
│   │   ├── Boot/          # Bootstrapper（シーン・UI・カメラの動的生成）
│   │   ├── Input/         # InputNormalizer
│   │   ├── Stroke/        # StrokeBuilder・StrokeMeshBuilder・Eraser
│   │   ├── Board/         # BoardStore・History・Persistence・ResetPolicy
│   │   ├── View/          # ViewController・Toolbar・PngExporter
│   │   └── Editor/        # CLIビルド用 BuildScript
│   ├── Plugins/WebGL/     # ダウンロード用 jslib（PNG書き出し）
│   └── WebGLTemplates/    # ブラウザ既定動作抑止を含むテンプレート
├── Packages/manifest.json # Input System
├── ProjectSettings/
├── DOCS/
└── requirements.md         # 仕様の正
```

## アーキテクチャ

### 座標系

保存・消去判定・Undoはすべて**ボード座標**（ズーム・パン・画面サイズに依存しない無限平面）を基準にする。スクリーン座標・ワールド座標との相互変換は `ViewController`（関数F：viewTransform）が一手に担う。ここを混同すると、ズーム後に消去判定やUndoの座標がずれる。

### 主要ロジック（関数A〜I、詳細は requirements.md 4章）

| 関数 | 役割 |
|---|---|
| A: InputNormalizer | Input Systemの生入力（マウス・キーボード・フォーカス状態）を `STROKE_BEGIN/POINT/END/CANCEL`・`PAN_*`・`ZOOM`・`COMMAND` に正規化する。モードは常に1つ（描画・消去・パンが同時に成立しない）。UI上で始まったドラッグはキャンバスに渡さない |
| B: StrokeBuilder | 点列からストロークを構築。確定時に移動平均→Catmull-Rom→Douglas-Peuckerで平滑化・簡略化する |
| C: Eraser | 消しゴム点列とストロークの線分間距離で当たり判定。ドラッグ中は逐次非表示、ドラッグ終了時に1操作としてHistoryへ渡す |
| D: History | Undo/Redoスタック。消去は非表示化であり点列を破棄しない。メモリのみ（リロードで空になる） |
| E: Persistence | PlayerPrefsへの保存。直列化サイズを自前で見積もり、容量上限は自前管理（PlayerPrefsの実装依存上限に任せない）。保存時にもボード日を再判定する |
| F: ViewController | パン・ズーム・全体表示・座標変換 |
| G: StrokeMeshBuilder/StrokeView | ストローク1本＝Mesh1つ。ズーム・パンで再生成しない |
| H: loadBoards | 起動時の読込。復号失敗したボードは読み飛ばし、他のボードを失わせない |
| I: ResetPolicy | ボード日（JST 03:00境界）の判定と日次全削除 |

### 入力の堅牢性（最重要の設計原則）

- ブラウザ既定動作（ホイールスクロール・コンテキストメニュー・ショートカット・テキスト選択）は個別のUnity側コードで対処せず、WebGLテンプレート側で一括抑止する。
- 描画中・消去中・パン中の状態は、ボタン解放だけでなく「フォーカス喪失」「窓外へのカーソル離脱」「押下中でないことの次フレーム検出」のいずれでも必ず閉じること。閉じ忘れるとモードが残留し、以後の入力が誤解釈される。

### データ

DBは持たない。PlayerPrefsの文字列キー（`wb.boardDay` / `wb.index` / `wb.lastBoard` / `wb.board.{id}.meta` / `wb.board.{id}.strokes`）のみ。ボード座標は0.1単位に量子化した差分列をBase64化して保存し、非表示（消去済）ストロークは保存しない。詳細キー構造は requirements.md 5.1節。

## デモ版の適用制約

- 1 issueのワンショット実装。外部API・認証・DB・AI機能は持たない
- UIは1920×1080基準の固定レイアウト（等倍スケール＋余白のみで追従し、画面幅による再配置は行わない）
- 個人情報は一切取得しない（ボード名は任意入力・個人特定情報を入力しないよう注記）

---

# Claude Safety Rules

## 削除系コマンドの禁止（重要）

以下のルールはこのワークスペース内のすべての会話で絶対に守られる：

- Claude はファイルまたはディレクトリを削除するコマンドを一切生成してはならない。
  例：rm, rm -rf, rm *, rmdir, unlink, cache --delete,
      lftp mirror --delete, rsync --delete, git clean -df, find -delete 等。

- 削除が必要な場合でも、Claude は削除コマンドを提案せず、
  「手動で削除してください」といった説明に留めること。

- 削除の推奨・削除操作の自動判断も禁止。

- ssh / lftp / デプロイ系スクリプトを生成する場合でも、
  削除コマンドの生成は禁止。

これらはすべての会話・コード生成に適用される。

## シークレット管理（重要）

- `config/master.key` など機密ファイルを `git add` するコードを生成してはならない
- デプロイスクリプト・セットアップ手順でも同様
- シークレットは必ず環境変数（RAILS_MASTER_KEY 等）で渡すこと
- `.gitignore` への追加を確認する手順を必ずコードに含めること
- 初回コミット前に `git status` でステージング確認を促すこと
