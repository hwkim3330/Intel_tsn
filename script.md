
# Intel i210/i225v TSN TAS 실험 스크립트 모음

이 문서는 Intel i210 및 i225v NIC에서 TAS(Time‑Aware Shaper)를 적용하고  
Payload‑timestamp 방식으로 지연을 측정·분석하기 위한 전체 스크립트를 제공합니다.  
아래 파일 하나(.md)만 깃허브에 올리시면 됩니다.

---

## 1. `setup_namespaces.sh`

네임스페이스 생성, NIC 이동 및 IP 설정을 자동화합니다.

```bash
#!/usr/bin/env bash
# usage: bash setup_namespaces.sh

set -e

# 1) 기존 네임스페이스 삭제
sudo ip netns del ns1 2>/dev/null || true
sudo ip netns del ns2 2>/dev/null || true

# 2) 네임스페이스 생성
sudo ip netns add ns1
sudo ip netns add ns2

# 3) 물리 NIC down → 네임스페이스로 이동
sudo ip link set dev enp7s0 down
sudo ip link set dev enp9s0 down
sudo ip link set dev enp7s0 netns ns1
sudo ip link set dev enp9s0 netns ns2

# 4) 송신기(ns1) 설정
sudo ip netns exec ns1 ip link set lo up
sudo ip netns exec ns1 ip link set dev enp7s0 up
sudo ip netns exec ns1 ip addr add 169.254.59.170/16 dev enp7s0

# 5) 수신기(ns2) 설정
sudo ip netns exec ns2 ip link set lo up
sudo ip netns exec ns2 ip link set dev enp9s0 up
sudo ip netns exec ns2 ip addr add 169.254.59.100/16 dev enp9s0

echo "✔ 네임스페이스 및 NIC 설정 완료"
````

---

## 2. `toggle_tas.sh`

10 µs ↔ 2 µs TAS 스케줄을 지정한 횟수만큼 반복합니다.

```bash
#!/usr/bin/env bash
# usage: sudo bash toggle_tas.sh <iterations> <hold_sec>

