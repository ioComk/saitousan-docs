# ADR運用

このフォルダーでは、開発や調査サイト作成に関する意思決定をADRとして残す。

## 書き方
1. `0000-template.md` をコピーする。
2. 連番で `0002-title.md` のように作る。
3. `Status` は `Proposed`、`Accepted`、`Deprecated`、`Superseded` のどれかにする。
4. 決定理由だけでなく、捨てた選択肢とデメリットも書く。

## 基本フロー
1. 調査または要件を整理する。
2. 選択肢を2-3個出す。
3. ADRに決定を書く。
4. その決定に沿って実装やHTML更新を行う。
5. 方針が変わったら古いADRを書き換えず、新しいADRで上書き判断を残す。


## ADR一覧

- [ADR-0001: Static HTML Research Site](0001-static-html-research-site.md)
- [ADR-0002: YouTube配信を斉藤さんLIVEへ中継するラッパー構成](0002-youtube-to-saitousan-live-wrapper.md)
- [ADR-0004: Saitousan LIVE to YouTube Mirror](0004-saitousan-live-to-youtube-mirror.md)
- [ADR-0005: Saitousan Comment to YouTube](0005-saitousan-comment-to-youtube.md)
- [ADR-0006: クラウドコストは段階的PoCの成立後に拡大する](0006-cloud-cost-phased-poc.md)
- [ADR-0007: SlackとCodexを使った開発ワークフロー](0007-slack-codex-development-workflow.md)
- [ADR-0008: Android SDK/Emulator用EC2 Runtime Host候補とOS方針](0008-android-runtime-ec2-host-candidates.md)
- [ADR-0009: ADR-0002の一次配信元は当面YouTubeを維持する](0009-adr-0002-owned-source-rejected-for-initial-phases.md)
- [ADR-0010: 開発初期の主導線はUIではなくCLIとprotoにする](0010-cli-and-proto-first-control-surface.md)
- [ADR-0011: OTPログインは初期段階では人手ゲートとして扱う](0011-otp-login-is-a-human-gated-boundary.md)
- [ADR-0012: 初期PoCのアプリ導入はPlay Storeより検証済みAPK系artifactを優先する](0012-pinned-apk-artifacts-over-play-store-for-poc.md)
- [ADR-0013: 配信時間入力は目標時間にし、固定枠選択は内部で吸収する](0013-target-duration-maps-to-fixed-live-slots.md)
- [ADR-0014: Android向けエージェント操作の初期検証はAppium MCPを第一候補にする](0014-appium-mcp-first-for-mobile-agent-evaluation.md)
- [ADR-0015: 配信開始設定は型付きCLI/API契約として先に固定する](0015-stream-start-configuration-contract.md)
- [ADR-0016: 配信開始導線の最小PoCはホーム画面から中継開始設定画面到達までに限定する](0016-open-stream-setup-from-home-screen.md)
- [ADR-0017: 初回起動とアカウント切り替えはOTP本体とは別の事前状態として扱う](0017-login-entry-and-account-switch-are-explicit-prelogin-states.md)
- [ADR-0018: 同一アカウントの配信系ジョブは直列実行に制限する](0018-serialize-stream-jobs-per-account.md)
- [ADR-0019: YouTubeのミラー元はLive Control Roomではなくwatch pageにする](0019-youtube-watch-page-over-live-control-room-capture.md)
- [ADR-0020: YouTube watch page入力を画質固定とtelemetryで安定化する](0020-youtube-watch-page-input-stabilization.md)
- [ADR-0021: スマホWebRTC起点 + mediaMTX fan-out へ一次配信元を切り替える](0021-webrtc-mediamtx-owned-ingest-fanout.md)

## 運用ルール

- このプロジェクトで「検討して」と依頼された場合は、基本的にADRとして残す。
- まだ決定していない検討は `Status: Proposed` にする。
- ユーザーが明示的に採用を指示した、または既に合意済みの方針は `Status: Accepted` にする。
- 一時的な実験ログや実測結果はADRではなく `research/validation-log.md` に記録する。
