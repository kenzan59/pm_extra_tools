# Pacemaker の pgsql RA による wal_sender_timeout 動的調整機能

## 1. 背景と課題

### 1.1 同期レプリケーションにおける wal_sender_timeout の役割

PostgreSQL の同期レプリケーションでは、Primary が WAL（Write-Ahead Log）を Standby に送信し、
Standby からの応答を待ちます。`wal_sender_timeout` は、この応答が一定時間内に返ってこない場合に、Standby との接続を切断するためのタイムアウト値です。

#### タイムアウト判定の仕組み

`wal_sender_timeout` は、**最後に Standby から応答を受信してからの経過時間**でタイムアウトを判定します。WAL 送信時刻ではなく、応答受信時刻がタイマーの起点となります。

```
Primary                                       Standby
   |                                             |
   | [T=0] Last reply received <- Timer start    |
   | <-------------------------------------------|
   |                                             |
   | [T=3s] WAL sent                             |
   | ------------------------------------------->|
   |                                             |
   | [T=5s] Reply received <- Timer reset        |
   | <-------------------------------------------|
   |                                             |
   | [T=8s] WAL sent                             |
   | ------------------------------------------->|
   |                                             |
   |     (No reply - Standby failure)            |
   |                                             |
   | [T=5s + wal_sender_timeout] Timeout         |
   |     -> Disconnect from Standby              |
```

| 項目 | 説明 |
|------|------|
| タイマーの起点 | **最後に応答を受信した時刻** |
| タイムアウト条件 | 最後の応答から `wal_sender_timeout` 経過しても次の応答がない |

WAL の送信がない（アイドル状態の）場合でも、PostgreSQL は `wal_sender_timeout / 2` の間隔で keepalive メッセージを送信し、Standby の生存を確認します。

```
Primary                                       Standby
   |                                             |
   | [T=0] Last reply received                   |
   |                                             |
   |     (No WAL to send - Idle state)           |
   |                                             |
   | [T=wal_sender_timeout/2] Keepalive sent     |
   | ------------------------------------------->|
   |                                             |
   | Keepalive reply received <- Timer reset     |
   | <-------------------------------------------|
```

これにより、トランザクションがない状態でも Standby の障害を検出できます。

### 1.2 wal_sender_timeout と pgsql RA の連携

`wal_sender_timeout` によるタイムアウト検出から、pgsql RA による同期レプリケーション構成の変更までの流れを説明します。

#### 全体の流れ

```
[1] wal_sender_timeout の期限が過ぎる
        ↓
[2] PostgreSQL が該当 Standby との接続を切断する
        ↓
[3] PostgreSQL が pg_stat_replication テーブルから該当 Standby のエントリを削除する
        ↓
[4] Pacemaker が monitor 操作（monitor interval 間隔で実行される操作）を実行する
        ↓
[5] pgsql RA が pg_stat_replication テーブルを参照する
        ↓
[6] pgsql RA が該当 Standby の削除を検知する
        ↓
[7] synchronous_standby_names から該当 Standby を除外する
        ↓
[8] PostgreSQL に設定変更を反映する（reload）
```

#### 各ステップの詳細

**[1] wal_sender_timeout の期限が過ぎる**

Primary は、最後に Standby から応答を受信してから `wal_sender_timeout` の時間が経過しても次の応答がない場合、タイムアウトと判定します。

**[2] PostgreSQL が該当 Standby との接続を切断する**

タイムアウトが発生すると、Primary の walsender プロセスは該当 Standby との接続を切断します。この時点で、PostgreSQL のログに以下のようなメッセージが記録されます。

```
LOG:  terminating walsender process due to replication timeout
```

**[3] pg_stat_replication テーブルから該当 Standby のエントリを削除する**

接続が切断されると、`pg_stat_replication` ビューから該当 Standby の行を削除します。

```sql
-- タイムアウト前
SELECT application_name, state, sync_state FROM pg_stat_replication;
 application_name |   state   | sync_state 
------------------+-----------+------------
 standby1         | streaming | sync
 dr-standby1      | streaming | sync
(2 rows)

-- タイムアウト後（dr-standby1 がタイムアウト）
SELECT application_name, state, sync_state FROM pg_stat_replication;
 application_name |   state   | sync_state 
------------------+-----------+------------
 standby1         | streaming | sync
(1 row)
```

**[4] Pacemaker が monitor 操作を実行する**

