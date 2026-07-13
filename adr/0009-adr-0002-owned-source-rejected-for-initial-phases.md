# ADR-0009: ADR-0002の一次配信元は当面YouTubeを維持する

## Status
Proposed

## Context
Issue #6 は、ADR-0002で採っている「YouTube Liveを一次配信元にして斉藤さんLIVEへ流す」前提を再評価し、自前の配信起点を持つべきかを問うている。

自前起点は設計上はきれいで、将来的に「配信制御基盤」として一般化しやすい。一方で、このリポジトリの現状はまだ Phase 0 の成立確認前であり、Android実行環境、Appium操作、カメラ入力差し替え、ログイン維持、規約確認といった主要リスクが未解決である。

つまり、現時点で自前起点へ広げると、難所を減らすのではなく制御対象を増やす。PoCの焦点をぼかさず、既存ADRとの整合を保つ判断が必要である。

## Decision
初期フェーズでは、ADR-0002の前提を維持し、一次配信元はYouTube Liveまたは固定テスト映像のままとする。

自前配信基盤案は否定しないが、Phase 0〜2 の範囲では採用しない。扱いは「将来の拡張候補」であり、次の条件を満たした時だけ再評価する。

- Android側の配信開始/停止と状態確認が自動化できる。
- カメラ入力差し替え経路が10分以上安定する。
- YouTube起点による遅延、規約、運用負荷がボトルネックとして実測で確認される。

初期PoCの責務境界は次のままとする。

- source of truth: YouTube Liveまたは固定テスト映像
- bridge target: 斉藤さんLIVE
- control plane: Appium主体の外部自動操作

## Consequences
ADR-0002の検証範囲を維持できるため、最重要リスクであるAndroid入力経路へ集中できる。

一方で、自前配信起点による低遅延化、複数出力先への一般化、YouTube依存低減は後回しになる。YouTubeの取得方式や運用制約は引き続き受け入れる必要がある。

## Alternatives Considered

| Option | Pros | Cons | Current Judgment |
| --- | --- | --- | --- |
| YouTube一次配信を維持する | 既存ADRと整合し、録画/アーカイブも利用できる | YouTube依存、遅延、取得方式の制約が残る | 採用 |
| 自前RTMP/SRT/WHIP起点へ切り替える | 将来の配信基盤としては筋がよい | 制御面と運用面の責務が一気に増える | 初期フェーズでは不採用 |
| OBSローカル起点で最小構成にする | テスト動画注入の自由度が高い | リモート運用や再現性が弱い | 固定テスト映像の代替候補 |

## Risk Notes
- 自前起点へ早期に広げると、PoC失敗時に原因分離が困難になる。
- YouTube一次配信を維持すると、取得方式の変更や規約面の確認が引き続き必要になる。
- 自前起点案を棚上げすると、将来の再設計コストが発生する。

## Validation Plan

### Gate A: ADR-0002の成立確認

1. Android実行環境で斉藤さんアプリを起動する。
2. 配信開始直前の画面まで到達する。
3. 任意映像をカメラ入力として渡せるか確認する。

Success criteria:

- Android実行環境が成立する。
- 5〜10分の映像注入が継続する。
- Appiumで最低限の制御が通る。

### Gate B: YouTube起点の実測評価

1. 固定映像ではなくYouTube由来映像を入力にする。
2. 遅延、停止率、運用負荷、障害切り分けを記録する。
3. その時点でYouTube依存がボトルネックか評価する。

Success criteria:

- YouTube起点でも必要品質を満たす、または不足点が定量化される。

## Next Action
次の作業はADR-0002の Phase 0/1 を進め、YouTube起点をやめるべき理由が実測で出るかを確認すること。自前配信基盤の検討は、その結果が出るまで新規ADRへ昇格しない。

## Notes
Issue: https://github.com/suzuki-engineering/saitousan-docs/issues/6

このADRは、ADR-0002の検証範囲を不用意に拡張しないための境界記録である。

2026-07時点の更新:

- Androidカメラ注入の映像経路は短時間PoCとして成立した。
- 一方でYouTube watch page起点のstall / bot / 画質変動が運用上の論点になっている。
- 自前起点の再評価は [ADR-0021](0021-webrtc-mediamtx-owned-ingest-fanout.md) で開始する。
- ADR-0021がAcceptedになった時点で、本ADRを `Superseded` に更新する。
