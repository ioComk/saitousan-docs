# ADR-0021: スマホWebRTC起点 + mediaMTX fan-out へ一次配信元を切り替える

## Status
Proposed

## Context
現行のADR-0002 / ADR-0009では、一次配信元をYouTube Live（または固定テスト映像）とし、その映像をAndroid仮想カメラ経由で斉藤さんLIVEへ流す構成を採っている。

この経路は、2026-07-08時点のPoCで「YouTube watch page → v4l2 → Emulator → 斉藤さんLIVE camera枠」までの映像注入がフレンド限定で成立した。一方で、次の課題が残っている。

- YouTube watch page取得はbot/sign-in、buffer stall、画質変動に依存する（ADR-0019 / ADR-0020）
- YouTubeを起点にすると、遅延と障害切り分けがYouTube再生側に引っ張られる
- 音声注入は未解決のまま
- 「スマホで撮った映像を、YouTubeと斉藤さんLIVEの両方へ出す」という運用意図に対し、YouTube一次配信は回り道になっている

このため、一次配信元を自前化し、次の構成へ切り替える案を再評価する。

```text
スマホ（映像・音声）
    |
    v  WebRTC (WHIP)
EC2上の mediaMTX
    |
    +--> RTMP/RTMPS --> YouTube Live
    +--> ローカル出力 --> 既存Android bridge --> 斉藤さんLIVE
```

これはADR-0009が「将来の拡張候補」として棚上げした自前起点案の具体化である。ADR-0009の再評価条件のうち、Androidカメラ注入の映像経路は短時間PoCとして成立済みである。音声と長時間安定性は未達だが、YouTube watch page依存そのものがボトルネックになり始めているため、構成見直しの議論を開始する。

## Decision
一次配信元をYouTubeから切り離し、**スマホ → WebRTC → EC2上のmediaMTX** をsource of truthとする方針を、次フェーズの採用候補にする。

mediaMTXはingest hubとして次を担う。

1. スマホからのWebRTC ingest（推奨: WHIP）
2. YouTube LiveへのRTMP/RTMPS push
3. 斉藤さんLIVE向けのローカル再出力（RTSP / WebRTC / HLS など）

斉藤さんLIVEへの到達経路は、公式の外部ingest APIが公開されていない前提を維持する。mediaMTXから斉藤さんへ直接pushせず、既存の「仮想カメラ → Android Runtime → 斉藤さんアプリ」bridgeを継続する。

責務境界は次の通りとする。

| Layer | Responsibility | Source of truth |
| --- | --- | --- |
| Phone publisher | カメラ/マイク取得、WebRTC送信 | 撮影現場 |
| mediaMTX on EC2 | ingest、再エンコード、fan-out | 配信ハブ |
| YouTube Live | 録画・アーカイブ・公開視聴 | 出力先A |
| Android bridge + 斉藤さんアプリ | アプリUI操作とLIVE投稿 | 出力先B |
| Control plane | 開始/停止、鍵管理、状態監視 | 運用制御 |

ADR-0002の目的「同じ映像をYouTube側に残しつつ斉藤さんLIVEにも流す」は維持する。変わるのは、YouTubeが入力元ではなく出力先の一つになる点である。

このADRは `Proposed` とする。Acceptedへ上げる前に、少なくとも次を確認する。

- スマホ → mediaMTX のWebRTC ingestが安定する
- mediaMTX → YouTube RTMP が非公開/限定公開で成立する
- mediaMTXローカル出力 → 既存v4l2/Emulator経路で斉藤さんcamera枠に映る
- 音声がYouTubeと斉藤さん側の少なくとも一方で成立する、または未達理由が定量化される

## Proposed Architecture

```text
[Smartphone App / Browser]
        |  WebRTC (WHIP publish)
        v
[EC2: mediaMTX]
        |
        +---- RTMP/RTMPS ----> [YouTube Live Ingest]
        |
        +---- RTSP/WebRTC/HLS -> [FFmpeg / v4l2loopback]
                                      |
                                      v
                               [Android Emulator / Device]
                                      |
                                      v
                               [斉藤さん App]
                                      |
                                      v
                               [斉藤さんLIVE]
```

制御面は既存方針（ADR-0010のCLI/proto先行）を踏襲し、最初から管理UIを必須にしない。

```text
operator / CLI
    |
    v
control API or scripts
    |
    +--> mediaMTX path/auth/publish URL 発行
    +--> YouTube stream key 注入（Secrets Manager想定）
    +--> Android / Appium 配信開始・停止
    +--> health: ingest connected, YouTube push OK, camera preview OK
```

