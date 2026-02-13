# Troubleshooting: GCP asia-south1-standard Not Emitting Metrics to Yamas

**Date:** 2026-02-13  
**Host:** udp-asia-south1-standard (GCP)  
**Symptom:** No metrics for this host in Yamas (YFlow.udping_gcp.count with to_host=udp-asia-south1-standard).  
**Context:** Previously required OTEL collector restart due to queue backlog.

---

## 1. Issue Summary

The udp-asia-south1-standard GCP host was not appearing in Yamas metrics. Queries for `to_host=udp-asia-south1-standard.*` returned **0** from_hosts for the last 10 minutes. Root cause was OTEL collector exporter backlog and high send failures (same pattern as prior queue-backlog incident).

---

## 2. Commands Executed (Troubleshooting)

Reference: **docs/udping ADVANCED_DEBUG_CHEATSHEET.md** (Quick Health Check, OTEL Collector Debugging, Yamas Horizon CLI).

### 2.1 GCP authentication (prerequisite)

```bash
gcpfed -domain pcp.monitor.network -role gcp.fed.admin.user -gcp-svc-name fed-admin-user
```

### 2.2 Host health and port 8127 traffic

```bash
gssh udp-asia-south1-standard -- bash << 'EOF'
echo "=== 1. udping_launcher status ==="
sudo systemctl status udping_launcher | head -8

echo "=== 2. udping_server process ==="
ps aux | grep "udping_server.*33435" | grep -v grep

echo "=== 3. Port 8127 traffic (5s) ==="
sudo timeout 5 tcpdump -i lo -c 10 -A udp port 8127 2>&1 | grep "count:" | head -3 || echo "(no count lines in 5s)"

echo "=== 4. Collector listening 8127/4317 ==="
sudo ss -nlup | grep -E "8127|4317"
EOF
```

**Findings:** Launcher and udping_server running; collector listening on 8127; no "count:" lines in 5s tcpdump on 8127 (udping_server on this host uses `-t localhost:8125`). Receiver had historically accepted large volume (see below).

### 2.3 OTEL collector status, queue, and exporter metrics

```bash
gssh udp-asia-south1-standard -- bash << 'EOF'
echo "=== 5. OTEL collector status ==="
sudo systemctl status netauto-otelcol_launcher | head -10

echo "=== 6. Queue / exporter (yamashttp) ==="
curl -s localhost:8888/metrics | grep -E "yamashttp" | grep -E "queue_size|sent|failed"

echo "=== 7. Receiver accepted (statsd 8127) ==="
curl -s localhost:8888/metrics | grep "receiver_accepted.*statsd"

echo "=== 8. otelcol uptime ==="
ps -o pid,etime,args -p $(pgrep -f "otelcol --config" | head -1)

echo "=== 9. udping_server stats target ==="
ps aux | grep udping_server | grep -v grep
EOF
```

**Findings:**

| Metric | Value |
|--------|--------|
| otelcol_exporter_queue_size (yamashttp) | **649** (expected 0–10; cheatsheet: restart if high) |
| otelcol_exporter_send_failed_metric_points_total | **~233M** |
| otelcol_exporter_sent_metric_points_total | ~723M |
| otelcol_receiver_accepted_metric_points_total (statsd) | ~960M |
| otelcol uptime | ~57 days |

Conclusion: Exporter to Yamas was failing; queue backed up. Restart collector to clear backlog.

### 2.4 Yamas query (confirm no metrics for host)

```bash
NOW=$(date +%s); TEN_MIN_AGO=$((NOW - 600))
cat > /tmp/yamas_asia_south1.json << EOF
{
  "start": ${TEN_MIN_AGO}000,
  "end": ${NOW}000,
  "queries": [{
    "aggregator": "zimsum",
    "metric": "YFlow.udping_gcp.count",
    "rate": false,
    "downsample": "1m-avg-nan",
    "filters": [
      {"type": "regexp", "tagk": "to_host", "filter": "udp-asia-south1-standard.*", "groupBy": false},
      {"type": "regexp", "tagk": "from_host", "filter": ".*", "groupBy": true}
    ]
  }]
}
EOF
yamas horizon metrics query -q /tmp/yamas_asia_south1.json 2>&1 | tr ',' '\n' | grep "from_host" | sort -u | wc -l
```

**Result:** **0** from_hosts → no metrics for udp-asia-south1-standard in Yamas for the last 10 minutes.

---

## 3. Fix Applied

Restart the OTEL collector on the host to clear the exporter queue and re-establish healthy export to Yamas:

```bash
gssh udp-asia-south1-standard -- 'sudo systemctl restart netauto-otelcol_launcher'
```

---

## 4. Fix Verification

### 4.1 Collector status and queue after restart

```bash
gssh udp-asia-south1-standard -- 'sudo systemctl status netauto-otelcol_launcher | head -6'
gssh udp-asia-south1-standard -- 'curl -s localhost:8888/metrics | grep -E "yamashttp" | grep -E "queue_size|sent|failed"'
```

**Result:**

| Metric | Before restart | After restart |
|--------|----------------|----------------|
| queue_size | 649 | **0** |
| send_failed_metric_points_total | ~233M | **0** |
| sent_metric_points_total | ~723M | 11,622 (increasing) |
| Service | active, uptime ~57 days | active (running) since Fri 2026-02-13 17:27:56 UTC |

### 4.2 Yamas query after ~1 minute

Same query as in 2.4 (last 10 min, to_host=udp-asia-south1-standard.*):

```bash
yamas horizon metrics query -q /tmp/yamas_asia_south1.json 2>&1 | tr ',' '\n' | grep "from_host" | sort -u | wc -l
yamas horizon metrics query -q /tmp/yamas_asia_south1.json 2>&1 | tr ',' '\n' | grep "from_host" | sort -u | head -10
```

**Result:** **100** from_hosts; sample from_hosts include ec2-*, tftp1.ops.ams.yahoo.com, etc. Metrics for udp-asia-south1-standard are present in Yamas.

### 4.3 Final queue check

```bash
gssh udp-asia-south1-standard -- 'curl -s localhost:8888/metrics | grep -E "yamashttp" | grep -E "queue_size|sent|failed"'
```

**Result:** queue_size=0, send_failed=0, sent growing (11,622+).

---

## 5. Conclusion

- **Root cause:** OTEL collector exporter backlog and high send failures to Yamas (queue_size 649, ~233M failed points), consistent with a previous queue-backlog incident.
- **Fix:** Restart `netauto-otelcol_launcher` on udp-asia-south1-standard.
- **Verification:** Queue cleared (0), no new failures (0), sent count increasing; Yamas shows 100 from_hosts for to_host=udp-asia-south1-standard in the last 10 minutes.

**Cheatsheet reference:** ADVANCED_DEBUG_CHEATSHEET.md — “Diagnose queue backup (export stalled)”, “Restart collector (fixes stale connections/memory issues)”, “Verify metrics flowing end-to-end”, “Yamas Horizon CLI”.
