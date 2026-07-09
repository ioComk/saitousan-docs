# Validation Log

ADR-0002のPoC結果を記録する。

## Phase 0: Android実行環境の成立確認

Status: Partial

Date: 2026-07-03 UTC

Environment:

- OS: Ubuntu 26.04 LTS / EC2
- Android Studio: 未使用
- Android SDK: API 36 / Emulator 36.6.11 / Platform Tools 37.0.0
- Emulator image: `system-images;android-36;google_apis_playstore;x86_64`
- AVD: `saitosan_play_api36`
- Acceleration: KVM 12
- Appium: 今回は未実行
- Saitousan app version: 3.7.19 (`versionCode=2665`)

Steps:

1. 現ユーザー`rwatanabe`を`kvm` groupへ追加する。
2. Google Play付きAPI 36 AVDをKVM有効でheadless起動する。
3. ADB接続、`sys.boot_completed=1`、Launcher表示を確認する。
4. 前回PoCの20 split APKから斉藤さん3.7.19をインストールする。
5. アプリを起動し、runtime permissionを付与する。
6. 強制update画面とPlay Store更新導線を確認する。
7. screenshot、package情報、crash logを保存する。
8. 検証後にEmulatorを停止する。

Result:

- Emulator boot: 成功。Android 16 / API 36。boot完了まで42.8秒。
- App launch: 成功。`yudo.work.saitosan/.activity.LaunchActivity`がforegroundになり、processも継続した。
- Crash: `FATAL EXCEPTION`なし。
- Login / setup: 未達。3.7.19は強制update画面を表示する。
- Play Store: 更新導線は開くが、unauthenticated状態でSign inが必要。
- Live screen reachability: 未達。
- Appium accessibility: 今回は未確認。
- Screenshots:
  - `/home/rwatanabe/android-live-camera-poc/artifacts/emulator-boot/settled.png`
  - `/home/rwatanabe/android-live-camera-poc/artifacts/emulator-boot/saitosan-initial.png`
  - `/home/rwatanabe/android-live-camera-poc/artifacts/emulator-boot/saitosan-after-permissions.png`
  - `/home/rwatanabe/android-live-camera-poc/artifacts/emulator-boot/play-store-update.png`

Findings:

- `/dev/kvm`は存在したが、kvm groupが旧ユーザー`tsubasaoura`だけを含んでいた。`rwatanabe`追加後、`emulator -accel-check`は`KVM (version 12) is installed and usable.`になった。
- Google Play付きAVD、ADB、Launcher、split APK install、斉藤さんprocess/UI起動まではEC2で成立する。
- 現在のblockerはEmulator互換性ではなく、アプリ3.7.19の強制updateとPlay Store認証である。
- creator/password/OTPを自動化せず、Google loginと追加認証は人間の操作境界にする。

Next Actions:

1. Play Storeへ人間がSign inして更新するか、検証済み最新APK/XAPKを用意する。
2. 最新版のpackage version、起動、login前画面を確認する。
3. 配信開始直前のcamera previewまで進む。
4. Appiumで画面要素とscreenshotを取得する。
5. YouTube watch page captureを仮想cameraへ流し、斉藤さんcamera previewで確認する（後続検証でdeviceを`/dev/video0`へ変更）。

## Phase 2準備: YouTubeブラウザcapture

Date: 2026-07-03 UTC

Result:

- Xvfb上のChromium画面をFFmpegでH.264 MP4へ保存する経路を実装した。
- 生成テスト映像で10秒、1280x720、30fps、H.264、yuv420pのcaptureに成功した。
- 再生時間の進行をcapture開始条件にし、黒画面、静止、duration、codecを自動検査する。
- 実YouTube Live URLは未検証。bot/sign-in保護の可能性が残る。
- ADR-0019を`Proposed`として作成し、ミラー元はLive Control Room previewではなく通常のwatch pageを第一候補とした。

Next Actions:

1. 配信者本人の限定公開YouTube Live URLでwatch page captureを実測する。
2. anonymous再生を試し、必要時だけ専用viewer accountのrepo外profileを使う。
3. low latencyとultra-low latencyの遅延・停止率を比較する。

## 2026-07-03 Session Handoff