## Component Responsibilities

| Component | Responsibility | First PoC Requirement |
| --- | --- | --- |
| Phone publisher | 映像・音声をWebRTCで送る | ブラウザWHIPまたは既存配信アプリで可 |
| mediaMTX | WHIP ingestと多重出力 | EC2単一インスタンスでよい |
| YouTube push | RTMP出力 | 非公開/限定公開Live |
| Local remux | mediaMTX出力をv4l2へ渡す | 既存FFmpegスクリプトを流用 |
| Android bridge | 斉藤さんcamera入力 | ADR-0002で成立済みの経路を再利用 |
| Secrets | YouTube stream key、WHIP token | 平文をrepoへ置かない |
| Observability | ingest切断、push失敗、黒画面検知 | 最低限のログとfail-closed |

## Goals
- スマホ撮影を一次ソースにし、YouTubeと斉藤さんLIVEへ同時配信できる形にする。
- YouTube watch page capture依存（stall、bot、画質変動）を配信主経路から外す。
- YouTube側の録画・アーカイブ利点は維持する。
- 斉藤さん向けは既存Android bridgeを壊さず接続する。

## Non-Goals
- 斉藤さん公式APIや非公式プロトコル解析による直接配信。
- 最初からマルチリージョン、オートスケール、大規模同時配信基盤を作ること。
- mediaMTX以外の配信サーバー製品比較を本ADRで完了すること。
- 管理画面の本実装。
- 音声品質の最終チューニング。

## Consequences
YouTubeを入力ではなく出力に回すため、watch page取得の不安定さから主経路を切り離せる。遅延も「YouTube再生待ち → 再注入」より短くなる見込みがある。

一方で、責務は増える。

- スマホ側publisherの実装または選定
- EC2上mediaMTXの運用（証明書、TURN/ICE、ポート、認証）
- YouTube stream keyの安全な注入
- mediaMTX出力と既存v4l2経路の接続
- 片方の出力だけ落ちたときの部分障害ハンドリング

ADR-0009が懸念した「制御対象の増加」は現実に起きる。ただし現状のYouTube起点でもChromium、Xvfb、watch page telemetry、fail-closedが既に重いため、主経路を自前ingestへ寄せる方が障害点を自分で観測しやすい可能性がある。

## Alternatives Considered

| Option | Pros | Cons | Current Judgment |
| --- | --- | --- | --- |
| 現行維持: YouTube一次配信 → 斉藤さんbridge | 既存PoC資産を継続利用できる | watch page依存、遅延、stall、botリスクが残る | 過渡期のfallbackとして残す |
| スマホ → OBS/FFmpeg RTMP → YouTube、別経路で斉藤さん | 実装が単純 | 二重送信・時刻同期・運用が割れやすい | 不採用。fan-outはサーバー側で行う |
| スマホ → mediaMTX → YouTubeのみ。斉藤さんは手動 | YouTube経路だけなら速い | 今回の同時配信目的を満たさない | 段階PoCの中間ゲートとしては可 |
| 自前SFUをゼロから実装 | 制御自由度が高い | コスト過大 | 不採用 |
| SRT/RTMPをスマホから直接送る | mediaMTX設定が簡単 | モバイル回線でのNAT/firewall耐性がWebRTCより弱いことが多い | 第二候補。まずはWebRTC/WHIP |

## Relation to Existing ADRs

| ADR | Relation |
| --- | --- |
| ADR-0002 | 目的は維持。入力元をYouTubeから自前ingestへ変更する上書き候補 |
| ADR-0004 | 引き続きRejected。斉藤さん画面をYouTubeへミラーする逆方向には戻らない |
| ADR-0009 | 本ADRで再評価を開始する。Accepted時にADR-0009をSupersededにする |
| ADR-0019 / ADR-0020 | YouTube watch page経路の安定化知見は、fallbackまたは比較基準として残す |
| ADR-0008 | Android Runtime Hostは斉藤さん出力側で引き続き必要 |
| ADR-0010 / ADR-0015 | 開始設定契約に「ingest URL / WHIP endpoint / output targets」を追加する必要がある |