Pacemaker は設定された間隔で pgsql リソースの monitor 操作を実行します。

**[5] pgsql RA が pg_stat_replication テーブルを参照する**

pgsql RA の monitor 処理内で、`control_slave_status()` 関数が `pg_stat_replication` テーブルを参照し、各 Standby の状態を確認します。

**[6] pgsql RA が該当 Standby の削除を検知する**

`pg_stat_replication` テーブルに存在しない Standby を検知します。

**[7] synchronous_standby_names から該当 Standby を除外する**

pgsql RA は、同期レプリケーションの構成を更新するため、`synchronous_standby_names` パラメータから該当 Standby を除外します。
これにより、トランザクションのコミットが該当 Standby の応答を待たなくなります。

```
# 変更前
synchronous_standby_names = 'FIRST 2 ("standby1", "dr-standby1")'

# 変更後
synchronous_standby_names = '"standby1"'
```

**[8] PostgreSQL に設定変更を反映する**

pgsql RA は `pg_ctl reload` を実行し、設定変更を PostgreSQL に反映します。

#### 障害検知時間の内訳

Standby 障害が発生してから、同期レプリケーション構成が変更されるまでの時間は以下の要素で構成されます。

```
wal_sender_timeout ≦ 障害検知時間 ≦ wal_sender_timeout + monitor interval
```

| 要素 | 説明 | 設定例 |
|------|------|-----|
| wal_sender_timeout | PostgreSQL がタイムアウトを検出するまでの時間 | 利用マニュアルでは 20秒（デフォルト値は 60秒） |
| monitor interval | Pacemaker が monitor 操作を実行する間隔 | 利用マニュアルでは 9秒 |

### 1.3 wal_sender_timeout を動的に変更することの意義

現状、`wal_sender_timeout` は `postgresql.conf` で固定値として設定されています。
`wal_sender_timeout` を適切な値に設定することで、**レプリケーション通信の故障が発生したときの検知時間を短縮**できます。

**理想的な動作:**
- 通常時: 小さい値で高速な障害検知
- 遅延発生時: 大きい値で誤切断を防止

この「理想的な動作」に近づけるために、`wal_sender_timeout` を動的に調整する機能を実装します。

---

## 2. 設計方針の検討

### 2.1 ping

レプリケーション通信の状態を監視する方法として、ping（ICMP）を使用することも考えられます。しかし、以下の理由から ping は採用しませんでした。

**理由 1: セキュリティ上の制約**

レプリケーション LAN は PostgreSQL 専用の通信経路であり、商用サービスにおいては ICMP を遮断している可能性があります。ping が通らない環境では監視が機能しません。

**理由 2: 実行時間の問題**

monitor 操作における ping 実行に時間がかかる可能性があり、monitor 処理全体の遅延につながります。

### 2.2 pg_stat_replication テーブルのカラム分析

PostgreSQL は `pg_stat_replication` ビューでレプリケーションの状態を公開しています。どのカラムが動的調整の入力として適切かを検討しました。

| カラム | 説明 | 動的調整への適用 |
|--------|------|-----------------|
| `state` | レプリケーション状態（streaming 等） | 状態の判定には有用だが、遅延の定量化には不向き |
| `sent_lsn` | 送信済み WAL 位置 | 遅延の定量化には LSN の差分計算が必要 |
| `write_lsn` | Standby が書き込んだ WAL 位置 | 同上 |
| `flush_lsn` | Standby が flush した WAL 位置 | 同上 |
| `replay_lsn` | Standby が適用した WAL 位置 | 同上 |
| `write_lag` | 書き込み遅延（interval 型） | 時間として直接利用可能 |
| `flush_lag` | flush 遅延（interval 型） | **時間として直接利用可能** |
| `replay_lag` | 適用遅延（interval 型） | 時間として直接利用可能 |
| `reply_time` | 最後の応答時刻 | 応答がない期間の計測には有用だが、RTT の反映には不向き |

### 2.3 flush_lag

`flush_lag` は以下の理由から、動的調整の入力として最も適切と判断しました。

#### flush_lag の定義

> Primary が WAL を送信してから、Standby がその WAL を永続ストレージに flush（fsync）するまでの時間差

#### flush_lag の測定方法

`flush_lag` は **Standby 側で測定された時刻** に基づいて計算されます。PostgreSQL の実装では以下の流れで測定されます。