- App repo: `/home/rwatanabe/android-live-camera-poc`
  - Branch: `feat/youtube-browser-recording`
  - browser録画実装、validation、README、RESULTS、lock fileに未commit差分あり。
  - 生成動画とEmulator証跡はignoredの`artifacts/`配下。
- Docs repo: `/home/rwatanabe/saitousan-docs`
  - Branch: `docs/adr-youtube-capture-surface`
  - ADR-0019、ADR index、このvalidation logに未commit差分あり。
- どちらもcommit、push、Pull Request作成は未実施。
- Emulatorは検証終了後に停止済み。

## 2026-07-06: Browser captureからAndroid Emulatorまでの統合

Status: Partial

Environment:

- Android AVD: `saitosan_play_api36`
- Saitousan app: 3.8.12 (`versionCode=2686`)
- Virtual camera: `/dev/video10`
- Browser display: Chromium + Xvfb

Result:

- 人間のGoogle認証後、Play Storeから斉藤さん3.8.12へ更新した。
- テストアカウントでログインし、ハンカチ中継設定画面へ到達した。
- 権利処理済みFFmpeg生成動画をChromiumで再生し、Xvfbから`/dev/video10`へ1280x720 / 30fpsでcaptureした。
- 5秒のV4L2 readbackはH.264 / yuv420p / 1280x720 / 30fpsで、黒画面と静止を検出しなかった。
- `scripts/08-youtube-to-emulator.sh`で`webcam0`をfront/back cameraへ割り当て、Google Play API 36 AVDのboot完了を確認した。
- state artifactへYouTube queryとvideo IDを保存しないredactionを追加した。
- 再生停止をhealth checkで検出し、browser/V4L2 workerと関連processを停止できることを確認した。
- player error、bot/sign-in画面、広告overlayを停止条件にした。
- 外部配信は開始していない。

Public YouTube watch page:

- 再配信許諾を確認した公開Liveを匿名browserとrepo外の永続profileで試した。
- YouTube resourceがHTTP 403となり、pageが閉じてplayback readyへ到達しなかった。
- fail-closedによりV4L2 worker、Emulator、斉藤さん配信は開始していない。
- Playwright起動browserでGoogle sign-inが拒否されたため、login専用経路は通常Chromiumを直接起動し、X11画面だけをlocalhost限定remoteへ中継する構成へ変更した。
- 専用viewer accountへの人間によるGoogle loginが成功した。
- remote終了後、port 4174とlogin用processが停止し、権限`0700`のrepo外profileにCookie storeが残ることを確認した。
- 同じprofileで公開Liveのwatch pageを再検証し、再生readyへ到達した。
- 30秒の画面録画はH.264 / yuv420p / 1280x720 / 30fps、黒画面0秒、静止0秒、player errorなしだった。
- state artifactにはYouTube queryとvideo IDを保存していない。
- この検証ではEmulatorと斉藤さん配信を起動していない。

Remaining:

- 実YouTube Liveの仮想camera投入とAndroid camera登録は後続検証で達成した。
- 斉藤さん内のcamera preview有無と`webcam0`映像認識を確認する。
- 音声入力は未接続。
- ADR-0019は`Proposed`のまま。

## 2026-07-06: フレンド限定テスト配信

Status: Failed camera injection, stream lifecycle verified

Result:

- 再配信許諾を確認した公開YouTube Liveを入力にした。
- テストアカウントで公開範囲をフレンド限定、通知3種をすべてOFFにして通常中継を開始した。
- 通常中継は30分固定だったため、テスト終了時に人間の承認範囲内で手動停止した。
- 配信開始、カメラON、配信終了のUI操作は成立した。
- カメラON後もYouTube映像はアプリ画面へ表示されなかった。
- `dumpsys media.camera`はcamera device 0件、active client 0件だった。
- 空映像を継続せず停止し、「中継を終了しました」を確認した。
- 終了後、Emulator、Chromium、Xvfb、FFmpegの残留processはなかった。

Safety change:

- `scripts/08-youtube-to-emulator.sh`はboot後にAndroid camera device数を確認する。
- 0件なら斉藤さんを起動せず、関連processを終了してfail-closedする。

Next:

1. `webcam0`未登録の原因と修正結果は次節に記録する。
2. camera previewでYouTube映像を確認した後に、短時間のフレンド限定配信を再試験する。

## 2026-07-06: webcam0未登録の解消

Status: Resolved

Root Cause:

- `v4l2loopback`を`/dev/video10`へ作成していた。
- Android EmulatorのLinux camera列挙実装は`/dev/video0`からcamera上限数までを走査する。
- `/dev/video10`は`emulator -webcam-list`へ現れず、guest Androidのcamera deviceも0件になった。

Change:

- 仮想cameraの既定deviceを`/dev/video0`へ変更した。
- `00-host-check.sh`は現在のユーザーだけにdeviceのread/write権限を付与する。
- `08-youtube-to-emulator.sh`はEmulator起動前に`webcam0`列挙、boot後にAndroid camera device数を検査する。
- どちらかが失敗した場合は斉藤さんを起動せず、関連processを終了する。

Validation:

- testsrc producer稼働中、`emulator -webcam-list`が`VirtualCam`を`webcam0`として列挙した。
- `-no-window -camera-back webcam0 -camera-front webcam0`でheadless bootした。
- `dumpsys media.camera`はcamera device 1件、API1公開device 1件を返した。
- login済みwatch pageから`/dev/video0`へYUYV / 1280x720 / 30fpsで投入した統合pipelineでも同じ結果だった。
- pipeline stateは`virtualCameraReady: true`、`cameraDeviceCount: 1`、`streamStarted: false`だった。
- 外部配信は開始していない。

## 2026-07-06: webcam0修正後のフレンド限定配信

Status: Camera injection succeeded, 10-minute stability not reached

Scope:

- テストアカウント
- フレンド限定
- 通知OFF
- 通常中継
- 音声なし
- ユーザーの明示承認後に開始

Result:

- 開始前にhost `webcam0`とguest Android camera device 1件を確認した。
- カメラON後、斉藤さん配信画面へYouTube由来の象映像が表示された。
- ユーザー側でも配信映像を確認できた。
- watch pageは1280x720、再生中、stall 0秒で動作した。
- 約6〜7分後、watch pageの`readyState`が2のまま進まず、45秒stall判定に到達した。
- fail-closedによりChromium、FFmpeg、Emulatorが終了し、配信接続が切断された。
- 10分連続配信は未達だが、YouTube映像から斉藤さん配信画面までのcamera injectionは成立した。

Shutdown:

- ユーザーの作業終了指示後、残留processがないことを確認した。
- ADB serverを停止した。
- `v4l2loopback`をunloadし、`/dev/video0`が消えたことを確認した。

Next:

1. watch pageのbuffering原因と画質・network条件を測る。
2. 配信中のsource stall時に安全停止したことを外部視聴側でも確認する手順を追加する。
3. 10分以上のフレンド限定配信を再試験する。

## 2026-07-08: 斉藤さんLIVE黒画面のfront camera仮説（後続検証で不採用）

Status: Superseded by BACK camera + cover validation later on 2026-07-08

Note:

- この節は途中仮説の記録であり、最終方針ではない。
- 後続検証で、`saitosan-square` + front-onlyはSaitousanがcameraをopenせず、YouTube映像表示には至らないことを確認した。
- 最終成功構成は後続の「Saitousan LIVE camera枠へのYouTube映像表示成功」を参照する。

Scope:

- YouTube watch page: `https://www.youtube.com/watch?v=xS85Jp8UE2w`
- Stream profile: `saitosan-square`
- Virtual camera: `/dev/video0`, `1080x1080`, YUYV, 30fps
- Emulator: `saitosan_play_api36`, Android Emulator 36.6.11
- Saitousan account: `szk`
- Public scope: フレンドのみ
- Audio: disabled

Root Cause:

- `-camera-back webcam0 -camera-front webcam0`で同じhost webcamをfront/back両方へ渡すと、guest Androidではcamera device 1件が`BACK`として登録された。
- 斉藤さんLIVEのカメラON時、host `/dev/video0` readbackは非黒でも、配信画面側は黒になった。

Change:

- `scripts/08-youtube-to-emulator.sh`の既定を斉藤さん向けに`-camera-back none -camera-front webcam0`へ変更した。
- `EMULATOR_CAMERA_BACK`、`EMULATOR_CAMERA_FRONT`、`EMULATOR_EXPECTED_CAMERA_FACING`を追加した。
- 起動後の`dumpsys media.camera`でcamera facingを読み取り、既定では`FRONT`でなければfail-closedする。
- pipeline stateへcamera back/front/facingを保存する。

Validation:

- front-only起動ではguest Androidのcamera device 1件が`FRONT`として登録された。
- pipeline state:
  - `videoSize: 1080x1080`
  - `browserSize: 720x1280`
  - `cameraFitMode: blur`
  - `emulatorCamera.back: none`
  - `emulatorCamera.front: webcam0`
  - `emulatorCamera.facing: FRONT`
  - `cameraDeviceCount: 1`
  - `cameraStreamHasExpectedSize: true`
- Saitousan LIVEをフレンド限定で開始し、カメラON時に`Layout_HostCamera`配下へ`SurfaceView`が出た。
- `dumpsys media.camera`は`Facing: Front`、`Camera error traces (0)`だった。
- ADB screenrecordのLIVE camera cropは`YAVG≈211`で黒画面ではなかった。
- カメラtoggleをOFFにすると`Image_HostAvatar` overlayへ戻ることを確認し、ON/OFF状態のUI差分も取れた。
- テスト配信は終了し、結果画面から退出してハンカチ中継一覧へ戻った。

Artifacts:

- `/home/rwatanabe/android-live-camera-poc/artifacts/youtube-emulator/20260708T191933Z/pipeline-state.json`
- `/home/rwatanabe/android-live-camera-poc/artifacts/youtube-emulator/20260708T191933Z/screen-live-camera-on-front.png`
- `/home/rwatanabe/android-live-camera-poc/artifacts/youtube-emulator/20260708T191933Z/live-camera-front-crop.png`
- `/home/rwatanabe/android-live-camera-poc/artifacts/youtube-emulator/20260708T191933Z/v4l2-live-front-frame.png`

Limit:

- ADB screenshot/screenrecordは`SurfaceView`の実カメラ内容を完全には反映しない場合がある。今回の合格条件は「配信画面が黒くならない」ことに置いた。YouTube映像そのものが外部視聴側へ届くかは、次回フレンド端末で追加確認する。

## 2026-07-08: 縦型YouTube入力の黒画面と音声なし

Status: Video mitigation implemented, audio unresolved

Scope:

- テストアカウント
- フレンド限定
- 通常中継
- login済みwatch page profile
- 縦型YouTube watch page入力

Observed:

- `STREAM_PROFILE=vertical`でChromium/Xvfbとv4l2出力をどちらも`720x1280`にした。
- host側`/dev/video0` readbackではYouTube映像が正常に見えた。
- guest Androidの`dumpsys media.camera`はcamera device 1件、stream configuration `720x1280`を返した。
- 斉藤さんLIVEの外部視聴では映像が黒、音声も出なかった。
- 空配信を継続せず、配信を停止した。終了後は通常のハンカチ中継画面へ戻った。

Video finding:

- host YouTube captureとv4l2loopbackは正常だったため、黒画面の失敗点はEmulator/Saitousan側のcamera ingestと判断した。
- 旧Camera APIやアプリ側の期待解像度がportrait-only `720x1280`と合わない可能性が高い。

Video change:

- `STREAM_PROFILE=vertical-in-720p`を追加した。
- Chromium/Xvfbは`BROWSER_WIDTH=720`、`BROWSER_HEIGHT=1280`で縦型watch pageを維持する。
- v4l2/webcamへ渡すcamera入力は`VIDEO_WIDTH=1280`、`VIDEO_HEIGHT=720`へscale/padする。
- `CAMERA_FIT_MODE=contain`は縦映像を中央配置し、左右を黒帯にする。`cover`はcropする。

Video validation:

- `scripts/07-youtube-browser-record.sh`で8秒録画を検証した。
- H.264 / yuv420p / 720x1280 / 30fps、黒画面0秒、静止0秒、validation errorなし。
- `scripts/03-youtube-browser-to-v4l2.sh`で`/dev/video0`へ`1280x720` / YUYV / 30fps出力した。
- readback frameは黒画面ではなく、縦コンテンツが中央配置された。
- `scripts/08-youtube-to-emulator.sh`で配信開始なしのpipelineを起動し、guest Android camera deviceは`1280x720` stream configurationを返した。

Audio finding:

- PulseAudio route自体は作成でき、Chromium sink inputは`yt_sink`へ接続された。
- Android Emulator 36.6.11は`pulseaudio` backend名を受け付けず、`pa` backendでも`Could not init pa audio driver`で初期化に失敗した。
- `AUDIO_TRANSPORT=grpc`の実験実装でEmulator gRPC `injectAudio`へ接続し、audio packet送信開始までは到達した。
- ただしこの環境では`injectAudio`開始後にEmulatorがsegmentation faultしたため、gRPC音声注入は採用しない。

Safety:

- 黒画面の外部配信は停止済み。
- 修正後の外部配信再開は未実施。再開前に人間の明示承認を必要とする。

Next:

1. `vertical-in-720p`でフレンド限定の短時間配信を再試験し、外部視聴側で映像が黒でないことを確認する。
2. 音声はPulseAudio backendが動作するEmulator/host構成、またはgRPC `injectAudio`のcrashしない条件を別途調査する。
3. 音声が未解決の間は、映像検証と音声検証を分離する。

## 2026-07-07: YouTube入力安定化の実装

Status: Implemented, runtime validation pending

Decision:

- ADR-0020として、watch page経路を維持したまま画質固定とtelemetryで安定化する方針を採用した。
- 既定のYouTube player qualityを`large`（480p相当）にした。
- `YOUTUBE_PLAYER_QUALITY`と`YOUTUBE_PLAYER_QUALITY_LOCK`で画質固定を切り替えられるようにした。
- `playback-telemetry.jsonl`へ1秒ごとの`readyState`、`networkState`、`bufferedAhead`、YouTube quality、available qualities、total/dropped framesを保存するようにした。
- URL、video ID、Cookie、account情報はtelemetryへ保存しない方針を維持した。

Validation:

- 実配信とYouTube再生の長時間runtime validationは未実施。
- 次はbrowser-only 20分、browser-to-v4l2 20分、full pipeline 10分以上の順に検証する。

## 2026-07-08: 音声入力と720p modeの実装

Status: Implemented, runtime validation pending

Change:

- `STREAM_PROFILE=720p`を追加し、YouTube player qualityの既定を`hd720`へ切り替えられるようにした。
- 720p modeでも既定運用を変えず、明示指定した時だけ`hd720`固定を試す方針にした。
- V4L2 ready metadataとpipeline stateへprofile、解像度、fps、YouTube qualityを残すようにした。
- PulseAudioの`yt_sink`と`yt_sink.monitor`を準備する音声route scriptを追加した。
- `ENABLE_AUDIO=1`の時だけEmulatorを`-allow-host-audio`付きで起動し、Chromium音声を仮想sink、Emulator入力をmonitor sourceへ向ける実装にした。
- 既定では音声OFFを維持し、従来の映像-only pipelineは`-no-audio`のままにした。

Validation:

- `scripts/12-audio-routing-prepare.sh`でPulseAudio null sink `yt_sink` と source `yt_sink.monitor` の作成を確認した。
- shell構文確認とNode構文確認は通した。
- Android guest内でマイク入力として認識されるか、斉藤さんLIVEへ音声が乗るかは未検証。
- `STREAM_PROFILE=720p`の長時間再生安定性は未検証。過去検証では720p相当でYouTube watch pageがstallしたため、まず短時間preview、その後10分以上のfull pipelineで確認する。

## 2026-07-08: 縦型YouTube動画向けcapture profile追加

目的:

- 携帯camera比率の縦型YouTube動画を、横長720pではなく縦長の仮想camera入力として扱う。

実装:

- `STREAM_PROFILE=vertical`を追加した。
- aliasとして`portrait`と`mobile`を`vertical`へ正規化する。
- 既定値は`VIDEO_WIDTH=720`、`VIDEO_HEIGHT=1280`、`VIDEO_FPS=30`、`YOUTUBE_PLAYER_QUALITY=hd720`、`YOUTUBE_PLAYER_QUALITY_LOCK=1`。
- `browser-player.mjs`側のviewport/screenも同じprofile既定値へ合わせた。

