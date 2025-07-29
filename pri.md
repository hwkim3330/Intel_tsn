# Prio vs. TAS 실험 보고서

**작성자**: hwkim3330  
**작성일**: 2025-07-29  

---

## 1. 서론

본 실험은 소프트웨어 우선순위(IEEE 802.1p Prio)와 하드웨어 기반 Time‑Aware Shaper(TAS, IEEE 802.1Qbv)를 동일 조건(10 µs 슬롯, 100 000개 패킷, 1 Gbps UDP 포화) 하에서 비교하여, 각각의 장·단점과 실제 환경 적용 가능성을 평가하기 위해 수행되었다.  

---

## 2. 주요 결과 요약

| Mode      | Count   | Mean (µs) | P50 (µs) | P95 (µs) | P99 (µs) | Max (µs) |
|:----------|--------:|----------:|---------:|---------:|---------:|---------:|
| **Prio**     | 100 000 |    **7.98**   |   **6.80**  |  10.92   |  15.68   |  83.60   |
| **TAS**      | 100 000 |   194.22   | 195.35  | 263.37  | 271.17  | 353.34  |

> ※ 단위 변환: 1 µs = 1 000 ns

---

## 3. 긍정적 측면

### 3.1 Prio 우선순위
- **극소 지연**: 평균 7.98 µs, P95 10.9 µs로 TAS(≈194 µs)에 비해 훨씬 낮은 지연  
- **간단한 설정**: `tc prio + tc filter` 만으로 구현 가능 → 빠른 PoC에 적합  
- **낮은 오버헤드**: 소프트웨어 qdisc 수준에서 처리되어 하드웨어 드라이버 수정 불필요  

### 3.2 TAS(Time‑Aware Shaper)
- **일관된 패턴**: TAS 스케줄(Open/Close) 구조를 통해 무부하 환경에서 tight한 지연 분포  
- **멀티홉 결정론**: 네트워크 전 구간에서 동일 GCL 적용 시 end‑to‑end 최대 지연 예측 가능  
- **하드웨어 오프로드**: NIC 차원에서 스케줄링 처리 → 나노초 단위 정밀도 확보  

---

## 4. 부정적 측면

### 4.1 Prio 우선순위
- **하위 클래스 지연 폭발**: BE 트래픽(우선순위 낮은 패킷)은 장시간 큐잉  
- **멀티홉 예측 불가**: 각 홉의 큐 상태에 따라 지연 변동성이 커, end‑to‑end 제어 한계  

### 4.2 TAS(Time‑Aware Shaper)
- **설정 복잡성**: GCL 설계·동기화(PTP) 필요 → 초기 구축 비용·운영 난이도 상승  
- **슬롯 대기 과금**: Close(비송신) 구간이 평균 지연을 높임 → 가벼운 실시간 트래픽엔 오버헤드  
- **드라이버 지원 제한**: 일부 NIC(i210 계열)만 하드웨어 off‑load 지원, flags 설정 이슈  

---

## 5. 미래 전망 및 제언

1. **하이브리드 접근**  
   - **Prio + TAS** 결합: 실시간 소수 트래픽은 Prio로 빠르게 통과, 초정밀 구간에 한해 TAS 적용  
   - **Adaptive GCL**: 트래픽 부하·우선순위에 따라 동적으로 슬롯 길이 조정  

2. **멀티홉 TSN 네트워크**  
   - PTP 기반 시간 동기화와 TAS 스케줄 분산 적용으로 **End‑to‑End 결정론** 실현  
   - 산업용 이더넷, 자율주행, 항공 임무 등 고신뢰성 시스템에 확대  

3. **자동화 도구 개발**  
   - GCL 생성·검증·배포 자동화 툴 체인  
   - 실시간 모니터링·알람 시스템 연계  