```
Primary                                       Standby
   |                                             |
   | [1] WAL sent                                |
   |     WAL includes current time T_send        |
   | ------------------------------------------->|
   |                                             |
   |                            [2] WAL received |
   |                                             |
   |                            [3] WAL flush    |
   |                                   Record time T_flush
   |                                             |
   | <-------------------------------------------|
   | [4] Reply received                          |
   |     Reply includes T_flush                  |
   |                                             |
   | [5] Calculate flush_lag                     |
   |     flush_lag = T_flush - T_send            |
```

| ステップ | 場所 | 説明 |
|---------|------|------|
| [1] | Primary | WAL 送信時に現在時刻のタイムスタンプ（T_send）を WAL に含める |
| [2] | Standby | WAL を受信 |
| [3] | Standby | WAL を flush（fsync）し、その時点の時刻（T_flush）を記録 |
| [4] | Primary | Standby からの応答を受信。応答には T_flush が含まれる |
| [5] | Primary | `flush_lag = T_flush - T_send` を計算 |

なお、この測定方法において、`flush_lag` は Primary と Standby 間の時刻同期に依存します。

#### 採用理由

1. **同期レプリケーションの完了条件と一致**: 同期レプリケーション（`synchronous_commit = on`）では、Standby が WAL を flush した時点でトランザクションがコミット完了となります。`flush_lag` はこの完了までの時間を直接表します。

2. **ネットワーク遅延 + ディスク I/O 遅延を含む**: `flush_lag` は、ネットワーク片道時間だけでなく、Standby のディスク I/O 時間も含みます。これは `wal_sender_timeout` が検出すべき「応答遅延」の主要な構成要素と一致します。

3. **interval 型で取得可能**: PostgreSQL 10 以降、`flush_lag` は interval 型で直接取得でき、ミリ秒単位への変換が容易です。

### 2.4 flush_lag が NULL になるケース

前述の通り、`flush_lag` は以下の計算式で求められます。

```
flush_lag = T_flush - T_send
```

しかし、この計算には **WAL 送信が発生している必要があります**。
Standby が完全に追いついており、新しい WAL 送信がない場合、**しばらくの時間**が経過後、 `flush_lag` は NULL になります。

> If the standby server has entirely caught up with the sending server and there is no more WAL activity, the most recently measured lag times will continue to be displayed **for a short time and then show NULL.**

この「しばらくの時間」は、**`wal_receiver_status_interval` パラメータの値**で定義されています。

| パラメータ | デフォルト値 | 説明 |
|-----------|-------------|------|
| `wal_receiver_status_interval` | 10秒 | Standby が Primary に状態を報告する最小間隔 |

つまり、Standby が完全に追いついた後、約 **10秒**（デフォルト）で `flush_lag` は NULL になります。

---

## 3. 提案手法

### 3.1 計算式

`monitor_interval` 間隔で取得可能な各同期 Standby ノードの `flush_lag` の値を入力として、以下の計算式により `wal_sender_timeout` を導出します。

```
[1] 今回の flush_lag 代表値 = max(各同期 Standby ノードの flush_lag の値)
[2] 途中値 = max（今回の flush_lag 代表値, 過去 4 回の flush_lag 代表値） × 安全係数
[3] 最終値 = max（下限値, min（上限値, 途中値））
[4] 最終値をステップ値（下限値, 下限値 + 5, 下限値 + 10, ...（5 秒刻み）、上限値））に変換
[5] wal_sender_timeout = ステップ値
```

### 3.2 安全係数

安全係数は 4 としました。導出過程は以下の通りです。

#### ステップ 1: flush_lag から RTT への変換（× 2）

`flush_lag` と RTT の関係は以下の通りです。

```
flush_lag ≒ 行きのネットワーク遅延 + Standby ディスク I/O 時間
RTT       ≒ 行きのネットワーク遅延 + Standby ディスク I/O 時間 + 帰りのネットワーク遅延
```

行きのネットワーク遅延と帰りのネットワーク遅延が等しいと仮定すると、以下の関係式が成り立ちます。

```
RTT ≒ flush_lag + 帰りのネットワーク遅延 < flush_lag × 2
```

したがって、`flush_lag × 2` を RTT の上限値として設定しました。

#### ステップ 2: ARP 再解決への備え（× 2）

ネットワーク通信では、ARP（Address Resolution Protocol）キャッシュが失効した場合、ARP 要求・応答の RTT が追加で発生します。