注意:

- YouTube URL、video ID、Cookie、account情報は記録しない。
- 実配信開始は別途人間の明示承認を必要とする。

Validation:

- viewer profileを使い、権利処理済みの縦型YouTube watch pageでbrowser-only 30秒captureを実行した。
- YouTube標準UIでfullscreenに入り、chat/replay panelを閉じてから録画する方式に変更した。
- persistent profile復元tabとChrome infobarがX11grabへ混入しないよう、toolbarなしapp windowとX11 close操作を追加した。
- 生成物はH.264 / yuv420p / 720x1280 / 30fps / 30秒。
- `maxBlackDuration=0`、`maxFreezeDuration=0`、validation error/warningなし。
- player stateは`paused=false`、`stalledForSeconds=0`、YouTube quality `hd720`。
- state artifactにはYouTube queryとvideo IDを保存していない。
- この検証ではEmulator、斉藤さん、外部配信は起動していない。

## 2026-07-08: vertical-in-720p 実配信で外部視聴が黒画面

Status: Failed, stopped

Scope:

- `STREAM_PROFILE=vertical-in-720p`
- browser capture: `720x1280`
- virtual camera output: `1280x720`
- audio: disabled
- 斉藤さんLIVE: フレンド限定

Observed:

- iPhone実機の外部視聴で映像が黒画面。
- ユーザー報告後、配信を終了し、結果画面まで到達した。
- ローカルpipeline、Emulator、ADB接続も停止済み。

Evidence:

- Artifact: `/home/rwatanabe/android-live-camera-poc/artifacts/youtube-emulator/20260708T183052Z/`
- YouTube telemetryは最終時点で`playbackReady=true`、`paused=false`、`readyState=4`、`videoWidth=720`、`videoHeight=1280`、`youtubeQuality=hd720`、`stalledForSeconds=0`。
- FFmpegはX11grab `720x1280`からV4L2 `/dev/video0`へ`1280x720` `yuyv422` `30fps`で出力継続。
- V4L2 formatは`1280/720`、Pixel Format `YUYV`。
- LIVE中のUI dumpでは`Layout_HostCamera`配下に`android.view.SurfaceView`が存在し、Saitousanはカメラ表示状態。

Current inference:

- YouTube再生停止やFFmpeg停止ではなさそう。
- 黒化地点は`/dev/video0`以後、特にEmulator camera HAL → Saitousan preview/配信encode → 外部視聴の境界が濃い。
- ADB screenshotでは`SurfaceView`内容が信頼できないため、次回は配信前にcamera inputを別経路でreadbackし、配信中は`dumpsys media.camera`とlogcat camera/codecを保存する。

Next:

1. 実配信前に`/dev/video0` readbackを自動検査し、非黒frameをartifact保存する。
2. LIVE開始直前・カメラON直後・外部視聴確認直後の3点で`dumpsys media.camera`、logcat camera/codec、UI XMLを保存する。
3. `vertical-in-720p`の`contain`出力がSaitousan側で黒化する可能性を切り分けるため、次回は`CAMERA_FIT_MODE=cover`または通常`720p`入力で比較する。

## 2026-07-08: 黒画面防止gateとblur背景の実装

Status: Implemented, local validated, live validation pending

Change:

- `STREAM_PROFILE=vertical-in-720p`の既定`CAMERA_FIT_MODE`を`contain`から`blur`へ変更した。
- `blur`はcover cropしたぼかし背景の上に縦動画を中央配置する。左右の純黒帯を避け、Saitousan側のcrop/encodeが黒帯だけを拾うリスクを下げる。
- `CAMERA_FIT_MODE=contain`と`cover`は引き続き指定可能。
- `scripts/03-youtube-browser-to-v4l2.sh`にV4L2 readback gateを追加した。ready前に`/dev/video0`から1 frameを読み戻し、平均輝度と黒pixel比率で黒画面を検出する。
- 黒判定時はEmulator起動前にfail-closedする。
- `scripts/08-youtube-to-emulator.sh`はpipeline stateへV4L2 readback artifact、camera dumpsys、camera logcat artifactを記録する。

