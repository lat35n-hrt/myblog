+++
date = '2026-01-08T20:36:21+09:00'
draft = false
title = 'Playwright Flake Lab'
categories = ["Playwright"]
+++


# クリック失敗がたまに起こる環境を作り、Playwrightの挙動を調べる

E2E テストが「たまに落ちる」状態は、プロダクト品質の議論より先に、開発の速度と信頼を削る。
一方でフレーク（flake）の議論は、体感や印象論に寄りやすい。

そこで今回は、フレークを意図的に注入し、50回実行で成功率と失敗理由を定量化し、さらに改善後の成功率が上がることを同じ条件で示す "Flake Lab" を作った。

この PoC の狙いは「テストが落ちるのは仕方ない」ではなく、落ち方を再現可能にして、原因を特定し、設計で潰すところまでを短いスコープで示すこと。

## ゴール

- UI フレーク（クリックがブロックされる）を確率付きで再現
- 50回実行して成功率と失敗理由を集計
- trace とログを証拠として残す
- 「待つべき対象を待つ」 という改善で成功率が上がることを示す

## 方針：フレークを「発生させて」「計測して」「潰す」

構成はシンプル。

1. フレークを注入する（わざと壊す）
2. 同条件で 50 回走らせ、成功率・失敗理由を集計する
3. ログで失敗モードを確定させる
4. 待機条件を設計して改善し、同条件で再計測する

「再現性」「分類」「証拠」「改善」を 1 本の PoC で閉じる。

## PoC の実行入口

この Flake Lab の本編は、pytest の自動収集とは切り離し、専用 runner から Playwright を実行します。

```bash

# Flake Lab
python scripts/run_flake_trials.py

```

## Flake の注入方法：Transient overlay（クリック遮断）

今回は、シンプルな UI フレークで動作を追うことにします。

- **Transient overlay（全画面オーバーレイ）** を一瞬だけ差し込む
- overlay が pointer events を奪い、クリックを遮る
- overlay は 300ms だけ存在
- 1回の試行で overlay を出す確率は 0.3
- 乱数 seed 固定で再現性を担保

技術的には、Playwright 側で DOM に `#__flake_overlay` を挿入し、一定時間後に消すだけです。

イベント（クリック）の遮断を意図的に作る。

```python
# inject overlay
def inject_overlay(page, duration_ms: int) -> None:
    page.evaluate(
        """
        (durationMs) => {
          const id = "__flake_overlay";
          const old = document.getElementById(id);
          if (old) old.remove();

          const el = document.createElement("div");
          el.id = id;
          el.style.position = "fixed";
          el.style.inset = "0";
          el.style.background = "rgba(0,0,0,0.01)";
          el.style.zIndex = "2147483647";
          el.style.pointerEvents = "auto";
          document.body.appendChild(el);

          setTimeout(() => { el.remove(); }, durationMs);
        }
        """,
        duration_ms,
    )
```

## テストフロー：代表フローを 1 経路に絞る

対象フローは 1 経路に固定しました。

```
一覧 → 詳細 → Approve
```

この "代表フロー" を 50回繰り返し、各試行の結果を JSONL で記録します。
失敗時は（設定により）trace も保存できるようにしておきました。

## 計測の前提：サーバ起動ノイズを除外する（health check）

計測のノイズ（サーバ未起動・起動途中）を除外するため、試行前に `GET /health` を確認します。
実装は標準ライブラリのみで、2秒タイムアウト、HTTP 200 以外は即失敗とします。

```python
def assert_server_up():
    with urlopen(f"{BASE_URL}/health", timeout=2) as r:
        if r.status != 200:
            raise RuntimeError(f"Server not ready: {r.status}")
```

これにより、フレークの議論を UI/同期の問題に集中させられます。

## 実行と測定：環境変数で制御する

実行条件は環境変数で固定しました。

