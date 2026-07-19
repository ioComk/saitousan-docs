# ADR-0021: Android Emulator音声入力をgRPC injectAudioへ段階移行する

## Status

Proposed

## Context

iPhoneからMediaMTXへ送った音声は、MediaMTXのWebRTC viewerで正常に再生できている。2026-07-19の再検証でも、合成WHIP入力からRTSPへH.264 videoとOpus `48 kHz / 2ch` audioが出ること、RTSP音声をFFmpegでPulseAudio null sinkへ出した時点で1 kHzトーンを`mean_volume=-24.1 dB`、`max_volume=-21.0 dB`として検出できることを確認した。

一方、現在のEC2 native構成はAndroid Emulator 36.6.11を`-audio pa -allow-host-audio`で起動している。Emulator内の`AudioRecord`で測ったRMSはトーン入力中も約`-92〜-93 dBFS`で、journalには次のエラーがある。

- `Could not init pa audio driver`
- `Failed to initialize PA context`

したがって失敗点はMediaMTXではなく、PulseAudioからAndroid Emulatorのguest microphoneへ入る境界である。

2026-07-19の切替実装では、追加で次のEC2固有挙動を確認した。無音PCMを先に`injectAudio`へ送るとEmulator 36.6.11がSIGSEGVし、RTSP source切断時にPulse sink inputが消えるとPulseAudioが`memblock_replace_import()` assertionでABRTした。このため、非無音PCMを検出するまでinjectorを起動せず、Pulse sinkには常時無音のkeepalive inputを残す必要がある。

`android-live-camera-poc`では次の方式を実測済みである。

| 方式 | 実測 | 主な制約 |
| --- | --- | --- |
| PulseAudio direct (`-audio pa`) | 現EC2ではFAIL | EmulatorがPA contextを初期化できない |
| gRPC `injectAudio` | guest micまでPASS実績あり | guest micを先に開かないとEmulator 36.6.11がSIGSEGVする可能性がある |
| wav (`qemu_in.wav`) | guest micまでPASS実績あり | 約0.9秒の追加遅延、同一inode維持、Emulator cwd管理が必要 |

映像経路は既に成立しているため、音声方式の変更でMediaMTX、V4L2、Emulator camera登録を壊さないことも必要である。

## Decision

PoCのAndroid Emulator音声入力は、次の優先順位で段階移行することを提案する。

1. 第一候補をgRPC `injectAudio`にする。
2. gRPCの安定性または運用性が受入基準を満たさない場合のみ、wav方式を明示的fallbackとして評価する。
3. 現EC2 + Emulator 36.6.11の`-audio pa`は、既知の失敗方式として新しい音声構成には採用しない。
4. PulseAudio null sinkは廃止しない。MediaMTX RTSP音声をPCMへ変換し、gRPC injectorまたはwav feederへ渡すhost-side bufferとして利用する。
5. gRPC失敗時に配信中の自動fallbackは行わない。音声断、Emulator再起動、映像停止を隠さず、fail-closedでテストを停止する。
6. MediaMTX sourceが無音または未接続の間はgRPC injectorを開始しない。`mediamtx_camera` sinkは無音keepaliveで維持する。

提案するgRPC経路は次の通り。

```text
iPhone microphone
  -> WHIP / WebRTC
  -> MediaMTX iphone-camera
  -> RTSP audio
  -> FFmpeg PCM
  -> PulseAudio mediamtx_camera.monitor
  -> gRPC injectAudio (loopback only)
  -> Android Emulator Built-In Mic
  -> 斉藤さん RECORD_AUDIO
```

MediaMTXのviewer経路は変更しない。

```text
MediaMTX -> WHEP / WebRTC viewer
```

### 起動順序

Emulator 36.6.11の既知クラッシュを避けるため、gRPC経路は次の順序を必須にする。

1. PulseAudio sink/sourceを作成し、無音keepaliveを開始する。
2. Emulatorを`-audio none -allow-host-audio -grpc 8554`で起動する。
3. ADB boot完了を待ち、`adb emu avd hostmicon`を実行する。
4. MediaMTX monitorの非無音PCMを確認する。無音中は`waiting_for_audio`で待機する。
5. MicHoldでguest `AudioRecord`を開く。
6. gRPC injectorを開始する。
7. injectorのframe増加とhost PCM RMSを確認する。
8. MicHoldの`AudioRecord`を再作成し、非無音RMSへlatchしたことを確認する。
9. 斉藤さんへ渡す直前にMicHoldをsoft-stopする。
10. 斉藤さんがマイクを開いた後、必要ならアプリ内のマイクOFF→ONで`AudioRecord`を再作成する。

MicHoldは起動時のlatch補助であり、通常配信中の音声consumerにはしない。injector稼働中にMicHoldを無条件force-stopしない。

### サービス境界

実装時は次の責務に分ける。