| 状況 | 説明 |
|------|------|
| ARP キャッシュ有効時 | 通常の RTT のみ |
| ARP キャッシュ失効時 | ARP RTT + 通常の RTT |

ARP キャッシュが失効するケース：
- ネットワーク障害からの復旧時
- 長時間のアイドル状態後（Linux デフォルト: 約 30〜60 秒で stale 状態に移行）

最悪ケースを考慮し、さらに 2 倍のマージンを確保しました。

### 3.3 履歴サンプル数

検討の余地はありますが、5 としました。今回の値と、過去 4 回の履歴をサンプルとしました。

### 3.4 下限値

`wal_sender_timeout` は、Pacemaker の monitor interval より大きい値を設定する必要があります。
理由は、`wal_sender_timeout` が monitor interval より小さい場合、遅延の急増時に調整が間に合わず、Standby が切断される可能性があるためです。
したがって、以下の通りに設定しました。

```
下限値 = monitor_interval + 1 秒
```

### 3.5 上限値

PostgreSQL のデフォルト値である 60 秒を採用しました。

### 3.6 計算例（monitor_interval = 9秒、下限値 = 10秒 の場合）

| flush_lag (最大値) | 計算 | 結果 |
|-------------------|------|------|
| 100ms | 100 × 4 = 400ms | 10秒（下限） |
| 500ms | 500 × 4 = 2000ms | 10秒（下限） |
| 1000ms | 1000 × 4 = 4000ms | 10秒（下限） |
| 2500ms | 2500 × 4 = 10000ms | 10秒 |
| 3000ms | 3000 × 4 = 12000ms | 15秒 |
| 5000ms | 5000 × 4 = 20000ms | 20秒 |
| 10000ms | 10000 × 4 = 40000ms | 40秒 |
| 15000ms | 15000 × 4 = 60000ms | 60秒 |
| 20000ms | 20000 × 4 = 80000ms | 60秒（上限） |

### 3.7 flush_lag が NULL である場合の動作

flush_lag が NULL（Standby が完全に追いついている状態）の場合、履歴に 0 を追加します。
これにより、一時的なスパイクがあっても、その後通信が安定すれば徐々に下限値に戻ります。

#### 動作例（履歴サンプル数 = 5、下限値 = 10秒）

| 回数 | 履歴 | max | wal_sender_timeout |
|------|------|-----|-------------------|
| 初期 | 4, 6, 6, 4005, 6 | 4005 | 20秒 |
| NULL 1 回目 | 6, 6, 4005, 6, 0 | 4005 | 20秒 |
| NULL 2 回目 | 6, 4005, 6, 0, 0 | 4005 | 20秒 |
| NULL 3 回目 | 4005, 6, 0, 0, 0 | 4005 | 20秒 |
| NULL 4 回目 | 6, 0, 0, 0, 0 | 6 | 10秒（下限） |
| NULL 5 回目 | 0, 0, 0, 0, 0 | 0 | 10秒（下限） |

---

## 4. Phi-Accrual Failure Detector について

Phi-Accrual Failure Detector は、Cassandra や Akka で使用されている適応型障害検出アルゴリズムです。検討しましたが、以下の理由から採用しませんでした。

**理由 1: 入力データの性質の違い**

| 項目 | Phi-Accrual の想定 | 今回の実装 |
|------|-------------------|-----------|
| 入力データ | ハートビート到着間隔 | flush_lag（レプリケーション遅延） |
| 通信形態 | メッシュ型（多対多） | スター型（1 対多） |
| サンプル頻度 | 高頻度（秒単位以下） | monitor interval（9秒） |

**理由 2: サンプル数の不足**

Phi-Accrual では統計的に有意な平均・分散を計算するために、数十〜数百のサンプルが必要です。今回の実装では monitor interval が 9秒であり、十分なサンプル数を確保できません。

**理由 3: 正規分布の仮定**

Phi-Accrual はハートビート到着間隔が正規分布に従うことを仮定しています。ネットワーク遅延は一般的にロングテール分布になることが多く、この仮定が成り立たない可能性があります。

**理由 4: 実装の複雑さ**

シェルスクリプト（pgsql RA）での浮動小数点演算や確率計算は複雑であり、デバッグも困難です。

---

## 5. 仕様概要

### 5.1. 新規パラメータ

| パラメータ名 | 型 | デフォルト値 | 説明 |
|-------------|-----|-------------|------|
| adjust_wal_sender_timeout | boolean | false | flush_lag に基づく wal_sender_timeout の動的調整を有効化 |