4. **비교 실험 확대**  
   - **다양한 부하 프로파일**: TCP 혼합, 멀티스트림 UDP, 실제 애플리케이션 트래픽  
   - **QoS 기능**: CBS, ATS, BBR 등과 함께 복합 성능 분석  

---

## 6. 결론

- **Prio(802.1p)** 는 낮은 구현 복잡도와 우수한 평균 지연을 제공하지만, end‑to‑end 결정론 보장에는 한계가 있다.  
- **TAS(802.1Qbv)** 는 하드웨어 기반 슬롯 스케줄링으로 일관된 지연 바운드를 제공하며, 멀티홉 TSN 환경에 적합하다.  
- 실제 배포 환경에서는 **서비스 요구사항**(지연 한계, 네트워크 규모, 운영 난이도)을 고려해 두 방식을 혼합하거나, **Adaptive TSN** 구현을 검토하는 것이 바람직하다.

---

**참고**  
- 실험 스크립트: `sender.py` / `receiver.py` / `setup_namespaces.sh` / `toggle_tas.sh` / `plot_latency_compare.py`  
- 데이터 및 그림: `latencies_prio.csv`, `latencies_tas.csv`, `latency_load_compare.png`  


# TSN 실험 스크립트 모음

이 문서는 **Prio vs TAS** 및 **Payload‑TS** 지연 측정, **토글 테스트**, **결과 분석**을 위한 전체 스크립트를 정리한 것입니다.  
각 파일을 복사해서 `.sh` 또는 `.py` 확장자로 저장한 뒤, 실행 권한을 부여하고 사용하세요.

---

## 1. 네임스페이스 & veth 설정 (`setup_namespaces.sh`)

```bash
#!/usr/bin/env bash
set -e

# 1) 기존 네임스페이스 삭제
sudo ip netns del ns1 2>/dev/null || true
sudo ip netns del ns2 2>/dev/null || true

# 2) 네임스페이스 생성
sudo ip netns add ns1
sudo ip netns add ns2

# 3) veth 페어 생성 및 이동
sudo ip link add veth0 type veth peer name veth1
sudo ip link set veth0 netns ns1
sudo ip link set veth1 netns ns2

# 4) 네임스페이스별 인터페이스 설정
sudo ip netns exec ns1 ip link set lo up
sudo ip netns exec ns1 ip link set dev veth0 up
sudo ip netns exec ns1 ip addr add 10.0.0.1/24 dev veth0

sudo ip netns exec ns2 ip link set lo up
sudo ip netns exec ns2 ip link set dev veth1 up
sudo ip netns exec ns2 ip addr add 10.0.0.2/24 dev veth1

echo "✔ veth 네임스페이스 준비 완료 (10.0.0.1 ↔ 10.0.0.2)"
````

사용:

```bash
chmod +x setup_namespaces.sh
bash setup_namespaces.sh
```

---

## 2. Prio‑qdisc 설정 (`setup_prio.sh`)

```bash
#!/usr/bin/env bash
set -e

# Prio 스케줄: 8 bands, ip dst=10.0.0.2 → band0
sudo ip netns exec ns1 tc qdisc replace dev veth0 root \
  handle 1: prio bands 8 priomap 0 1 2 3 4 5 6 7

sudo ip netns exec ns1 tc filter add dev veth0 protocol ip \
  parent 1: prio 1 u32 match ip dst 10.0.0.2/32 \
  action skbedit priority 0

echo "✔ Prio‑qdisc(802.1p 모의) 설정 완료"
```

사용:

```bash
chmod +x setup_prio.sh
bash setup_prio.sh
```

---

## 3. TAS‑qdisc 설정 (`setup_tas.sh`)

```bash
#!/usr/bin/env bash
set -e

# 10µs cycle, TXTIME‑assist (flags 0x1)
BT=$(( $(date +%s%N) + 3000000000 ))

sudo ip netns exec ns1 tc qdisc replace dev veth0 root handle 100: taprio \
  clockid CLOCK_TAI num_tc 1 map 0 0 0 0 0 0 0 0 queues 1@1 \
  base-time $BT \
  sched-entry S 1 5000 \
  sched-entry S 0 5000 \
  flags 0x1