## Open Questions
- スマホpublisherは専用アプリか、ブラウザWHIPか、既存配信アプリか。
- mediaMTXの配置はAndroid Runtime Hostと同一EC2か、分離か。
- TURNサーバーは自前か、Managedサービスか。モバイル回線実測で必要か。
- YouTubeと斉藤さんで解像度/フレームレート/ビットレートを分岐するか、共通パイプラインにするか。
- 音声はmediaMTX段階でAAC化し、Android側へどう渡すか。既存のEmulator音声注入未解決問題は残る。
- ingest切断時にYouTubeだけ継続するか、両方fail-closedするか。
- WHIP認証、path公開範囲、replay攻撃対策をどうするか。

## Risk Notes
- 規約リスク: 斉藤さん側の自動操作/仮想カメラ経路はADR-0002と同様に残る。
- 技術リスク: モバイルWebRTCのNAT越え、再接続、電波切れ復帰。
- 運用リスク: mediaMTXまたはEC2障害が両出力へ同時影響する単一障害点になる。
- 品質リスク: fan-out時の再エンコードによる劣化、音ズレ。
- セキュリティリスク: WHIP endpointとYouTube stream keyの漏洩。
- スコープリスク: publisher実装まで広げるとPoCが再び拡散する。

## Compliance Gate
- 配信者本人の映像・音声のみを扱う。
- YouTubeへ出す範囲（非公開/限定公開/公開）を明示する。
- 斉藤さんLIVE側の自動操作・仮想カメラ利用が規約上許容されるか、現行Compliance Gateを再確認する。
- stream key、Cookie、アカウント情報をartifactやログへ出さない。

## Validation Plan

### Phase A: mediaMTX ingest spike

1. EC2にmediaMTXを単体起動する。
2. ブラウザまたはテストクライアントからWHIPで映像・音声をpublishする。
3. mediaMTXのローカル再生（WebRTC/RTSP/HLS）で受信を確認する。

Success criteria:

- 5分以上、映像と音声が途切れない。
- 切断/再接続時の挙動を記録できる。

### Phase B: YouTube fan-out

1. mediaMTXからYouTube LiveへRTMP pushする。
2. 非公開または限定公開で10分配信する。

Success criteria:

- YouTube側で映像・音声が視聴できる。
- アーカイブが残る。

### Phase C: 斉藤さんbridge接続

1. mediaMTXのローカル出力を既存FFmpeg → v4l2 → Emulator経路へ接続する。
2. 斉藤さんcamera枠に映ることをフレンド限定で確認する。

Success criteria:

- YouTube起点ではなくmediaMTX起点で、斉藤さんLIVEに映像が出る。
- 短時間でもよいので、両出力が同時に生きていることを確認する。

### Phase D: 部分障害と運用

1. ingest切断、YouTube push失敗、Android bridge失敗を個別に起こす。
2. fail-closedまたは片系継続の方針を決める。

Success criteria:

- 障害時にどちらが落ちたか分かるログが残る。
- 空配信や黒画面を長時間流し続けない。

## Initial Implementation Boundary

このリポジトリ（`saitousan-docs`）はドキュメント検討専用とする。実装コード、バイナリ、実行artifactは置かない。

このリポジトリで残すもの:

- `architecture/aws-webrtc-mediamtx-fanout.md` の構成メモ
- `research/phase-a-mediamtx-ingest-plan.md` の Phase A 検証手順
- 実測結果の要約を `research/validation-log.md` へ追記

実装・実測の置き場:

- mediaMTX 起動、WHIP publisher、計測ログは別リポジトリまたは EC2 作業ディレクトリで行う
- 斉藤さん bridge は既存 PoC 資産を入力源差し替えで再利用する
- YouTube watch page capture 経路は、移行完了まで fallback として残す

## Decision Triggers

次を満たしたら、このADRを `Accepted` にし、ADR-0009を `Superseded` にする。

- Phase AとPhase Bが成立する。
- Phase CでmediaMTX起点の斉藤さん映像注入が確認できる。
- 現行YouTube起点より運用上の利点（遅延、安定性、切り分け）が実測で説明できる。

次に該当したら、本ADRを取り下げまたは縮小する。

- モバイルWebRTC ingestが安定しない。
- mediaMTX運用コストや障害面がYouTube起点より重い。
- 斉藤さんbridgeへの接続で新たなブロッカーが出て、現行経路の方が明らかに良い。

## Notes
関連:

- ADR-0002
- ADR-0009
- ADR-0019
- ADR-0020
- `architecture/aws-youtube-to-saitousan-live.md`
- `architecture/aws-webrtc-mediamtx-fanout.md`
- `research/phase-a-mediamtx-ingest-plan.md`
- Issue #6（一次配信元の再評価）
