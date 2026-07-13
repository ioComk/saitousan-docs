# Phase A 検証計画: mediaMTX WHIP ingest

対象ADR: [ADR-0021](../adr/0021-webrtc-mediamtx-owned-ingest-fanout.md)

このリポジトリはドキュメント検討用である。実装コード、バイナリ、実行ログはここに置かない。
実測は別リポジトリ（例: Android live camera PoC）または EC2 上の作業ディレクトリで行い、結果だけを `research/validation-log.md` に要約して戻す。

## Goal

一次配信元をスマホ WebRTC に移す前段として、mediaMTX 単体の ingest 成立を確認する。

範囲に含む:

1. EC2 などで mediaMTX を単体起動する
2. ブラウザ / OBS / テストクライアントから WHIP で映像・音声を publish する
3. ローカル再生（WebRTC 再生ページ / RTSP / HLS）で受信を確認する

範囲外:

- YouTube RTMP push（Phase B）
- 斉藤さん bridge / Emulator（Phase C）
- 認証・TLS・TURN の本番設計（必要なら別メモ）

## Success criteria

- 5分以上、映像と音声が途切れない
- 切断後の再接続挙動を記録できる
- mediaMTX path 上で video + audio track を確認できる

## Suggested minimal config（参考、secretsなし）

実装置き場は別環境。ここでは検討用の最小例のみ示す。

```yaml
logLevel: info

api: yes
apiAddress: 127.0.0.1:9997

rtsp: yes
rtspAddress: :8554

rtmp: yes
rtmpAddress: :1935

hls: yes
hlsAddress: :8888

webrtc: yes
webrtcAddress: :8889
webrtcEncryption: no
webrtcAllowOrigin: '*'
webrtcLocalUDPAddress: :8189
webrtcIPsFromInterfaces: yes
webrtcAdditionalHosts: []  # EC2の到達可能IPを必要に応じて追加

paths:
  cam:
    source: publisher
```

## Endpoints（PoC想定）

| Role | URL |
| --- | --- |
| Browser publish | `http://<host>:8889/cam/publish` |
| WHIP | `http://<host>:8889/cam/whip` |
| Browser playback | `http://<host>:8889/cam` |
| HLS | `http://<host>:8888/cam/index.m3u8` |
| RTSP | `rtsp://<host>:8554/cam` |

認証なし前提。共有ホストや公開IPでは使わない。

## Procedure

### A1. mediaMTX 起動

1. 検証用 EC2 / ホストに mediaMTX を置く
2. 上記相当の設定で起動する
3. API またはログで WebRTC listener 起動を確認する

### A2. WHIP publish

候補:

- ブラウザの `/cam/publish`
- OBS Studio の WHIP 出力
- 既存の WHIP 対応テストクライアント

確認:

- path `cam` が ready / online になる
- tracks に video と audio が両方出る

### A3. ローカル再生

1. `/cam` の WebRTC 再生、または HLS / RTSP で受信する
2. 映像と音声の有無を確認する
3. 5分継続する

注意:

- HLS はコーデック制約がある。WHIP 側が VP8 のみだと HLS で映像が落ちることがある
- Phase A の本確認は WebRTC 再生または RTSP を優先し、HLS は補助にする

### A4. 切断 / 再接続

1. publisher を止める
2. path が offline になることを確認する
3. 再 publish して復帰することを記録する

## Record back to this repo

実測後、`research/validation-log.md` に次を追記する。

- 日付、実施ホスト（EC2 type など個人情報なし）
- publisher 種別（browser / OBS / other）
- 継続時間
- tracks（例: H264 + Opus）
- 再接続の成否
- 見つかった制約（NAT、コーデック、ポートなど）
- 次アクション（Phase B に進む / 差し戻し）

artifact 本体、stream key、Cookie、端末固有情報は持ち込まない。

## Exit criteria for ADR-0021 Phase A

次を満たしたら Phase A 完了とし、Phase B 検討に進む。

- 上記 Success criteria を満たす検証ログが残っている
- 「このリポジトリ外のどこで再現するか」が一文で分かる
- 公開運用前提の認証は未実装でも、リスクとして明記されている
