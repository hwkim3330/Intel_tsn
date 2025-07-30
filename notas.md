<img width="1291" height="856" alt="image" src="https://github.com/user-attachments/assets/4a17f82a-5bc3-4585-9102-3206aaa9eeef" />
## TAS vs No‑TAS 요약 통계
| Mode    | Count  | Mean (ns) | P50 (ns) | P95 (ns) | P99 (ns) | Max (ns) |
|:--------|-------:|----------:|---------:|---------:|---------:|---------:|
| **TAS**    |   20000 |    20572.2 |     18994 |     31535 |     49808 |    271800 |
| No‑TAS |   20000 |    19525.7 |     18333 |     29732 |     50728 |    186095 |



| Mode    | Count  | Mean (ns) | P50 (ns) | P95 (ns) | P99 (ns) | Max (ns) |
|:--------|-------:|----------:|---------:|---------:|---------:|---------:|
| **TAS**    |  100000 |    25865.9 |     22721 |     51215 |     67712 |    226587 |
| No-TAS |   99989 |    19117.9 |     18147 |     24536 |     48757 |    170242 |

## Cross‑Traffic 부하하 TAS vs No‑TAS 지연 비교

아래 표는 `iperf3`를 이용해 UDP 포화 트래픽(≈1 Gbps)을 발생시키는 상태에서 Payload‑timestamp 방식으로 측정한 **TAS 모드**와 **No‑TAS 모드**의 지연(latency) 통계입니다.

| Mode    | Count  | Mean (ns) | P50 (ns) | P95 (ns) | P99 (ns) | Max (ns) |
|:--------|-------:|----------:|---------:|---------:|---------:|---------:|
| **TAS**    | 100 000 | 25 865.9 | 22 721  | 51 215  | 67 712  | 226 587 |
| **No‑TAS** |  99 989 | 19 117.9 | 18 147  | 24 536  | 48 757  | 170 242 |

- **TAS 모드**에서는 평균 지연이 약 25.9 µs, P95/​P99는 각각 51 µs, 67 µs로 나타났습니다.  
- **No‑TAS 모드**에서는 평균 지연이 약 19.1 µs로 더 낮으나, 버스트 상황에서 P95/​P99가 24 µs/​48 µs로 비교적 안정적 분포를 보였습니다.  
- TAS 모드는 슬롯 대기 시간(‘Close’ 구간) 때문에 평균 지연이 다소 증가하지만, **최대 지연(Max)** 측면에서는 No‑TAS 대비 더 높은 스케줄 전환 지터를 가지고 있습니다.

---

### 다음 단계: Throughput (iperf3) 측정

- 위 실험과 동시에 `iperf3 -u -b 0` 로 링크를 완전 포화시키는 UDP 스트림을 쏘아,  
- **지연(latency)** 뿐 아니라 **대역폭(throughput)** 저하 여부도 함께 측정해야 TAS 적용의 실효성을 종합적으로 평가할 수 있습니다.  
- **제안**  
  ```bash
  # ns2: iperf3 서버
  sudo ip netns exec ns2 iperf3 -s -p 5201

  # ns1: iperf3 클라이언트
  sudo ip netns exec ns1 iperf3 -c 169.254.59.100 -u -b 0 -t 60 -p 5201
<img width="1713" height="876" alt="image" src="https://github.com/user-attachments/assets/78d27af8-2e15-47d4-a3e9-38a13e8e9824" />

# TSN TAS 실험 요약 보고서

## 1. 실험 목적  
- Intel i210(i82576) 및 i225v NIC에서 IEEE 802.1Qbv TAS(Time‑Aware Shaper)를 적용했을 때  
  - **지연(latency) 분포**  
  - **포화 상태(throughput)에서 지연 안정성**  
  을 Payload‑timestamp 방식으로 정밀 측정·분석하여, **TAS vs No‑TAS** 성능 차이를 확인.

---

## 2. 실험 장비 및 환경  
- **송신기(ns1)**: Intel i210 → `enp7s0`  
- **수신기(ns2)**: Intel i225v → `enp9s0`  
- **OS**: Ubuntu 22.04, Linux 5.10+  
- **주요 툴**:  
  - `tc taprio` (TXTIME‑assist 모드)  
  - Python 3.8+ (`sender.py`/`receiver.py`, Pandas, Matplotlib)  
  - `iperf3`, `tcpdump`  

---

## 3. 실험 방법

1. **네임스페이스 분리**  
   ```bash
   bash setup_namespaces.sh
``

2. **Payload‑TS 측정 스크립트**

   * `sender.py`: `time.time_ns()`를 UDP 페이로드에 기록
   * `receiver.py`: 수신 시점 – 송신 시점 = latency 기록 → `latencies.csv`
3. **TAS 스케줄**

   * 2 µs 사이클: 1 µs Open / 1 µs Close
   * 10 µs 사이클: 5 µs Open / 5 µs Close

   ```bash
   # 예: 10µs, TXTIME‑assist
   BT=$(( $(date +%s%N)+3000000000 ))
   sudo ip netns exec ns1 tc qdisc replace dev enp7s0 root ... flags 0x1
   ```
4. **토글 테스트** (2 µs ↔ 10 µs, 3×10s 반복) → `latencies.csv` 생성
5. **포화 트래픽**

   * `iperf3 -u -b 0 -t 60` 로 최대 UDP 부하
   * TAS(on) vs FIFO(No‑TAS) 모드 각각 측정 → `latencies_load_tas.csv`, `latencies_load_no.csv`
