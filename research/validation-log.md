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
5. YouTube watch page captureを`/dev/video10`へ流し、斉藤さんcamera previewで確認する。

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
- 次はcreator権限を持たない専用viewer accountへ人間がloginしたprofileで再検証する。

Remaining:

- 配信者本人の権利処理済みYouTube Live watch pageで再生readyを実測する。
- 斉藤さん内のcamera preview有無と`webcam0`映像認識を確認する。
- 音声入力は未接続。
- ADR-0019は`Proposed`のまま。