Validation:

- `devbox run -- npm test`: pass、7 tests。
- `bash -n scripts/lib.sh scripts/03-youtube-browser-to-v4l2.sh scripts/08-youtube-to-emulator.sh`: pass。
- `STREAM_PROFILE=vertical-in-720p`でYouTube watch pageから8秒間`/dev/video0`へ出力し、readback gateが通過した。
- Artifact: `/home/rwatanabe/android-live-camera-poc/artifacts/manual/black-guard-vertical-in-720p/`
- readback metrics: `meanLuma=36.70`、`blackPixelRatio=0.3517`、`ok=true`。
- full pipeline smokeも配信開始なしで通過した。Artifact: `/home/rwatanabe/android-live-camera-poc/artifacts/manual/black-guard-full-pipeline/`
- full pipeline state: `cameraFitMode=blur`、`cameraDeviceCount=1`、`streamStarted=false`。
- full pipeline readback metrics: `meanLuma=34.25`、`blackPixelRatio=0.3582`、`ok=true`。

Remaining:

- 実配信の外部視聴で黒画面が解消したかは未検証。配信開始には人間の明示承認が必要。
- `cameraStreamHasExpectedSize=false`は、appがcameraを開く前の`dumpsys media.camera`にstream configurationが出ないためのsoft warningとして残した。次回LIVE中のcamera ON直後に再取得して確認する。

## 2026-07-08: Saitousan配信画面サイズに合わせるsquare入力

Status: Implemented, local validated, live validation pending

Context:

- 直近のSaitousan LIVE画面XMLでは、配信映像SurfaceView boundsが`[0,128][1080,1208]`だった。
- つまりpreview/配信映像枠は`1080x1080`の正方形。
- 既存`vertical-in-720p`はEmulator-safeのため`1280x720`へ変換していたが、Saitousan側の正方形枠とaspectが合わない。

Change:

- `STREAM_PROFILE=saitosan-square`を追加した。
- aliasとして`STREAM_PROFILE=square`も使える。
- browser captureは縦型`720x1280`を維持する。
- v4l2/webcam出力はSaitousan配信映像枠に合わせて`1080x1080`へ変換する。
- 既定fitは`CAMERA_FIT_MODE=blur`。coverしたぼかし背景の上に縦動画を中央配置し、純黒帯を避ける。
- `scripts/browser-player.mjs`も同profile/aliasを受け付けるようにした。

Validation:

- `bash -n scripts/lib.sh scripts/03-youtube-browser-to-v4l2.sh scripts/08-youtube-to-emulator.sh scripts/07-youtube-browser-record.sh`: pass。
- `node --check scripts/browser-player.mjs`: pass。
- `devbox run -- npm test`: pass、7 tests。
- `STREAM_PROFILE=saitosan-square scripts/00-host-check.sh`で`/dev/video0`が`1080x1080` YUYVへ設定されることを確認した。
- YouTube watch pageから8秒間`/dev/video0`へ出力し、readback gateが通過した。
- Artifact: `/home/rwatanabe/android-live-camera-poc/artifacts/manual/saitosan-square-v4l2/`
- readback metrics: `meanLuma=29.40`、`blackPixelRatio=0.3491`、`ok=true`。
- full pipeline smokeも配信開始なしで通過した。
- Artifact: `/home/rwatanabe/android-live-camera-poc/artifacts/manual/saitosan-square-full-pipeline/`
- full pipeline state: `streamProfile=saitosan-square`、`videoSize=1080x1080`、`cameraFitMode=blur`、`cameraDeviceCount=1`、`streamStarted=false`。
- `dumpsys media.camera`で`availableStreamConfigurations`に`1080 1080 OUTPUT`が出ていることを確認した。
- 残留Emulator/FFmpeg/Chromium processなし、ADB deviceなし。

Remaining:

- 実配信の外部視聴で黒画面が解消したかは未検証。次回のフレンド限定テスト配信は`STREAM_PROFILE=saitosan-square`を第一候補にする。

## 2026-07-08: Saitousan LIVE camera枠へのYouTube映像表示成功

Status: Validated, friend-only live stopped