| Component | Responsibility |
| --- | --- |
| `mediamtx-virtual-camera.service` | RTSP映像をV4L2へ、RTSP音声をPulseAudio sinkへ出す |
| `android-emulator.service` | webcam0付きEmulatorを`-audio none -grpc`で起動する |
| 新規`emulator-audio-inject.service` | ADB待機、hostmicon、MicHold、injector、latch、状態記録を管理する |
| validation scripts | host RMS、inject frame、guest RMS/HAL、camera非回帰を判定する |

Emulator 36.6.11のgRPC portは`-grpc 8554`でwildcard bindになるため、`emulator-grpc-firewall.service`のIPv4/IPv6 firewall ruleでnon-loopbackのTCP/8554を拒否し、実効的にlocal-onlyへ制限する。firewallなしでEC2 private/public interfaceから到達できる場合は受入不可とする。

### 採用ゲート

このADRを`Accepted`へ変更する条件は次とする。

1. 合成入力
   - WHIP/RTSPにaudio trackがある。
   - host Pulse RMSが`-50 dBFS`より大きい。
   - injectorのframe数が継続増加し、inject直前PCM RMSが`-50 dBFS`より大きい。
   - MicHoldのguest RMSが`-60 dBFS`より大きい。
   - guest HAL判定が`-40 dB`より大きい、または同等の非無音証跡がある。
2. 安定性
   - 合成音声・映像を10分連続投入してEmulator crash、inject停止、camera消失がない。
   - Emulatorまたはaudio injectorを個別再起動した後、規定順序で復旧できる。
3. 実iPhone
   - senderでcamera/microphone permissionを許可し、MediaMTX viewerとguest micの両方で音声を確認する。
   - 外部配信なしで5分以上継続する。
4. 斉藤さん
   - テストアカウントでマイク入力状態を確認する。
   - フレンド限定LIVEの外部耳確認は、対象・通知設定・停止条件を示して人間の明示承認後にだけ行う。
5. Security / operation
   - gRPC portがloopback onlyである。
   - token、Cookie、OTP、password、個人情報をログやartifactへ保存しない。
   - 失敗時に映像-onlyの既知状態へrollbackできる。

## Consequences

良くなること:

- MediaMTXまで正常な音声を、既にPASS実績のあるEmulator入力方式へ接続できる。
- host、injector、guestの3地点でRMSを測れるため、無音位置を判定しやすい。
- PulseAudio directの既知失敗を繰り返さずに済む。
- 映像経路と音声経路の責務を分離できる。

難しくなること:

- MicHoldを含む厳密な起動順序とlatch処理が必要になる。
- systemd unitとhealth checkが1つ増える。
- gRPC APIはEmulator version依存があり、version更新時に再検証が必要になる。
- 理論上約300 msのinject bufferに加え、MediaMTX、app、外部配信の遅延が加わる。

リスク:

- guest micを開く前に`injectAudio`を開始するとEmulatorがSIGSEGVする可能性がある。
- injectorが停止しても映像は継続できるため、音声断を見逃す可能性がある。
- source切断・再接続時にAudioRecord latchが外れ、マイクOFF→ONが必要になる可能性がある。
- 未認証のgRPC endpointを外部公開すると、任意音声注入の攻撃面になる。
- `iphone-live-camera-ec2`の現在のcheckoutはGit worktreeではないため、実装source of truthを確定せずに配置済みファイルを直接編集すると変更管理できない。

## Alternatives Considered

### PulseAudio directを継続する

利点:

- 構成が単純で遅延が小さい可能性がある。

不採用理由:

- 現EC2 + Emulator 36.6.11でPA context初期化が失敗している。
- host PulseAudioまで信号がある一方、guest micが無音であることを再現済み。
- Emulator versionまたはhostを変更して再検証するまでは採用根拠がない。

### wavを第一候補にする

利点:

- guest micまでPASS実績がある。
- gRPC crash条件を避けられる。

第一候補にしない理由:

- `qemu_in.wav`のinode、cwd、prefill/leadを管理する必要がある。
- 追加遅延がgRPCより大きい。
- source再接続や長時間運用の保守が複雑になる。

### EmulatorまたはEC2 hostを変更する

利点:

- `-audio pa`が正常に動く構成へ移れる可能性がある。

現時点の扱い:

- gRPC/wavが受入基準を満たさない場合の次候補とする。
- instance、OS、Emulator versionを同時に変えず、変数を1つずつ評価する。

### 物理端末または物理音声deviceを使う

利点:

- Emulator固有のaudio backend問題を回避できる。

現時点の扱い:

- EC2 headless PoCの目的から外れるため、最後のfallbackとする。

## Notes

- 実施手順: `architecture/emulator-audio-transport-switch-runbook.md`
- 実測結果: `research/validation-log.md`
- 参照実装: `/home/rwatanabe/android-live-camera-poc/scripts/13-emulator-grpc-audio-inject.mjs`
- MicHold: `/home/rwatanabe/android-live-camera-poc/scripts/21-emulator-mic-hold.sh`
- fault isolate: `/home/rwatanabe/android-live-camera-poc/scripts/23-grpc-audio-fault-isolate.sh`
- 背景調査: `/home/rwatanabe/android-live-camera-poc/docs/host-audio-to-emulator-mic.md`
