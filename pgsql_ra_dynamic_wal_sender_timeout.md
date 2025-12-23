# pgsql RA 修正: 動的 wal_sender_timeout 調整機能

## 概要

Pacemaker pgsql Resource Agent に、`flush_lag` に基づく `wal_sender_timeout` の動的調整機能を追加します。この機能により、同期 Standby ノードの障害をより迅速に検出しつつ、ネットワーク遅延の変動による誤検出を防止します。

---

## 仕様概要

### 新規パラメータ

| パラメータ名 | 型 | デフォルト値 | 説明 |
|-------------|-----|-------------|------|
| adjust_wal_sender_timeout | boolean | false | flush_lag に基づく wal_sender_timeout の動的調整を有効化 |

### 新規ファイル

| ファイルパス | 説明 |
|-------------|------|
| `$OCF_RESKEY_tmpdir/wal_sender_timeout.conf` | wal_sender_timeout の設定ファイル |
| `$OCF_RESKEY_tmpdir/flush_lag_history` | 過去 5 回の flush_lag 履歴（タイムスタンプ付き） |

### 履歴ファイルの形式

```
2024-12-23T11:24:20 4005
2024-12-23T11:24:30 4
2024-12-23T11:30:12 6
2024-12-23T11:30:21 6
2024-12-23T11:30:30 0
```

各行は `タイムスタンプ flush_lag値（ミリ秒）` の形式です。

### 計算式

```
target = max(過去5回のflush_lag) × 4
下限値 = monitor_interval + 1秒
最終値 = max(下限値, min(60秒, target))
```

### パラメータ値

| 項目 | 値 | 根拠 |
|------|-----|------|
| 安全係数 | 4倍 | RFC 6298 (TCP RTO) の K 値 |
| 下限値 | monitor_interval + 1秒 | monitor interval より大きい値を動的に計算 |
| 上限値 | 60秒 | PostgreSQL デフォルト値 |
| 履歴サンプル数 | 5回 | 平滑化のため |
| ステップ値 | 下限値, 下限値+5, 下限値+10, ...（5秒刻み、最大60秒） | 動的に計算 |

### 計算例（monitor_interval = 9秒、下限値 = 10秒 の場合）

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

### 計算例（monitor_interval = 19秒、下限値 = 20秒 の場合）

| flush_lag (最大値) | 計算 | 結果 |
|-------------------|------|------|
| 100ms | 100 × 4 = 400ms | 20秒（下限） |
| 2500ms | 2500 × 4 = 10000ms | 20秒（下限） |
| 5000ms | 5000 × 4 = 20000ms | 20秒 |
| 6000ms | 6000 × 4 = 24000ms | 25秒 |
| 10000ms | 10000 × 4 = 40000ms | 40秒 |

### 動作仕様

| 項目 | 仕様 |
|------|------|
| 平滑化 | 過去 5 回の flush_lag の最大値 |
| 値を上げる場合 | 即座に反映 |
| 値を下げる場合 | 即座に反映 |
| flush_lag が NULL の場合 | 履歴に 0 を追加し、徐々に下限値へ近づける |
| 対象 | クラスタ内 + external_standby_node_list の全同期 Standby |

### flush_lag が NULL の場合の動作

flush_lag が NULL（Standby が完全に追いついている状態）の場合、履歴に 0 を追加します。これにより、一時的なスパイクがあっても、その後通信が安定すれば徐々に下限値に戻ります。

動作例（履歴サンプル数 = 5、下限値 = 10秒）：

| 回数 | 履歴 | max | wal_sender_timeout |
|------|------|-----|-------------------|
| 初期 | 4, 6, 6, 4005, 6 | 4005 | 20秒 |
| NULL 1回目 | 6, 6, 4005, 6, 0 | 4005 | 20秒 |
| NULL 2回目 | 6, 4005, 6, 0, 0 | 4005 | 20秒 |
| NULL 3回目 | 4005, 6, 0, 0, 0 | 4005 | 20秒 |
| NULL 4回目 | 6, 0, 0, 0, 0 | 6 | 10秒（下限） |
| NULL 5回目 | 0, 0, 0, 0, 0 | 0 | 10秒（下限） |