- `BASE_URL`：テスト対象の URL
- `FLAKE_STRATEGY`：クリック戦略（naive / wait / off）
- `FLAKE_OVERLAY_MS`：overlay の表示時間（例：300ms）
- `FLAKE_OVERLAY_PROB`：overlay を入れる確率（例：0.3）
- `TEST_FLAKE_SEED`：乱数 seed（例：42）

ポイントは、同じ seed / prob / duration を使って before/after を比較することです。
ここが曖昧だと「たまたま直った」「今回は運が良かった」などが混じる。

## 実装の要点：overlay 注入は確率、差分は"待つ対象"

### overlay 注入は「毎回」ではなく「確率的」

各試行で次の条件を満たすときだけ overlay を注入します。

- `FLAKE_STRATEGY != "off"`
- かつ `random() < FLAKE_OVERLAY_PROB`（例：0.3）

注入したかどうかは `overlay_injected` としてログ（JSONL）に残します。

```python
# overlay 注入のロジック（抜粋）
overlay_injected = False

if FLAKE_STRATEGY != "off" and (_rng.random() < FLAKE_OVERLAY_PROB):
    overlay_injected = True
    inject_overlay(page, FLAKE_OVERLAY_MS)
```

### before/after の差分は "クリック前の同期設計"

改善点は「闇雲なリトライ」ではなく、待つ対象の明確化。

- **naive**：注入された overlay が生きている間にクリックすると失敗し得る
- **wait**：注入された場合、クリック前に overlay が消えるまで待つ

```python
# before/after の分岐（抜粋）
if overlay_injected and FLAKE_STRATEGY == "wait":
    # improved: wait overlay to disappear
    page.locator("#__flake_overlay").wait_for(
        state="detached",
        timeout=FLAKE_OVERLAY_MS + 2000
    )

if FLAKE_STRATEGY == "naive" and overlay_injected:
    # intentionally fragile: short timeout so click fails while overlay exists
    btn.click(timeout=200)
else:
    btn.click()
```

## 結果：Before / After を同条件で比較する（50回）

### Before（naive）：SuccessRate 60%

実行コマンド：

```bash
BASE_URL=http://127.0.0.1:8004 \
FLAKE_STRATEGY=naive \
FLAKE_OVERLAY_MS=300 \
FLAKE_OVERLAY_PROB=0.3 \
TEST_FLAKE_SEED=42 \
python scripts/run_flake_trials.py
```

出力（50回）：

```
Runs: 50
OK: 30, FAIL: 20, SuccessRate: 60.0%
Failure reasons:
  - click_timeout: 20
```

naive では、overlay が差し込まれた試行でクリックが遮られ、`click_timeout` が発生します。
「ボタンは見えているのに、透明な要素が上に被っていてクリックできない」典型例。

### After（wait）：SuccessRate 100%

実行コマンド：

```bash
BASE_URL=http://127.0.0.1:8004 \
FLAKE_STRATEGY=wait \
FLAKE_OVERLAY_MS=300 \
FLAKE_OVERLAY_PROB=0.3 \
TEST_FLAKE_SEED=42 \
python scripts/run_flake_trials.py
```

出力（50回）：

```
Runs: 50
OK: 50, FAIL: 0, SuccessRate: 100.0%
```

改善内容：

1. overlay を注入した試行では、クリック前に `#__flake_overlay` が DOM から消える（detached）まで待ちます
2. その後にクリックします

「待つ対象」を overlay の消滅に固定しています。リトライに頼らず同期設計でフレークを潰せることが結果から確認できました。

## ログ取得：JSONL と Playwright の call log

JSONファイルに出力されたログを比較します。

