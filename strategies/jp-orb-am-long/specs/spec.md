# ORB Pine Script 仕様書

## 概要
東京時間の寄り付き（9:00）からの ORB（Opening Range Breakout）上抜けを、複数フィルタと時間条件で判定してロングのみ取引する戦略。VWAP オフセット、出来高、日足トレンド、ORB レンジのフィルタを組み合わせ、利確は 2 段階、損切りは R 基準または % 基準。さらに LossCut と TimeStop を備える。

## 対象/前提
- `strategy()` で実装されたロング専用戦略。
- `process_orders_on_close=true` により、エントリーは確定足（クローズ）基準。
- 日足データは `request.security(..., "D", ...)` で取得し、`lookahead_off` で先読みを回避。

## 入力パラメータ
- ORB
`orbMinutes` ORB期間（1?60分）
`entryDelayMin` ORB終了後の待機（0?10分）
- 取引時間
`endHour`, `endMinute` エントリー/保有終了時刻
- 取引制限
`maxTradesPerDay` 1日最大トレード数
`maxConsecLosses` 連敗上限
- 出来高フィルタ
`useVolFilter` ON/OFF
`volLen` SMA期間
`volMult` SMA倍率
- VWAPフィルタ
`useVWAPFilter` ON/OFF
`vwapOffsetTicks` VWAP上方オフセット
- ブレイク条件
`breakExtraTicks` ORB高値への上乗せティック
- 利確/損切り方式
`exitMode` `"R"` or `"Percent"`
`tp1R`, `tp2R` R倍率
`take1QtyPct` TP1分割比率
`stopBufferTicks` ORB安値からのストップバッファ
`stopPct`, `take1Pct`, `take2Pct` %方式の値
- 診断表示
`showDiag` テーブル表示
- LossCut
`useLossCut` ON/OFF
`lossCutHour`, `lossCutMinute` 発動開始時刻
`lossCutMinLossR` 最小損失R
`lossCutUseVwapGate` VWAPゲート使用
`lossCutVwapOffTicks` LossCut VWAPオフセット
`lossCutUseBreakGate` Breakゲート使用
- ORBレンジフィルタ
`useOrbRangeFilter` ON/OFF
`orbMinTicks`, `orbMaxTicks` ORBレンジ許容
- バックテスト期間
`useDateFilter` ON/OFF
`btStartY/M/D`, `btEndY/M/D`

## 時間/セッション定義
- `openTS` 9:00（シンボルのタイムゾーン）
- ORB期間: `openTS` ? `orbEndTS`
- エントリー開始: `startTS`（ORB終了＋待機）
- エントリー/保有終了: `endTS`
- LossCut開始: `lossCutTS`
- 日跨ぎ: `timeframe.change("D")` で日次リセット

## ORB計算
- ORB高値/安値は `inOrb` の間で更新
- ORB終了後に `orbReady = true`

## 日足トレンドフィルタ（Regime Gate）
- `close(D) > SMA20(D) > SMA50(D)` のとき `trendOk = true`
- `useDailyTrendGate` OFF なら常に通過

## その他フィルタ
- 出来高: `volume > SMA(volLen) * volMult`
- VWAP: `close > vwap + vwapOffsetTicks * tick`
- ORBレンジ: `orbMinTicks <= (orbHigh-orbLow)/tick <= orbMaxTicks`
- 連続損失/最大回数: `consecLosses < maxConsecLosses` かつ `tradesToday < maxTradesPerDay`

## エントリー条件（ロング）
以下すべて満たすとブレイクアウト成立。
- ORB確定済み
- エントリー時間内
- バックテスト期間内
- ORBレンジOK
- 出来高OK
- VWAP OK
- 日足トレンドOK
- 日次トレード制限OK
- ポジションなし
- `close > breakLevel` かつ `close[1] <= breakLevel`（確定足で上抜け）

## エントリー動作
- `strategy.entry("L", strategy.long)`
- `tradesToday` 加算
- `L_entryBar` 記録
- `L_tp1Done` リセット

## 利確・損切りの計算
- `exitMode == "R"`
`stop = orbLow - stopBufferTicks * tick`
`R = entry - stop`
`tp1 = entry + R * tp1R`
`tp2 = entry + R * tp2R`
- `exitMode == "Percent"`
`stop = entry * (1 - stopPct/100)`
`tp1 = entry * (1 + take1Pct/100)`
`tp2 = entry * (1 + take2Pct/100)`

## 利確/ストップ挙動
- TP1到達後は `L_tp1Done = true`
- TP1到達後の残りポジはストップを建値に引き上げ
- `strategy.exit("L-TP1", qty_percent=take1QtyPct, limit=tp1, stop=stopPrice)`
- `strategy.exit("L-TP2", qty_percent=残り, limit=tp2, stop=stopAfterTp1)`
- エントリーバーと同一バーでのExitを避ける `bar_index > L_entryBar`

## LossCut
以下条件で強制クローズ。
- `useLossCut` ON
- ポジションあり
- TP1未達
- `close <= entry - R * lossCutMinLossR`
- `lossCutTS` 以降 `endTS` 以前
- エントリーバー以降
- `lossCutUseBreakGate` が ON なら `weakByBreak` のときのみ

## TimeStop
`time_close >= endTS` で全クローズ。

## 日次・連敗管理
- 日跨ぎでリセット
- 連敗数はクローズ時損益で更新
- 日次最大回数・連敗上限でエントリー抑制

## アラート
- `alertcondition` でブレイク条件の検知
- エントリー時 `alert()` を発火

## 診断HUD（テーブル）
- TP1/TP2/SL/LC/TS の回数
- TS が TP1 前/後
- TS 勝ち/負け/引き分け
- 直近の exit_id / exit_comment / PnL

## プロット
- ORB High/Low
- VWAP
- Break Level

## バックテスト期間フィルタ
`useDateFilter` ON の場合、範囲外なら保有中も `strategy.close_all` で終了。