---

## 設計根拠

### 安全係数 = 4 の根拠

RFC 6298 で定義された TCP の再送タイムアウト（RTO）計算式において、安全係数 K = 4 が採用されています。この値は長年の TCP 運用実績に基づいており、ネットワーク遅延の変動を考慮した標準的な値です。

### 下限値 = monitor_interval + 1秒 の根拠

Pacemaker の monitor interval より大きい値を設定する必要があります。`wal_sender_timeout` が monitor interval より小さい場合、遅延の急増時に調整が間に合わず、Standby が切断される可能性があります。

`OCF_RESKEY_CRM_meta_interval` 環境変数を使用することで、Pacemaker が設定した monitor interval を動的に取得し、適切な下限値を自動計算します。

### 履歴サンプル数 = 5 の根拠

短期的なスパイクによる過剰反応を防ぎつつ、遅延の傾向を適切に捉えるためのバランスを考慮しました。

### flush_lag が NULL の場合に 0 を追加する根拠

flush_lag が NULL の状態が続く場合、レプリケーションが快適であることを意味します。履歴に 0 を追加することで、過去のスパイクを徐々に履歴から押し出し、下限値に近づけます。これにより、一時的な遅延増加後も適切に回復できます。

---

## 修正箇所

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
The adjustment uses the formula: max(flush_lag) * 4 (safety factor from RFC 6298),
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
# Formula: flush_lag * safety_factor (4, from RFC 6298) -> round up to step values.
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

## 動作確認

### パラメータ設定

リソース定義に以下を追加:

```xml
<nvpair id="pgsql-instance_attributes-adjust_wal_sender_timeout" name="adjust_wal_sender_timeout" value="true"/>
```

または pcs コマンドで設定: → これを実行すると、pgsql monitor error になるため、使用しないこと

```bash
pcs resource update pgsql adjust_wal_sender_timeout=true
```

### ログ確認

```bash
grep -E "(wal_sender_timeout|flush_lag)" /var/log/messages | tail -50
```

期待されるログ:
- `Initializing wal_sender_timeout.conf with XXs` - 初期化時
- `wal_sender_timeout adjustment:` - 毎回の調整計算（debug レベル）
- `Changing wal_sender_timeout:` - 値を変更する時

### ファイル確認

```bash
cat /var/lib/pgsql/tmp/wal_sender_timeout.conf
cat /var/lib/pgsql/tmp/flush_lag_history
```

履歴ファイルの出力例：
```
2024-12-23T11:24:20 4005
2024-12-23T11:24:30 4
2024-12-23T11:30:12 6
2024-12-23T11:30:21 6
2024-12-23T11:30:30 0
```

### PostgreSQL 設定確認

```bash
sudo -u postgres psql -c "SHOW wal_sender_timeout;"
```

### その他の有益なコマンド

```bash
# tc コマンドによる遅延（例では 2000ms）の挿入
tc qdisc add dev eth2 root netem delay 2000ms

# tc コマンドの設定確認
tc qdisc show dev eth2
qdisc netem 8003: root refcnt 2 limit 1000 delay 2s

# tc コマンドによる遅延の削除
tc qdisc del dev eth2 root

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

## 参考文献

- レプリケーション遅延の監視について（第40回PostgreSQLアンカンファレンス@オンライン 発表資料）
  - https://www.slideshare.net/slideshow/postgresql-replication-lags-pgunconf40-nttdata/256567436

- RFC 6298: Computing TCP's Retransmission Timer
  - https://www.rfc-editor.org/rfc/rfc6298
  - 安全係数 K = 4 の根拠

- PostgreSQL Documentation: pg_stat_replication
  - https://www.postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-REPLICATION

- PostgreSQL Source Code: walsender.c
  - wal_sender_timeout の動作詳細
