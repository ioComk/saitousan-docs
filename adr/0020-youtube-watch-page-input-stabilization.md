# ADR-0020: YouTube watch page入力を画質固定とtelemetryで安定化する

## Status

Accepted

## Context

ADR-0019では、初期PoCのミラー元をYouTube StudioのLive Control Roomではなく、通常のYouTube watch pageにする方針を検討している。

2026-07-06のフレンド限定テスト配信では、YouTube watch pageからChromium、Xvfb、FFmpeg、`/dev/video0`、Android Emulator、斉藤さんLIVEまでのcamera injectionは成立した。一方で、約6〜7分後にwatch page内の`<video>`が45秒進まず、fail-closedでChromium、FFmpeg、Emulatorを停止した。

artifact上の状態は次の通りだった。

- `paused: false`
- `readyState: 2`
- `mediaError: null`
- `detectedPageError: null`
- `videoWidth: 1280`
- `videoHeight: 720`
- `stalledForSeconds: 45.274`

FFmpegはXvfb画面を30fpsで`/dev/video0`へ書き続けていたため、主因はV4L2やAndroid camera登録ではなく、EC2上のChromiumで再生しているYouTube watch page側のbuffer不足と判断する。

## Decision

YouTube watch page経路は維持し、初期安定化として次を実装する。

- YouTube playerの画質を既定で`large`（480p相当）へ固定する。
- `YOUTUBE_PLAYER_QUALITY`で`medium`、`large`、`hd720`などへ変更できるようにする。
- `YOUTUBE_PLAYER_QUALITY_LOCK=0`でYouTubeのauto qualityへ戻せるようにする。
- 1秒ごとにplayback telemetryをJSON Linesで保存する。
- telemetryには`readyState`、`networkState`、`currentTime`、`bufferedAhead`、buffer range、YouTube quality、available qualities、player state、total/dropped framesを含める。
- URL、video ID、Cookie、account情報、限定公開URLはtelemetryへ保存しない。
- playback error、bot/sign-in、広告overlay、stall検出時はこれまで通りfail-closedする。

この段階では自動reloadや無制限retryは実装しない。配信中にreloadすると斉藤さん側へ黒画面、停止画、UI overlayが流れる可能性があるため、復旧よりも安全停止を優先する。

## Rationale

直近の失敗は、再生開始時には480pだったYouTube映像が停止時には720pになっていた。EC2上ではChromium、Xvfb、FFmpeg、Android Emulator、斉藤さんアプリを同時に動かしており、auto qualityで720pへ上がるとCPU、decode、network bufferに余裕がなくなる可能性がある。

画質固定は画質を犠牲にするが、次の利点がある。

- YouTube側のauto quality変動を減らせる。
- EC2上のdecode負荷とnetwork必要量を下げられる。
- stall再現時に原因を比較しやすい。
- watch page、viewer account、権限分離の境界を維持できる。

telemetryは、次の切り分けに必要である。

- `bufferedAhead`が減り続けるならnetwork/CDN/source側のbuffer不足寄り。
- `droppedVideoFrames`が増えるならdecode/render負荷寄り。
- `youtubeQuality`が上がるならquality lock不発またはYouTube側挙動変更。
- `networkState`や`readyState`が変化するならHTML media stateの問題。

## Alternatives Considered

### YouTube watch pageを自動reloadする

利点:

- 一時的なplayer stallから戻れる可能性がある。

不採用理由:

- 配信中にreload画面、黒画面、YouTube UI overlayが斉藤さん側へ流れる。
- reload後の広告、consent、bot/sign-in画面の混入リスクがある。
- 原因の切り分け前に復旧処理を重ねると検証が濁る。

### Live Control Room previewへfallbackする

利点:

- creator sessionではwatch pageより安定する可能性がある。

不採用理由:

- ADR-0019の権限分離と画面混入リスクに反する。
- Studio controls、analytics、stream key周辺をcapture対象へ近づける。
- 誤操作で配信開始・終了・設定変更につながる。

### 配信元で映像を分岐する

利点:

- YouTube viewer latency、browser UI、CDN buffer、再圧縮を避けられる。

現時点での扱い:

- watch page経路の10分以上安定が満たせない場合の第一再検討候補にする。
- 採用時はADR-0009とADR-0019の関係を整理し、新しいADRで決定する。

## Validation Plan

### Gate A: browser-only安定性

- `scripts/07-youtube-browser-record.sh`で20分録画する。
- `YOUTUBE_PLAYER_QUALITY=large`で実行する。
- `playback-telemetry.jsonl`を確認する。

Success criteria:

- stallによるfail-closedが起きない。
- `bufferedAhead`が長時間0付近に張り付かない。
- `youtubeQuality`が意図した値から大きく外れない。
- dropped frameが継続的に増え続けない。

### Gate B: browser-to-v4l2安定性

- `scripts/03-youtube-browser-to-v4l2.sh`で20分実行する。
- `/dev/video0`への書き込みが継続する。
- telemetryでbuffer不足がないことを確認する。

### Gate C: full pipeline安定性

- `scripts/08-youtube-to-emulator.sh`でEmulatorまで起動する。
- 斉藤さんcamera previewでYouTube映像を確認する。
- 外部配信開始は別途人間の明示承認を必要とする。

Success criteria:

- 10分以上、watch page stallなし。
- guest Android camera deviceが1件以上。
- camera previewがYouTube映像を表示し続ける。
- fail-closed時はprocessと`/dev/video0`を安全に片付けられる。

## Consequences

良くなること:

- 720p auto quality上昇によるbuffer不足を避けやすくなる。
- 次の失敗時にnetwork、decode、YouTube player stateのどこが悪いか見える。
- 機密情報を保存せず、検証に必要な最小telemetryを残せる。

悪くなること:

- 480p相当へ固定するため、斉藤さん側へ届く映像品質は下がる。
- YouTube playerの内部methodに依存するため、YouTube UI変更でquality lockが効かなくなる可能性がある。
- telemetry artifactが増える。

## Notes

確認日: 2026-07-07 UTC

関連:

- [ADR-0002: YouTube配信を斉藤さんLIVEへ中継するラッパー構成](0002-youtube-to-saitousan-live-wrapper.md)
- [ADR-0019: YouTubeのミラー元はLive Control Roomではなくwatch pageにする](0019-youtube-watch-page-over-live-control-room-capture.md)
- `../research/validation-log.md`
- `/home/rwatanabe/android-live-camera-poc/RESULTS.md`