Final working configuration:

- `STREAM_PROFILE=vertical-in-720p`
- `CAMERA_FIT_MODE=cover`
- `EMULATOR_CAMERA_BACK=webcam0`
- `EMULATOR_CAMERA_FRONT=webcam0`
- `EMULATOR_EXPECTED_CAMERA_FACING=BACK`

Result:

- YouTube watch pageの映像が、斉藤さんLIVEの配信画面camera枠へ表示された。
- フレンド限定・音声なしの短時間LIVEで確認した。
- テスト配信は停止し、結果画面から退出してハンカチ中継一覧へ戻った。

Evidence:

- `/home/rwatanabe/android-live-camera-poc/artifacts/youtube-emulator/20260708T194044Z/pipeline-state.json`
- `/home/rwatanabe/android-live-camera-poc/artifacts/youtube-emulator/20260708T194044Z/browser-v4l2/v4l2-readback.png`
- `/home/rwatanabe/android-live-camera-poc/artifacts/youtube-emulator/20260708T194044Z/screen-live-camera-on-cover.png`
- `/home/rwatanabe/android-live-camera-poc/artifacts/youtube-emulator/20260708T194044Z/camera-dumpsys-live-camera-on-cover.txt`
- `/home/rwatanabe/android-live-camera-poc/artifacts/youtube-emulator/20260708T194044Z/ui-after-exit.xml`

Observed state:

- `dumpsys media.camera` showed active client package `yudo.work.saitosan`.
- Camera ID 10 was open.
- `Facing: Back`.
- `Camera error traces (0)`.
- UI XML had `Layout_HostCamera` with `android.view.SurfaceView`.
- ADB screenshot showed the YouTube frame inside the Saitousan LIVE camera area.

Correction of prior hypothesis:

- `saitosan-square` + front-only (`-camera-back none -camera-front webcam0`) produced a non-black UI state, but Saitousan did not open the camera and YouTube映像 was not shown.
- The accepted runtime default for this PoC should be the BACK camera registration path created by passing `webcam0` to both back and front emulator camera options.
- `vertical-in-720p` with `cover` avoids black bars and matched the successful display better than `blur` or `saitosan-square` in this test.

Remaining:

- Audio injection remains unresolved and should be handled separately.
- Longer duration stability should be retested after this camera path is fixed as default.

## 2026-07-09: side-clip（左右clip・左寄せ）で横型YouTubeを斉藤さんLIVEへ

Status: Mostly OK（ユーザー確認）。微調整は残る。

Target:

- URL: `https://www.youtube.com/watch?v=2Csu9A8YVow`（礼賛「高ぶるブルー」、4:3）
- Handoff: `/home/rwatanabe/android-live-camera-poc/artifacts/manual/HANDOFF-2026-07-09-side-clip.md`

Working configuration:

- `STREAM_PROFILE=stable`
- `CAMERA_FIT_MODE=side-clip`
- ffmpeg: `crop=ih:ih:(iw-ih)/2:0,scale=720:720,pad=1280:720:0:(oh-ih)/2:color=black`
- Emulator camera: BACK/FRONT `webcam0`、facing BACK
- フレンド限定 / 音声なし

Findings:

- 斉藤さんホスト枠は `Layout_HostCamera` = 1080x1080。
- landscape カメラは高さ合わせ＋左寄せで枠に載るため、正方形コンテンツは **中央padではなく左寄せpad** が必要。
- プレビュー用に `scripts/15-saitosan-frame-preview.sh --mode side-clip-1080` を追加。
- 短尺VOD終端で browser-player が stalled するため、終端ループを追加。

Evidence:

- `/home/rwatanabe/android-live-camera-poc/artifacts/youtube-emulator/20260709T042316Z/fix2-05-camera.png`
- `/home/rwatanabe/android-live-camera-poc/artifacts/youtube-emulator/20260709T042316Z/browser-v4l2/v4l2-left-now.png`
- `/home/rwatanabe/android-live-camera-poc/artifacts/youtube-emulator/20260709T042316Z/pipeline-state.json`

Remaining:

- iPhone側での最終フレーミング微調整
- 再生解像度の改善
- 音声・長時間安定性は別途