### 5.2. 新規ファイル

| ファイルパス | 説明 |
|-------------|------|
| `$OCF_RESKEY_tmpdir/wal_sender_timeout.conf` | wal_sender_timeout の設定ファイル |
| `$OCF_RESKEY_tmpdir/flush_lag_history` | 直近 5 回の flush_lag 履歴（タイムスタンプ付き） |

### 5.3. 履歴ファイルの形式

各行は `タイムスタンプ flush_lag（ミリ秒）` の形式です。

```
2024-12-23T11:24:20 4005
2024-12-23T11:24:30 4
2024-12-23T11:30:12 6
2024-12-23T11:30:21 6
2024-12-23T11:30:30 0
```

### 5.4 動作フロー

```
[Pacemaker monitor 実行]
        ↓
[control_slave_status() 呼び出し]
        ↓
[pg_stat_replication から各同期 Standby ノードの flush_lag 取得]
        ↓
[各同期 Standby ノードの flush_lag のうち、最大値を計算]
        ↓
[flush_lag_history ファイルを更新]
        ↓
[（今回の分を含めた）過去 5 回の最大値を計算]
        ↓
[wal_sender_timeout を計算]
        ↓
[現在値と異なれば設定ファイルを更新]
        ↓
[PostgreSQL に reload 指示]
```

---

## 6. 修正箇所

### 修正1: デフォルト値の追加

**場所**: 約76行目付近（既存のデフォルト値定義の後）

```bash
OCF_RESKEY_adjust_wal_sender_timeout_default="false"
```

**場所**: 約111行目付近（既存の変数初期化の後）

```bash
: ${OCF_RESKEY_adjust_wal_sender_timeout=${OCF_RESKEY_adjust_wal_sender_timeout_default}}
```

---

### 修正2: meta_data() 関数内にパラメータ定義を追加

**場所**: `</parameters>` タグの直前（external_standby_node_list パラメータの後）

```xml
<parameter name="adjust_wal_sender_timeout" unique="0" required="0">
<longdesc lang="en">
If this is true, RA dynamically adjusts wal_sender_timeout based on
the observed flush_lag from synchronous standby nodes.
The adjustment uses the formula: max(flush_lag) * 4 (safety factor),
then rounds up to predefined steps.
The minimum value is dynamically calculated as monitor_interval + 1 second.
This helps to detect synchronous standby failures more quickly while
avoiding false positives from normal network latency variations.
This is optional for replication with sync mode.
</longdesc>
<shortdesc lang="en">adjust_wal_sender_timeout</shortdesc>
<content type="boolean" default="${OCF_RESKEY_adjust_wal_sender_timeout_default}" />
</parameter>
```

---

### 修正3: pgsql_replication_start() 関数内に初期化処理を追加

**場所**: `set_async_mode_all` の呼び出し後、`PGSQL_LOCK` チェックの前

```bash
    # Initialize wal_sender_timeout adjustment files
    if ocf_is_true ${OCF_RESKEY_adjust_wal_sender_timeout}; then
        rm -f $WAL_SENDER_TIMEOUT_CONF $FLUSH_LAG_HISTORY_FILE
        init_wal_sender_timeout_conf
    fi
```

---

### 修正4: control_slave_status() 関数の末尾に調整処理を追加

**場所**: `return 0` の直前

```bash
    # Adjust wal_sender_timeout if enabled
    if ocf_is_true ${OCF_RESKEY_adjust_wal_sender_timeout}; then
        max_flush_lag_ms=$(get_max_flush_lag_ms)
        if [ -n "$max_flush_lag_ms" ]; then
            adjust_wal_sender_timeout "$max_flush_lag_ms"
        fi
    fi
```

---

### 修正5: 新しい関数群を追加

**場所**: `control_slave_status()` 関数の後

#### get_max_flush_lag_ms()

全同期 Standby ノードから最大の flush_lag（ミリ秒）を取得する。