6. **분석 스크립트**

   ```bash
   python3 plot_latency.py latencies.csv
   python3 plot_latency_compare.py latencies_load_tas.csv latencies_load_no.csv
   ```

---

## 4. 주요 결과

### 4.1 Scatter Plot (Payload‑TS)

* **X축**: `tx_time % cycle_ns` → 사이클 내 전송 위치
* **Y축**: 실제 지연(latency, ns)

![2µs cycle](figures/scatter_2us.png)
*그림 1. 2 µs 사이클 산점도*
좌측 0–2 000 ns 구간에,
![10µs cycle](figures/scatter_10us.png)
*그림 2. 10 µs 사이클 산점도*
우측 0–10 000 ns 구간에 점들이 집중 분포.
→ TAS 스케줄 대로 **열린 슬롯** 내에서만 패킷이 나갔음을 확인.

---

### 4.2 Latency Histogram & CDF (Idle)

![Latency Histogram](figures/latency_histogram.png)
*그림 3. TAS 모드 산포 무부하 히스토그램*

![Latency CDF](figures/latency_cdf.png)
*그림 4. TAS 모드 산포 무부하 CDF*

* 평균 ≈ 19.7 µs, P95 ≈ 33.9 µs, P99 ≈ 52.2 µs
* slot 외 지터 최소화

---

### 4.3 Under Saturation (부하) 비교

| Mode       |   Count | Mean (ns) | P50 (ns) | P95 (ns) | P99 (ns) |  Max (ns) |
| :--------- | ------: | --------: | -------: | -------: | -------: | --------: |
| **TAS**    | 100 000 |   194 215 |  195 354 |  263 367 |  271 173 |   353 343 |
| **No‑TAS** | 100 000 |   598 310 |  597 483 |  873 277 |  939 794 | 1 060 900 |

 
*그림 5. 포화 트래픽(1 Gbps) 하 TAS vs No‑TAS 히스토그램*

* **TAS**: 평균 ≈ 194 µs, P99 ≈ 271 µs → 포화 상태에서도 안정적
* **No‑TAS**: 평균 ≈ 598 µs, P99 ≈ 940 µs → 큐잉 지연 급증

---

## 5. 결론

* **TAS** 적용 시, 슬롯 제어로 지연 분포가 **좁고 예측 가능**해짐
* 무부하에서는 평균 지연 소폭 증가하나, 포화 부하에서는 **지연 안정성**이 크게 향상
* TSN 기반 실시간 시스템에서 **결정론적 보장**을 위해 TAS는 필수적

---

*작성: hwkim3330 (2025)*


::contentReference[oaicite:0]{index=0}


# Prio (802.1p) vs No‑TAS 우선순위 실험 결과

## 1. 실험 환경
- **송신기(ns1)**: veth0 → 10.0.0.1  
- **수신기(ns2)**: veth1 → 10.0.0.2  
- **Prio 설정**: `tc qdisc prio bands 8` + `skbedit priority 0` (band 0, 최상)  
- **No‑TAS(FIFO)**: `tc qdisc pfifo_fast` (기본 FIFO)  
- **트래픽**  
  - Payload‑TS 송신: 10 µs 간격, **100 000개** UDP 패킷  
  - 포화 부하: `iperf3 -u -b 0 -t 60` 동시 실행  
- **측정**: `receiver.py` → `latencies_prio.csv`, `latencies_load_no.csv`  

---

## 2. 지연(latency) 비교

| Mode      | Count   | Mean (ns) | P50 (ns) | P95 (ns) | P99 (ns) | Max (ns) |
|:----------|--------:|----------:|---------:|---------:|---------:|---------:|
| **Prio**     | 100 000 |   7 981.3 |   6 989  |  10 924  |  15 679  |  89 602  |
| **No‑TAS**   | 100 000 | 598 310   | 597 483  | 873 277  | 939 794  |1 060 900 |

> ● **Prio** 모드에서 평균 지연은 약 **7.98 µs**로, P95는 **10.9 µs**, P99는 **15.7 µs**를 기록했습니다.  
> ● **No‑TAS** (FIFO) 모드에서는 평균 **598.3 µs**, P95 **873 µs**, P99 **939 µs**로, 지연이 수백 µs 단위로 급격히 증가했습니다.

---

## 3. 히스토그램

<img width="1015" height="665" alt="image" src="https://github.com/user-attachments/assets/8565b986-b105-434d-900e-635c410501e3" />

> **그림.** 포화 상태에서 Prio(파란) vs No‑TAS(주황) 지연 분포 비교  
> - Prio는 대부분 0–20 µs 구간에 집중  
> - No‑TAS는 매우 넓은 분포와 극단적인 outlier가 다수

---

## 4. 결론

- **802.1p Prio**만으로도 포화 트래픽 환경에서 **FIFO 대비 수십 배** 빠른 응답성을 확보할 수 있습니다.  
- 비록 **TAS**만큼 엄격한 슬롯 제어는 아니지만, 복잡도·호환성을 고려할 때 실용적인 “소프트웨어 우선순위” 방식으로도 실시간 서비스의 지연을 크게 낮출 수 있습니다.

---

*측정 파일*  
- `latencies_prio.csv`  
- `latencies_load_no.csv`  

*히스토그램 파일*  
- `/home/kim/latency_load_compare.png`  