echo "✔ TAS qdisc (10µs cycle) 설정 완료"
```

사용:

```bash
chmod +x setup_tas.sh
bash setup_tas.sh
```

---

## 4. Sender 스크립트 (`sender.py`)

```python
#!/usr/bin/env python3
import socket, struct, time, sys

# usage: sender.py <interval_us> <count> <dst_ip>
interval_us = int(sys.argv[1])
count       = int(sys.argv[2])
dst_ip      = sys.argv[3]
port        = 54321

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.bind(("0.0.0.0", port))

for _ in range(count):
    tx = time.time_ns()
    sock.sendto(struct.pack("!Q", tx), (dst_ip, port))
    time.sleep(interval_us / 1_000_000)
```

사용:

```bash
chmod +x sender.py
sudo ip netns exec ns1 python3 sender.py 10 100000 10.0.0.2
```

---

## 5. Receiver 스크립트 (`receiver.py`)

```python
#!/usr/bin/env python3
import socket, struct, time, csv, sys

# usage: receiver.py <outfile.csv>
outfile = sys.argv[1]
port    = 54321

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.bind(("0.0.0.0", port))

with open(outfile, "w", newline="") as f:
    w = csv.writer(f)
    w.writerow(["tx_time_ns", "latency_ns"])
    while True:
        data, _ = sock.recvfrom(8)
        rx = time.time_ns()
        tx, = struct.unpack("!Q", data)
        w.writerow([tx, rx - tx])
```

사용:

```bash
chmod +x receiver.py
sudo ip netns exec ns2 python3 receiver.py latencies_prio.csv
```

---

## 6. 비교·분석 스크립트 (`plot_latency_compare.py`)

```python
#!/usr/bin/env python3
import pandas as pd, numpy as np
import matplotlib.pyplot as plt
from tabulate import tabulate
import sys, os

# usage: plot_latency_compare.py <csv1> <csv2>
csv1, csv2 = sys.argv[1], sys.argv[2]
df1 = pd.read_csv(csv1, header=0)
df2 = pd.read_csv(csv2, header=0)

summary = []
for label, df in [("Mode1", df1), ("Mode2", df2)]:
    lat = df["latency_ns"].astype(float)
    desc = lat.describe(percentiles=[.5, .95, .99])
    summary.append({
        "Mode":    label,
        "Count":   int(desc["count"]),
        "Mean(ns)":f"{desc['mean']:.1f}",
        "P50(ns)": int(desc["50%"]),
        "P95(ns)": int(desc["95%"]),
        "P99(ns)": int(desc["99%"]),
        "Max(ns)": int(desc["max"]),
    })

print("\n## Latency Comparison\n")
print(tabulate(summary, headers="keys", tablefmt="github"))

# 히스토그램
hist = os.path.expanduser("~/latency_compare.png")
bins = np.linspace(0, max(df1["latency_ns"].max(), df2["latency_ns"].max()), 200)

plt.figure(figsize=(6,4))
plt.hist(df1["latency_ns"], bins=bins, histtype="step", label="Mode1")
plt.hist(df2["latency_ns"], bins=bins, histtype="step", label="Mode2")
plt.title("Latency Comparison")
plt.xlabel("latency (ns)")
plt.ylabel("count")
plt.legend(); plt.grid(True); plt.tight_layout()
plt.savefig(hist)
print(f"\n✔ Saved histogram: {hist}")
```

사용:

```bash
chmod +x plot_latency_compare.py
python3 plot_latency_compare.py latencies_prio.csv latencies_tas.csv
```

---

이 스크립트 모음으로 **Prio vs TAS** 및 **노타스 비교**, **Payload‑TS 지연 측정**, **그래프·통계 분석**을 손쉽게 재현할 수 있습니다.