if [ $# -ne 2 ]; then
  echo "Usage: $0 <iterations> <hold_sec>"
  exit 1
fi

ITER=$1
HOLD=$2
DEV=enp7s0

for ((i=1;i<=ITER;i++)); do
  for C in 10 2; do
    if [ "$C" -eq 10 ]; then
      OPEN=5000; CLOSE=5000
    else
      OPEN=1000; CLOSE=1000
    fi
    BT=$(( $(date +%s%N) + 3000000000 ))
    echo "[$(date '+%H:%M:%S')] Iter $i – TAS ${C}µs (open=${OPEN}ns)"
    sudo ip netns exec ns1 tc qdisc replace dev $DEV root handle 100: taprio \
      clockid CLOCK_TAI \
      num_tc 1 \
      map 0 0 0 0 0 0 0 0 \
      queues 1@0 \
      base-time $BT \
      sched-entry S 01 $OPEN \
      sched-entry S 00 $CLOSE \
      flags 0x1
    echo " → hold ${HOLD}s"
    sleep $HOLD
  done
done

echo "✔ toggle_tas.sh 완료"
```

---

## 3. `sender.py`

UDP 페이로드에 송신 타임스탬프를 기록하여 보냅니다.

```python
#!/usr/bin/env python3
# sender.py <interval_us> <count> <dst_ip>

import socket, struct, time, sys

if len(sys.argv) != 4:
    print("Usage: sender.py <interval_us> <count> <dst_ip>")
    sys.exit(1)

interval_us = int(sys.argv[1])
count       = int(sys.argv[2])
dst_ip      = sys.argv[3]
port        = 54321

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.bind(("0.0.0.0", port))

for i in range(count):
    tx = time.time_ns()
    pkt = struct.pack("!Q", tx)
    sock.sendto(pkt, (dst_ip, port))
    time.sleep(interval_us / 1_000_000)
```

---

## 4. `receiver.py`

수신한 패킷의 송신 타임스탬프를 읽고 실제 지연을 CSV로 기록합니다.

```python
#!/usr/bin/env python3
# receiver.py <outfile.csv>

import socket, struct, time, csv, sys

if len(sys.argv) != 2:
    print("Usage: receiver.py <outfile.csv>")
    sys.exit(1)

outfile = sys.argv[1]
port    = 54321

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.bind(("0.0.0.0", port))

with open(outfile, "w", newline="") as f:
    w = csv.writer(f)
    w.writerow(["tx_time_ns", "latency_ns"])
    while True:
        data, addr = sock.recvfrom(8)
        rx = time.time_ns()
        tx, = struct.unpack("!Q", data)
        w.writerow([tx, rx - tx])
```

---

## 5. `plot_latency.py`

`latencies.csv`를 읽어 요약 통계, 히스토그램, CDF를 생성·저장합니다.

```python
#!/usr/bin/env python3
# plot_latency.py <latencies.csv>

import pandas as pd
import matplotlib.pyplot as plt
import numpy as np
import os
from tabulate import tabulate

# 입력 파일
csv_path = "latencies.csv"
df = pd.read_csv(csv_path)

# 요약 통계 테이블
summary = []
for cycle_us in [2, 10]:
    cycle_ns = cycle_us * 1000
    sub = df[df["tx_time_ns"] % cycle_ns < cycle_ns]
    desc = sub["latency_ns"].describe(percentiles=[.01, .05, .5, .95, .99])
    summary.append({
        "Cycle (µs)": cycle_us,
        "Count": int(desc["count"]),
        "Mean (ns)": f"{desc['mean']:.1f}",
        "Std (ns)": f"{desc['std']:.1f}",
        "Min": int(desc["min"]),
        "1%":  int(desc["1%"]),
        "5%":  int(desc["5%"]),
        "50%": int(desc["50%"]),
        "95%": int(desc["95%"]),
        "99%": int(desc["99%"]),
        "Max": int(desc["max"]),
    })

print("\n## 요약 통계")
print(tabulate(summary, headers="keys", tablefmt="github"))

# 히스토그램
hist_path = os.path.expanduser("~/latency_histogram.png")
plt.figure(figsize=(6,4))
bins = np.linspace(0, df["latency_ns"].max(), 200)
for cycle_us in [2, 10]:
    cycle_ns = cycle_us * 1000
    sub = df[df["tx_time_ns"] % cycle_ns < cycle_ns]
    plt.hist(sub["latency_ns"], bins=bins, histtype="step", label=f"{cycle_us}µs")
plt.title("Latency Histogram")
plt.xlabel("latency (ns)")
plt.ylabel("count")
plt.legend()
plt.grid(True)
plt.tight_layout()
plt.savefig(hist_path)
plt.close()

# CDF
cdf_path = os.path.expanduser("~/latency_cdf.png")
plt.figure(figsize=(6,4))
for cycle_us in [2, 10]:
    cycle_ns = cycle_us * 1000
    sub = df[df["tx_time_ns"] % cycle_ns < cycle_ns]
    data = np.sort(sub["latency_ns"])
    cdf = np.arange(1, len(data)+1) / len(data)
    plt.plot(data, cdf, label=f"{cycle_us}µs")
plt.title("Latency CDF")
plt.xlabel("latency (ns)")
plt.ylabel("CDF")
plt.legend()
plt.grid(True)
plt.tight_layout()
plt.savefig(cdf_path)
plt.close()

print(f"\n✔ 저장된 히스토그램: {hist_path}")
print(f"✔ 저장된 CDF:        {cdf_path}")
```

---

## 사용 가이드

1. **설치 & 의존성**

   ```bash
   sudo apt update
   sudo apt install -y iproute2 ethtool tc hping3 tcpdump python3-pip
   pip3 install pandas matplotlib numpy tabulate
   ```

2. **네임스페이스 & NIC 설정**

   ```bash
   bash setup_namespaces.sh
   ```

3. **수신기 실행 (ns2)**

   ```bash
   sudo ip netns exec ns2 python3 receiver.py latencies.csv
   ```

4. **송신기 실행 (ns1)**

   ```bash
   sudo ip netns exec ns1 python3 sender.py 10 20000 169.254.59.100
   ```

5. **TAS 토글 (호스트)**

   ```bash
   sudo bash toggle_tas.sh 3 10
   ```

6. **분석 & 그래프 저장**

   ```bash
   python3 plot_latency.py latencies.csv
   ```

---

**끝**

```
::contentReference[oaicite:0]{index=0}
```