### Before（naive）：overlay ありで click_timeout（クリック遮断が原因）
```json
{"run_id": 45, "ok": false, "error_type": "click_timeout", "error_message": "Locator.click: Timeout 200ms exceeded.\nCall log:\n  - waiting for locator(\"button#approve\")\n    - locator resolved to <button id=\"approve\">Approve</button>\n  - attempting click action\n    2 × waiting for element to be visible, enabled and stable\n      - element is visible, enabled and stable\n      - scrolling into view if needed\n      - done scrolling\n      - <div id=\"__flake_overlay\"></div> intercepts pointer events\n    - retrying click action\n    - waiting 20ms\n    - waiting for element to be v", "elapsed_ms": 1627.665, "base_url": "http://127.0.0.1:8004", "ts_epoch_ms": 1767687410485, "step": "click_approve", "flake_strategy": "naive", "overlay_injected": true, "overlay_ms": 300, "overlay_prob": 0.3, "test_flake_seed": 42}
```

この call log が示しているのは、ボタン自体は「visible, enabled and stable」と判定されているにもかかわらず、#__flake_overlay が pointer events を intercept してクリックを遮断している、という事実です。
つまり「要素は見えているのにクリックできない」典型的な UI フレークです。

### Before（naive）：overlay なしなら OK
```json
{"run_id": 49, "ok": true, "error_type": "", "error_message": "", "elapsed_ms": 4023.302, "base_url": "http://127.0.0.1:8004", "ts_epoch_ms": 1767687421734, "step": "wait_approved", "flake_strategy": "naive", "overlay_injected": false, "overlay_ms": 300, "overlay_prob": 0.3, "test_flake_seed": 42}
```

### After（wait）：overlay ありでも OK（待つ対象を overlay 消滅に固定）
```json
{"run_id": 2, "ok": true, "error_type": "", "error_message": "", "elapsed_ms": 3706.224, "base_url": "http://127.0.0.1:8004", "ts_epoch_ms": 1767687944178, "step": "wait_approved", "flake_strategy": "wait", "overlay_injected": true, "overlay_ms": 300, "overlay_prob": 0.3, "test_flake_seed": 42}
```

wait 戦略では overlay 注入が発生しても成功することが、同じスキーマで確認できました。クリック前に #__flake_overlay が消える（detached）まで待つ、という 同期設計の変更が反映されています。

## なぜこの PoC が役立つのか

フレークの本質はだいたい以下のどれかです。

- 状態がまだ準備できていない（待つ対象が違う）
- UI が一瞬だけ変化する（アニメーション、overlay、スピナー）
- 非同期処理の完了が不定（API遅延、レンダリング遅延）

本 PoC では、フレークを「偶然」に任せず、

1. 再現条件を固定する（seed / prob / duration）
2. 失敗を分類する（reason）
3. エビデンスを残す（logs / trace）
4. 対策を当てて成功率で示す（before/after）

という手順を、最小スコープで確認できました。

## レンダリング方式（Jinja2）と今回の結論

なお、この PoC が扱っているのは 「レンダリング方式」 ではなく、DOM 上のイベント（クリック遮断）という DOM/event-level のフレークです。

今回の UI はシンプルな構造で、Jinja2（サーバー側テンプレート）を有効化しても DOM 構造やフローが実質変わらなかったため、同様の失敗モードと改善効果が再現できました。

ただし、UI 実装の変更で DOM 構造・セレクタ・タイミングが変わるような現実的なテストの場合は、待機条件の前提も変わるため、再調整が必要です。

## 付録：ディレクトリ構成（最小）

本記事の計測の中心は runner なので、以下の最小構成で走らせています。

```
scripts/run_flake_trials.py：50回実行ランナー（overlay 注入・集計・JSONL記録）
artifacts/runs/YYYYMMDD_HHMMSS_*：実行結果（JSONL、必要に応じて trace）
```

（UI 側の実装は `BASE_URL` が指すアプリケーション側に存在します。PoC の焦点は「クリック遮断の再現→原因特定→同期設計で改善」。）

## まとめ

- フレークは「運が悪い」ではなく、再現・分類・証拠・改善の対象
- overlay によるクリック遮断を確率付きで注入し、50回で成功率を測定
  - naive では 60%（click_timeout 20）
  - overlay を待つ戦略で 100%（同条件で再現）
- ログ（JSONL）と Playwright call log により、改善の議論を観測可能なデータに寄せられました
```