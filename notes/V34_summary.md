# V34 — Professional S/R & ATR Optimized Trendlines (Pine v6) — סיכום למפתח

## V35 UPDATE (Outlier Candle Filter) - applied on top of V34 (in /workspace/V34.txt)
- New input group 'Outlier Candle Filter': show_outlier_filter (bool, default true), outlier_atr_mult (2.5), outlier_range_mult (8.0), outlier_body_pct_min (55.0).
- Block inserted right before pattern detection (~line 1739): ha_atr_val = ta.atr(ha_internalAtrLength) (3rd ATR call site -> Pine reuses buffer #2, zero extra cost), candle_true_rng (True Range vs close[1]), three criteria:
  1. outlier_vs_atr: TR > HA-ATR * mult, only when trend_dir != 0.
  2. outlier_vs_trend: (high-low) > |ha_haClose - trend_start_price| * range_mult.
  3. outlier_engulf_weak_body: engulfing with body% of range < threshold.
- Effects: bull/bear_engulfing gated by all 3; is_bull/is_bear_pattern force-false via := if any criterion; sweeps vetoed by criteria 1+2 only (wicks legit for sweeps); red 'X' tiny label marks filtered candles. Title renamed V35. File uses CRLF line endings.

## Header
- indicator(..., overlay=true, max_boxes_count=500, max_lines_count=100, max_labels_count=500)
- Shallow Git history: single commit "Add files via upload". File lives at /workspace/V34.txt.

## Inputs (קבוצות)
- ליבה: pivot_len(10), max_silent_age(150), max_active_zones=6(קבוע), lookback_pivots(20), atr_mult_trend(0.2), min_slope_atr(0.05), max_trendline_length(300), atr_zone_mult(0.3)
- Market Structure: show_ms_trend, show_choch, ms_line_width
- Fibonacci: show_fib, fib_sensitivity(0.214)
- Volume Profile: show_vp, ltf_res('1'), bins_count(50)
- Liquidity Sweeps: show_sweeps, ltf_absorption_threshold(60%), max_history_per_level(10)
- FVG: show_fvg, fvg_filter_atr(20%), max_fvg_boxes(20)
- Patterns: local_extr_bars(5)
- HA Fractal MS: show_ha_labels, ha_minBarsBetweenPivots(5), ha_internalN(2), ha_internalAtrLength(14), ha_internalAtrMult(1.0), ha_internalFloorPct(0.2)
- Position Mgmt: show_setups, min_score_threshold(65), rr_ratio(2.0)
- MAs: show_ema25/50/200 ; VWAP: show_vwap, show_vwap_bands, vwap_period(יומית/שבועית/חודשית)
- RVOL: rvol_min_threshold(0.8), breakout_window(3), show_rvol_label

## Colors
res=#c23131, sup=#0a846d, bear=#c23131, bull=#0a846d, choch=white 30%, liq/vp_bull=#00bcd4, vp_bear=#9c27b0, fib618=#ffc107, fib500=#ff9800, fib382=#f44336

## Types
- DivergenceLabel{lbl, invalidation_price, is_bullish}
- SweepLabel{lbl, invalidation_price, is_bullish}
- FVGBox{bx, top, bottom, is_bullish, is_broken, gap_pct, start_bar}
- SetupVisuals{entry_line, setup_lbl, sl_line, tp_line, entry_price, sl_price, tp_price, is_long, creation_bar, status, score, triggered, trigger_price, breakout_level}
- Zone{bx, top, bottom, start_bar, last_touch_bar, touches, is_res, is_active, is_flipped, break_count}

## Core computed values
- atr_val=ta.atr(14); label_base_offset=atr*1.5; rsi_val=rsi(close,14); ema25/50/200 with color by side of close
- VWAP: cumulative tp=hlc3 reset per timeframe.change(D/W/M); std bands ±1σ ±2σ; tracks vwap_period_start_bar
- HA series from request.security(same tf)[open,high,low,close]; ha_haClose=(o+h+l+c)/4; ha_haOpen recursive avg; green/red runs; ha_trend state machine (wave detection sets ha_scanHighNow/ha_scanLowNow)

## HA Fractal Pivot Engine (lines ~402-756)
- Internal pivots via ta.pivothigh/low(ha_trueHigh/Low, ha_internalN) + ATR-based depth filter (max(atr*mult, close*floor%))
- Arrays: ha_highBars/Prices/Labels/RSI, ha_lowBars/... ; pending_* arrays feed confirmation loop
- Main pivot logic on scan flags: window = ha_lastUpWave+2; merge vs new main pivot (ha_minBarsBetweenPivots); labels H/HH/LH and L/HL/LL; backfill internal pivots between prev & new main pivot; cap arrays at 50
- trend_absolute_extreme updated when bIndex > trend_start_bar in trend_dir

## Pending pivot -> Zone + Divergence processing (~757-900)
- For each confirmed PH/PL: divergence check vs previous same-side pivot using RSI + trend_dir => 'Divergence' label stored in div_labels (invalidated later)
- Zone creation: half_width=(atr*atr_zone_mult)/2; merges into active or silent zone if candle intersects or zone overlaps; silent zones capped 50/res and 50/sup
- Silent->Active promotion (1033-1052): touches>=2&age>=10 OR >=3&age>=5 OR >=4&age>=30; delete after break_count>=2 or age>max_silent_age
- Active zone flip on first break (is_res flips, touches=1); flipped zone deleted on re-break; center adjusts toward wick beyond center; cap active_zones to max_active_zones (FIFO delete oldest box)

## LTF order flow (901-926)
- request.security_lower_tf(ltf_res='1') => buy/sell volume split by ltf candle direction (doji 50/50); fallback to current candle sign; buy_pct_ltf/sell_pct_ltf; cum_buy_vol/cum_sell_vol accumulate on confirmed bars

## Label invalidation loops (927-944): div_labels & sweep_labels removed when price crosses invalidation_price

## Level touch volume history (945-991)
- extreme_high/low, choch_high/low, sr_hist arrays store [volume, buy/sell pct] per touch (cap max_history_per_level)

## Impulse metrics (995-1005): impulse_vol=cum total at new trend_extreme; impulse_bars = bar_index-trend_start_bar
## lowest/highest_since_choch tracking (1006-1008)

## Pre-Divergence labels (1009-1032): 'Pre-Div' when new extreme w/ worse RSI than last confirmed pivot; label follows extreme; deleted otherwise

## BOS/CHOCH/MS Trend engine (1152-1540)
- ms_point_a/b/c derived from latest HH & LL sequence (1091-1151): a=more recent extreme, b=opposite, c=prior opposite pivot before a
- ref_price = c else b; signal: a_is_high ? (close<ref => bear else bull) : (close>ref => bull else bear)
- choch_broken: close crosses choch_pivot_val against trend => full reset (trend_dir=0, delete lines/labels/fibs, clear_all_labels(), close pending setups status 0/3, clear vol hist arrays, traded_entries_prices)
- valid_bull/bear_bos: close beyond trend_absolute_extreme whose idx>trend_start_bar => set bos line/label, confirm trend (ms_trend_line color), compute new choch_pivot as min low/max high since extreme, draw CHOCH dashed line+label
- On ms_bull/bear_signal & trend change: massive reset block then set trend_dir, trend_start_bar/price, choch_pivot=start, init trend_absolute_extreme from HA pivots after start, gray 'MS Trend Start (Bull/Bear - Pending)' line+label, CHOCH line
- find_best_trend(indices,values,is_resistance,atr,trend_start,min_slope,max_len): pairwise pivot lines, slope>=min_slope_atr*ATR, tolerance=atr*atr_mult_trend, no close breaks (scan every 2 bars up to 300 back), score touches>=3 with right_score>=1 after trend_start; picks best by (right_score,total,length); returns extend.right line
- find_parallel_line: opposite-side pivots with matching slope within tolerance, score>=2, positioned correctly relative to main line => dashed parallel
- Recompute trendlines only when new_pivot_added; delete bull/bear lines when close crosses them

## FVG (1551-1604)
- Only when trend_confirmed & bar>=trend_start_bar; bull gap: high[2]<low & high[2]<high[1] & low[2]<low & size> fvg_filter_atr % of ATR; overlap check vs existing unbroken same-side FVG; box text '% ATR'; cap max_fvg_boxes FIFO; broken flag when price fully closes through (low<bottom for bull / high>top for bear) => transparent

## Confluence flags (1605-1680) — long & short versions for:
- SR zones (tol .1ATR), HVA boxes (from vp_boxes + hva_is_bullish), FVG (.1ATR), Fib levels 382/500/618 (.1ATR), trendlines bull/bear (.2ATR), EMAs 25/50/200 (.15ATR)

## Fibonacci pullback display (1681-1731)
- ratio=(extreme-close)/total_move >= fib_sensitivity => draw 0.382/0.5/0.618 lines+labels from trend_start_bar..bar_index; also fills Buy/Sell volume table (vol_table top-right) with proportional widths; else clears

## Candlestick patterns (1732-1823)
- bull_engulfing, hammer, shooting_star, morning/evening star, bear_engulfing; pattern_name string
- context validity: near pattern_extreme (+ATR*.5) or in pullback zone; confluence_count sums the six confluence bools + pre-div proximity (<=2 bars) + absolute-extreme touch
- Labels shown only at local extremes (ta.highest/lowest(local_extr_bars)); invalidated when price exceeds bull_pattern_low/bear_pattern_high

## Liquidity Sweeps (1826-1936)
- Conditions: wick beyond level (trend_absolute_extreme, choch_pivot_val, active zone top/bottom, FVG edges) but close back inside; requires buy/sell_pct_ltf >= ltf_absorption_threshold; deviation vs historical avg arrays picks best sweep; 'Sweep (+x% Vol, +y% LTF)' label stored in sweep_labels with invalidation price
- FVG-based sweeps: bear sweep when high>FVG.top&close<=top&buy%>=thr (bearish FVG); bull symmetric

## RVOL (1937-1968)
- Intraday: project current volume by elapsed fraction, compare to same-time-of-day average over N days (20/10/5 by tf); Daily+: sma(volume,20); label 'RVOL x.xx' colored by threshold

## Setup lifecycle management (1969-2055)
- status codes: 0=active/live, 1=respected(win), 2=failed(SL), 3=pending breakout, 5=cancelled(window/trigger invalidate)
- Pending (status 3): V34 fix — cancel if close moves against trigger_price; cancel after breakout_window bars; activate (status 0, triggered) when high/low crosses breakout_level
- Active: entry fill when price reaches entry (triggered flag); invalidation if low/high breaches trigger_price vs lowest/highest_since_choch; respected when price moves 1R from entry; failed when close beyond SL; all extend visuals +10 bars

## Scoring & position creation (2056-2208)
- Triggers: (bull pattern & local trough) or bull sweep (mirror short); valid_trigger_position requires new lowest/highest since CHoCH or absolute extreme
- base 30; context +10 trend align, +10 confirmed; location +5 per confluence tier (1..5); pullback vol behavior (<0.8 impulse=>10, >1.2=>2, else 5); trigger combo 15/10/8; EMA direction penalty -5 each against; VWAP +/-8 side, +5 touch bounce, +3/-? sigma1 touch +3, sigma2 +5
- total clamped 0..100; needs >= min_score_threshold AND valid position
- entry=high(long)/low(short); SL=low-.2ATR (or bull_pattern_low-.1ATR) ; TP=entry±risk*rr_ratio
- Dedupe: skip if within 1ATR of traded_entries_prices unless better entry; V34 fix — skip duplicate pending within .5ATR entry; max 3 setups FIFO
- Visuals: gray dashed entry, red SL, green TP, label 'LONG|Score:x|⏳'

## Stats & AI outputs (2209-2379)
- barstate.islast: refresh zone/fib/fvg/ms/choch geometry; stats_table bottom-right: Total/Respected%/Failed/Cancelled
- Volume Profile calc on islast or period change: lookback from vwap_period_start_bar (cap 1000), bins=bins_count, buy/sell split from LTF arrays; top-5 bins with proximity spacing (5% of bins) => invisible vp_boxes (used for confluence) + visible hva_display_boxes with text '% | x% B/S'
- plot AI_STATUS = 1/-1 if any pending setup exists (long/short); AI_FP = hash of last 6 chars of ticker (base64 mod 1e9+7) — likely webhook payload channel
- plots: EMA25/50/200, VWAP + ±1σ/±2σ bands

## Notable quirks / potential bugs to keep in mind
- close[bar_index - j] indexing inside find_best_trend/parallel relies on dynamic offset (works but fragile/expensive)
- Zones created silently are invisible until promoted; update_zone_visuals hides inactive
- max_active_zones hardcoded 6 (not an input)
- ha_atrSeries uses current-TF ATR length param; int_ph/int_pl computed on ha_true* security values
- Duplicate variable name 'size_ha_high_init' style scoping fine in Pine
- VP boxes array used by confluence even though invisible
- No plot/plotshape for signals besides AI_STATUS/AI_FP; alerts would need alertcondition (none present!)

## V36 changes (S/R merge fixes) — applied on top of V35 in V35.txt
1. NEW function `merge_zone_with_pivot(Zone z, float p_val, int bIndex)` (line ~287, after clear_all_labels, before update_zone_visuals): geometric union (top=max, bottom=min), touches+1, last_touch_bar, extends box left if older pivot, calls update_zone_visuals.
2. In both pending loops (high & low): overlap test now ONLY `zone_overlaps` (pivot ATR band vs zone) — removed `candle_intersects` (full candle range) which caused over-merging; merged via helper (was touch-count only). 4 call sites.
3. `last_confirmed_ph/pl_price/rsi` updated ONLY when `i == size_pending_* - 1` (newest pivot) — was overwritten by every element incl. oldest.
4. Removed now-unused `int offset = bar_index - bIndex` locals (2x). No other refs to [offset].
5. Title bumped to 'V36 + S/R Merge Fix'. NOTE: local file had reverted VWAP bands to plain style_line/full-value vars (no _v dotted hack) relative to cd5e927 — scoring uses full values, plots solid LINE lw=2 (user sets Dotted in STYLE tab manually).
Commits: 7dcce14 (fixes), 280513b (offset cleanup), 7768b7e (title). File CRLF, 2425 lines, git clean.