```bash
#
# Get maximum flush_lag in milliseconds from all sync standby nodes.
# Returns "0" if flush_lag is NULL (standby is fully caught up).
#
get_max_flush_lag_ms() {
    local output
    local flush_lag_ms
    local max_lag=0
    local rc

    output=$(exec_sql "${CHECK_FLUSH_LAG_SQL}")
    rc=$?

    if [ $rc -ne 0 ]; then
        ocf_log warn "Can't get flush_lag from pg_stat_replication."
        echo ""
        return 1
    fi

    if [ -z "$output" ]; then
        ocf_log debug "No sync standby nodes found for flush_lag calculation."
        echo "0"
        return 0
    fi

    for flush_lag_ms in $output; do
        if [ -n "$flush_lag_ms" ] && [ "$flush_lag_ms" -gt "$max_lag" ] 2>/dev/null; then
            max_lag=$flush_lag_ms
        fi
    done

    ocf_log debug "Maximum flush_lag: ${max_lag}ms"
    echo "$max_lag"
    return 0
}
```

#### calculate_wal_sender_timeout()

flush_lag から目標の wal_sender_timeout を計算する。

```bash
#
# Calculate target wal_sender_timeout from flush_lag.
# Formula: flush_lag * safety_factor (4) -> round up to step values.
# Min: monitor_interval + 1 second (dynamic), Max: 60s.
#
calculate_wal_sender_timeout() {
    local flush_lag_ms=$1
    local safety_factor=4
    local calculated_ms
    local calculated_sec
    local target_sec

    # Get monitor interval from Pacemaker environment variable (in milliseconds)
    # Default to 9000ms (9 seconds) if not set
    local monitor_interval_ms=${OCF_RESKEY_CRM_meta_interval:-9000}
    local monitor_interval_sec=$((monitor_interval_ms / 1000))
    local min_timeout=$((monitor_interval_sec + 1))

    # Calculate: flush_lag * safety_factor
    calculated_ms=$((flush_lag_ms * safety_factor))
    # Convert to seconds (round up)
    calculated_sec=$(( (calculated_ms + 999) / 1000 ))

    # Apply minimum and maximum limits, then round up to nearest 5 seconds
    if [ $calculated_sec -le $min_timeout ]; then
        target_sec=$min_timeout
    elif [ $calculated_sec -le 60 ]; then
        # Round up to nearest 5 seconds
        target_sec=$(( ((calculated_sec + 4) / 5) * 5 ))
        # Ensure at least min_timeout
        if [ $target_sec -lt $min_timeout ]; then
            target_sec=$min_timeout
        fi
    else
        target_sec=60
    fi

    echo "$target_sec"
}
```

#### get_current_wal_sender_timeout()

現在の wal_sender_timeout 値（秒）を取得する。

```bash
#
# Get current wal_sender_timeout value in seconds.
#
get_current_wal_sender_timeout() {
    local output
    local value

    # First check if we have our own conf file
    if [ -f "$WAL_SENDER_TIMEOUT_CONF" ]; then
        value=$(grep "^wal_sender_timeout" "$WAL_SENDER_TIMEOUT_CONF" | sed "s/.*=[ ]*'\?\([0-9]*\).*/\1/")
        if [ -n "$value" ]; then
            echo "$value"
            return 0
        fi
    fi

    # Query PostgreSQL for current value
    output=$(exec_sql "SHOW wal_sender_timeout;")
    if [ $? -eq 0 ] && [ -n "$output" ]; then
        # Convert to seconds (output could be like "60s" or "60000ms" or "1min")
        value=$(echo "$output" | sed 's/[^0-9]//g')
        case "$output" in
            *ms)
                value=$((value / 1000))
                ;;
            *min)
                value=$((value * 60))
                ;;
            *)
                # Assume seconds or already numeric
                ;;
        esac
        echo "$value"
        return 0
    fi

    # Default value
    echo "60"
}
```

#### init_wal_sender_timeout_conf()

wal_sender_timeout.conf を初期化する。

```bash
#
# Initialize wal_sender_timeout.conf with current or default value.
#
init_wal_sender_timeout_conf() {
    local current_timeout
    local conf_value

    # Check postgresql.conf for existing wal_sender_timeout setting
    conf_value=$(get_pgsql_param wal_sender_timeout)
    if [ -n "$conf_value" ]; then
        # Parse the value (could be "60s", "60000ms", "1min", etc.)
        current_timeout=$(echo "$conf_value" | sed 's/[^0-9]//g')
        case "$conf_value" in
            *ms)
                current_timeout=$((current_timeout / 1000))
                ;;
            *min)
                current_timeout=$((current_timeout * 60))
                ;;
            *)
                # Assume seconds
                ;;
        esac
    else
        current_timeout=60
    fi

    ocf_log info "Initializing wal_sender_timeout.conf with ${current_timeout}s"
    runasowner "echo \"wal_sender_timeout = '${current_timeout}s'\" > \"$WAL_SENDER_TIMEOUT_CONF\""
}
```

