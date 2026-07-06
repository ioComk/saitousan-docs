# ADR-0019: YouTubeのミラー元はLive Control Roomではなくwatch pageにする

## Status

Proposed

## Context

ADR-0002では、YouTube Liveを一次配信元とし、その映像をAndroid側のカメラ入力へ渡して斉藤さんLIVEへミラーリングする方針を採用している。

EC2上では、Xvfbに表示したChromiumの画面をFFmpegでMP4または仮想カメラへ渡す経路を検証している。ここで、ブラウザに表示するYouTube画面として次の2案がある。

1. YouTube StudioのLive Control Roomにあるpreview
2. 視聴者が開く通常のYouTube watch page

YouTube公式手順では、encoderから映像を送った後、Live Control Roomでpreviewの表示を待って`Go live`を実行する。Live Control Roomではstream health、視聴者数、chat rateなども確認できる。したがって、Live Control Roomは配信開始と監視のためのcontrol surfaceである。

一方、watch pageは視聴者へ配信されるYouTube側の出力である。今回の目的は、管理画面のpreviewを複製することではなく、YouTube視聴者へ届く映像を斉藤さんLIVEへミラーリングすることである。

既存検証では、Chromium、Xvfb、FFmpegによる画面取得経路は成立したが、EC2から公開YouTube URLを開くとbot/sign-in保護に当たる可能性が確認されている。どちらの画面を選んでも、認証、セッション維持、UI変更、映像遅延を評価する必要がある。

## Decision

初期PoCのミラー元には、YouTube StudioのLive Control Roomではなく、通常のYouTube watch pageを使う。

運用境界は次の通りとする。

- 配信開始、終了、stream health確認はLive Control Roomで人間が行う。
- ミラーworkerは`https://www.youtube.com/watch?v=...`形式のwatch pageだけを開く。
- YouTube側で配信開始後、watch page内の`<video>`について、映像サイズ、再生状態、再生時間の進行を確認してからcaptureを開始する。
- watch pageはfullscreen表示にし、chat、Studio controls、analytics、stream keyなどをcapture対象に含めない。
- 初期検証では、配信者本人のテスト映像を限定公開で配信する。
- 限定公開URLは認証を伴うアクセス制御ではない。URLを知る人は閲覧・再共有できるため、ログ、Pull Request、公開artifactへ残さない。
- 匿名再生を第一候補にする。sign-inが必要な場合は、creator権限を持たない専用viewer accountを使う。
- browser profileとCookieはrepo外に置き、Git、AMI、ログ、artifactへ保存しない。loginと追加認証は人間の操作境界にする。
- creator accountのStudio sessionを、無人のcapture workerへ渡さない。
- watch pageがbot/sign-in保護で再生できない場合、Live Control Room previewへ自動fallbackしない。失敗として記録し、専用viewer accountまたは配信元での映像分岐を再評価する。
- ミラーworkerから`Go live`、`End stream`、配信設定変更を実行しない。

YouTubeのviewer latencyは、encoder入力からwatch page表示までの遅延を含む。公式資料では、low latencyは多くの視聴者で10秒未満、ultra-low latencyは5秒未満とされる一方、latencyを下げるほどbufferingが増える可能性がある。初期PoCではlow latencyとultra-low latencyを実測比較し、安定性を満たす方を選ぶ。Studio previewの見た目だけで遅延を判断しない。

## Rationale

| 観点 | Live Control Room preview | 通常のwatch page |
| --- | --- | --- |
| 役割 | 配信開始、設定、監視 | 視聴者への映像配信 |
| 出力の意味 | operator向けpreview | 視聴者が見る最終出力 |
| 必要権限 | creator accountが必要 | 限定公開ならURL共有可能。匿名またはviewer accountでよい |
| 画面混入リスク | controls、analytics、chat、stream key周辺 | fullscreen化すれば映像中心にできる |
| 誤操作リスク | `Go live`、`End stream`、設定変更が同じ画面にある | 配信管理操作を持たない |
| 結合度 | Studio UIとcreator sessionに強く依存 | watch page playerに限定できる |
| 視聴者との一致 | viewer出力と同一とは限らない | viewer delivery pathそのもの |
| 主な弱点 | 機密画面、UI変更、権限集中 | bot/sign-in保護、viewer latency、再生overlay |

watch pageを選ぶ最大の理由は、映像品質だけではなく権限境界である。配信を管理できるcreator sessionと、映像を読むだけのcapture workerを分離すると、画面誤操作、Cookie漏えい、stream key露出の影響を小さくできる。

## Consequences

良くなること:

- YouTube視聴者へ実際に届く映像をミラー対象にできる。
- Live Control Roomのcontrols、analytics、stream key周辺を録画しなくて済む。
- capture workerへcreator権限を持たせずに運用できる。
- Studio UI変更とcapture処理の結合を減らせる。
- YouTube側の再生不能を視聴者影響に近い形で検出できる。

難しくなること:

- YouTube ingest、transcode、viewer bufferを通るため遅延が追加される。
- YouTubeによる再圧縮後の映像をさらにcapture・encodeするため、画質劣化とCPU負荷が増える。
- EC2のanonymous browserがbot/sign-in保護に当たる可能性がある。
- 配信開始前は実映像を取得できないため、YouTube配信開始と斉藤さん側ミラー開始に時間差が出る。
- 限定公開URLが漏れると、第三者に閲覧・再共有される可能性がある。
- playback error、consent、sign-in、広告などのoverlayを検知して停止する必要がある。

## Alternatives Considered

### YouTube StudioのLive Control Room previewをcaptureする

利点:

- 配信開始前にpreviewを確認できる。
- creator sessionではanonymous browserよりbot/sign-in保護を回避しやすい可能性がある。
- stream healthと同じ画面で状態を確認できる。

不採用理由:

- 配信管理画面であり、視聴者向け最終出力ではない。
- creator Cookieと権限を無人workerへ渡す必要がある。
- controls、analytics、chat、配信情報の混入・漏えいリスクがある。
- capture automationの誤clickが配信開始・終了・設定変更につながる。
- Studio UI変更がcapture処理へ直接影響する。

Live Control Roomはoperatorの診断・監視画面として残すが、映像のsourceにはしない。

### 配信元でYouTube用と斉藤さん用へ映像を分岐する

利点:

- YouTube viewer latency、bot検知、browser UI、再圧縮を避けられる。
- 同一の元映像を両方へ渡せる。

欠点:

- 現在の「YouTube出力を取得する」PoCから責務が変わる。
- 配信元、分岐、音声同期、障害復旧を別途設計する必要がある。

watch page経路が安定しない、または遅延が許容できない場合の第一再検討候補とする。採用時はADR-0009との関係を整理し、新しいADRで決定する。

### 非公式なstream URL抽出またはdownload toolを使う

既存検証でbot/sign-in保護に当たり、YouTube側の変更にも強く依存する。今回のbrowser capture PoCでは採用しない。

## Validation Plan

### Gate A: watch page再生

1. 配信者本人のテスト映像を限定公開で配信する。
2. Live Control Roomで人間が配信開始する。
3. EC2のChromiumでwatch pageを開く。
4. `<video>`の`readyState`、`videoWidth`、`videoHeight`、`paused`、`currentTime`を記録する。
5. 10分間、fullscreen画面をMP4へcaptureする。

Success criteria:

- 再生時間が進んだ後だけcaptureを開始する。
- 10分間のMP4が生成される。
- 長時間の黒画面、静止、sign-in、playback errorを検出しない。
- creator accountのCookieを使わない。

### Gate B: latencyと安定性

1. 元映像に時刻または連番を表示する。
2. low latencyとultra-low latencyで各10分captureする。
3. 元映像からwatch pageまでの遅延、buffering、frame drop、再接続回数を測る。
4. 斉藤さん側の追加遅延と合わせて許容可否を判断する。

Success criteria:

- 両設定の遅延と停止率を比較できる。
- 採用するlatency設定と理由を検証ログへ記録できる。

### Gate C: 認証境界

1. anonymous browserで再生する。
2. 失敗時だけ専用viewer accountの永続profileを人間が作る。
3. EC2再起動後にprofileをrepo外から読み、再生可否を確認する。
4. profile、Cookie、限定公開URLがGit、ログ、artifactに含まれないことを確認する。

## Failure Policy

- playback readinessを確認できない場合は録画を開始しない。
- sign-in、bot確認、playback error、長時間の黒画面・静止を検出したらミラーを停止する。
- Studioへの自動遷移やcreator accountでの自動loginを行わない。
- 配信中の自動復旧は無制限に繰り返さず、回数上限後に人間へ通知する。

## Notes

確認日: 2026-07-03 UTC

関連ADR・実測:

- [ADR-0002: YouTube配信を斉藤さんLIVEへ中継するラッパー構成](0002-youtube-to-saitousan-live-wrapper.md)
- [ADR-0009: ADR-0002の一次配信元は当面YouTubeを維持する](0009-adr-0002-owned-source-rejected-for-initial-phases.md)
- `../research/validation-log.md`
- `/home/rwatanabe/android-live-camera-poc/RESULTS.md`

YouTube公式資料:

- Create a YouTube live stream with an encoder: https://support.google.com/youtube/answer/2907883
- See your live stream's metrics: https://support.google.com/youtube/answer/2853833
- Understand live streaming latency: https://support.google.com/youtube/answer/7444635
- Manage live stream settings: https://support.google.com/youtube/answer/9854503
- Change video privacy settings: https://support.google.com/youtube/answer/157177

公式資料から確認できるのは、Live Control Roomがpreview、配信開始、stream health、analyticsを扱うこと、watch pageがviewer向け配信面であること、限定公開URLの共有条件、latency設定の目安である。watch pageを主経路にする判断と権限分離の評価は、これらと既存PoC結果に基づく本プロジェクトの推論である。
