# saitousan-docs

斉藤さん / 斉藤さんLIVE に関する技術調査、ADR、PoC計画をMarkdownで管理するドキュメント基盤。

## Contents
- `research/`: 技術調査メモ
- `adr/`: Architecture Decision Records
- `architecture/`: 構成図とアーキテクチャメモ
  - `aws-youtube-to-saitousan-live.md`: 現行のYouTube一次配信構成
  - `aws-webrtc-mediamtx-fanout.md`: スマホWebRTC + mediaMTX fan-out案（ADR-0021）

## ADR Flow
1. 課題や選択肢を整理する。
2. `adr/0000-template.md` を使ってADRを書く。
3. `Status` を `Proposed` で作成する。
4. 決定後に `Accepted` へ更新する。
5. 方針変更時は既存ADRを書き換えず、新しいADRで上書き判断を残す。

## Current ADRs
- `ADR-0001`: 調査結果サイトを静的HTMLで管理する
- `ADR-0002`: YouTube配信を斉藤さんLIVEへ中継するラッパー構成
- `ADR-0004`: 斉藤さんLIVEの映像をYouTubeへミラーリングする案 rejected
- `ADR-0005`: 斉藤さんコメントを名前付きでYouTubeへフィードバックする
- `ADR-0006`: クラウドコストは段階的PoCの成立後に拡大する proposed
- `ADR-0007`: SlackとCodexを接続した開発タスク依頼・PR作成ワークフロー proposed
- `ADR-0008`: Android SDK/Emulator用EC2 Runtime Host候補とOS方針 proposed
- `ADR-0009`: ADR-0002の一次配信元は当面YouTubeを維持する proposed（ADR-0021で再評価中）
- `ADR-0010`: 開発初期の主導線はUIではなくCLIとprotoにする proposed
- `ADR-0011`: OTPログインは初期段階では人手ゲートとして扱う proposed
- `ADR-0012`: 初期PoCのアプリ導入はPlay Storeより検証済みAPK系artifactを優先する proposed
- `ADR-0013`: 配信時間入力は目標時間にし、固定枠選択は内部で吸収する proposed
- `ADR-0014`: Android向けエージェント操作の初期検証はAppium MCPを第一候補にする proposed
- `ADR-0015`: 配信開始設定は型付きCLI/API契約として先に固定する proposed
- `ADR-0016`: 配信開始導線の最小PoCはホーム画面から中継開始設定画面到達までに限定する proposed
- `ADR-0017`: 初回起動とアカウント切り替えはOTP本体とは別の事前状態として扱う proposed
- `ADR-0018`: 同一アカウントの配信系ジョブは直列実行に制限する proposed
- `ADR-0019`: YouTubeのミラー元はLive Control Roomではなくwatch pageにする
- `ADR-0020`: YouTube watch page入力を画質固定とtelemetryで安定化する
- `ADR-0021`: スマホWebRTC起点 + mediaMTX fan-out へ一次配信元を切り替える proposed

## Local Usage
MarkdownファイルをエディタまたはGitHub上で読む。