#### update_flush_lag_history()

flush_lag 履歴ファイルを更新し、平滑化された最大値を返す。

```bash
#
# Update flush_lag history file and return smoothed max value.
# History keeps last 5 flush_lag values with timestamps.
# Format: "YYYY-MM-DDTHH:MM:SS value_in_ms" per line.
#
update_flush_lag_history() {
    local new_value=$1
    local history=""
    local smoothed_max=0
    local value
    local timestamp

    # Get current timestamp
    timestamp=$(date '+%Y-%m-%dT%H:%M:%S')

    # Read existing history
    if [ -f "$FLUSH_LAG_HISTORY_FILE" ]; then
        history=$(cat "$FLUSH_LAG_HISTORY_FILE")
    fi

    # Add new value and keep only last 5
    if [ -n "$history" ]; then
        # Get last 4 lines and add new one
        history=$(echo "$history" | tail -4)
        history=$(printf "%s\n%s %s" "$history" "$timestamp" "$new_value")
    else
        history="$timestamp $new_value"
    fi

    # Write updated history
    echo "$history" > "$FLUSH_LAG_HISTORY_FILE"

    # Calculate max from history (second column)
    for value in $(echo "$history" | awk '{print $2}'); do
        if [ -n "$value" ] && [ "$value" -gt "$smoothed_max" ] 2>/dev/null; then
            smoothed_max=$value
        fi
    done

    ocf_log debug "Flush lag history updated, smoothed max: ${smoothed_max}ms"
    echo "$smoothed_max"
}
```

#### adjust_wal_sender_timeout()

flush_lag に基づいて wal_sender_timeout を調整する。

```bash
#
# Adjust wal_sender_timeout based on flush_lag.
#
adjust_wal_sender_timeout() {
    local flush_lag_ms=$1
    local smoothed_lag_ms
    local target_timeout
    local current_timeout

    # Update history and get smoothed value
    smoothed_lag_ms=$(update_flush_lag_history "$flush_lag_ms")

    # Calculate target timeout
    target_timeout=$(calculate_wal_sender_timeout "$smoothed_lag_ms")

    # Get current timeout
    current_timeout=$(get_current_wal_sender_timeout)

    ocf_log debug "wal_sender_timeout adjustment: flush_lag=${flush_lag_ms}ms, smoothed=${smoothed_lag_ms}ms, target=${target_timeout}s, current=${current_timeout}s"

    if [ "$target_timeout" -ne "$current_timeout" ]; then
        ocf_log info "Changing wal_sender_timeout: ${current_timeout}s -> ${target_timeout}s (flush_lag=${smoothed_lag_ms}ms)"
        set_wal_sender_timeout "$target_timeout"
    fi
}
```

#### set_wal_sender_timeout()

wal_sender_timeout 値を設定し、PostgreSQL をリロードする。

```bash
#
# Set wal_sender_timeout value and reload PostgreSQL.
#
set_wal_sender_timeout() {
    local timeout_sec=$1

    runasowner "echo \"wal_sender_timeout = '${timeout_sec}s'\" > \"$WAL_SENDER_TIMEOUT_CONF\""
    if [ $? -ne 0 ]; then
        ocf_log err "Failed to write wal_sender_timeout.conf"
        return 1
    fi

    exec_with_retry 0 reload_conf
    return $?
}
```

---

### 修正6: validate_ocf_check_level_10() 関数内に設定ファイルの include 追加

**場所**: REP_MODE_CONF の include directive 追加処理の後（約2270行付近）

```bash
        # Add include directive for wal_sender_timeout.conf if adjust_wal_sender_timeout is enabled
        if ocf_is_true ${OCF_RESKEY_adjust_wal_sender_timeout}; then
            WAL_SENDER_TIMEOUT_CONF=${OCF_RESKEY_tmpdir}/wal_sender_timeout.conf
            FLUSH_LAG_HISTORY_FILE=${OCF_RESKEY_tmpdir}/flush_lag_history

            wal_sender_timeout_string="include '$WAL_SENDER_TIMEOUT_CONF' # added by pgsql RA"
            if ! grep -q "^[[:space:]]*$wal_sender_timeout_string" $OCF_RESKEY_config; then
                ocf_log info "adding include directive $wal_sender_timeout_string into $OCF_RESKEY_config"
                echo "$wal_sender_timeout_string" >> $OCF_RESKEY_config
            fi

            # Define SQL for getting flush_lag
            CHECK_FLUSH_LAG_SQL="SELECT COALESCE(EXTRACT(EPOCH FROM flush_lag) * 1000, 0)::bigint FROM pg_stat_replication WHERE sync_state = 'sync'"
        fi
```

---

### 修正7: グローバル変数の初期化

**場所**: validate_ocf_check_level_10() 関数内、is_replication ブロック内（CHECK_REPLICATION_STATE_SQL 定義の後、約2220行付近）

```bash
        # Variables for wal_sender_timeout adjustment
        WAL_SENDER_TIMEOUT_CONF=""
        FLUSH_LAG_HISTORY_FILE=""
        CHECK_FLUSH_LAG_SQL=""
```

---

## 7. 動作確認

### パラメータ設定

リソース定義に以下を追加:

```xml
<nvpair id="pgsql-instance_attributes-adjust_wal_sender_timeout" name="adjust_wal_sender_timeout" value="true"/>
```

### ログ確認

```bash
grep -E "(wal_sender_timeout|flush_lag)" /var/log/messages | tail -50
```

ログ出力例

```
Dec 23 11:10:47 primary1 pgsql(pgsql)[61071]: INFO: Initializing wal_sender_timeout.conf with 20s
Dec 23 11:14:22 primary1 pgsql(pgsql)[71499]: INFO: Changing wal_sender_timeout: 20s -> 10s (flush_lag=6ms)
Dec 23 11:24:20 primary1 pgsql(pgsql)[93687]: INFO: Changing wal_sender_timeout: 10s -> 20s (flush_lag=4006ms)
Dec 23 11:24:30 primary1 pgsql(pgsql)[94110]: INFO: Changing wal_sender_timeout: 20s -> 40s (flush_lag=8839ms)
Dec 23 11:30:12 primary1 pgsql(pgsql)[106995]: INFO: Changing wal_sender_timeout: 40s -> 20s (flush_lag=4005ms)
```

### ファイル確認

```bash
cat /var/lib/pgsql/tmp/wal_sender_timeout.conf
cat /var/lib/pgsql/tmp/flush_lag_history
```

履歴ファイルの出力例：
```bash
cat /var/lib/pgsql/tmp/flush_lag_history
2024-12-23T11:24:20 4005
2024-12-23T11:24:30 4
2024-12-23T11:30:12 6
2024-12-23T11:30:21 6
2024-12-23T11:30:30 0
```

### その他の有益なコマンド

```bash
# tc コマンドによる遅延（例では 3000ms）の挿入
tc qdisc add dev eth2 root netem delay 3000ms

# tc コマンドの設定確認
tc qdisc show dev eth2
qdisc netem 8003: root refcnt 2 limit 1000 delay 3s

# tc コマンドによる遅延の削除
tc qdisc del dev eth2 root

# 現在の wal_sender_timeout の値を確認
sudo -u postgres psql -c "SHOW wal_sender_timeout;"

# テスト用テーブル作成とデータ挿入
sudo -u postgres psql -c "CREATE TABLE IF NOT EXISTS test_lag (id serial, data text, created_at timestamp default now());"

# データを挿入して WAL を生成
sudo -u postgres psql -c "INSERT INTO test_lag (data) SELECT md5(random()::text) FROM generate_series(1, 1000);"

# pg_stat_replication テーブルを全確認
sudo -u postgres psql -c "SELECT * FROM pg_stat_replication;"

# flush_lag_history を確認
cat /var/lib/pgsql/tmp/flush_lag_history
```

---

## 8. 今後の課題

今後、検討が必要な項目は以下の通りです。

1. **pgsql の monitor interval の最適化**: 現在は 9 秒で固定としています。環境によっては短くできる可能性があります。

2. **履歴サンプル数の最適化**: 現在は 5 回としています。環境によっては調整が必要になります。

---

## 9. 参考文献

- レプリケーション遅延の監視について（第 40 回PostgreSQLアンカンファレンス@オンライン 発表資料）
  - https://www.slideshare.net/slideshow/postgresql-replication-lags-pgunconf40-nttdata/256567436

- PostgreSQL Documentation: pg_stat_replication
  - https://www.postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-REPLICATION

- PostgreSQL Source Code: walsender.c

- Database Internals, Chapter 9: Failure Detection
  - Phi-Accrual Failure Detector の解説
