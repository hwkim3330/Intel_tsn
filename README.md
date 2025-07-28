<img width="1728" height="700" alt="image" src="https://github.com/user-attachments/assets/7e6a6367-473e-4c66-897c-9f5710d530fd" />
<img width="1840" height="699" alt="image" src="https://github.com/user-attachments/assets/86decc3c-7920-41af-94b5-ad0b2b4acb85" />

# Intel i210/i225v 기반 TSN TAS 성능 평가

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)  
[![Python 3.8+](https://img.shields.io/badge/Python-3.8%2B-orange.svg)](https://www.python.org/)  

> **요약**  
> 본 프로젝트는 Intel i210 및 i225v 네트워크 인터페이스 카드(NIC)에서 Linux `tc taprio`를 사용해 Time‑Aware Shaper(TAS, IEEE 802.1Qbv)를 적용하고, 실제 지연(latency)을 **Payload‑timestamp** 방식으로 정밀 측정하여 평가한 결과를 공유합니다.

---

## 🔍 실험 장비

- **송신 측**: Intel i210 기반 NIC (`enp7s0`)  
- **수신 측**: Intel i225v 기반 NIC (`enp9s0`)  
- **호스트 OS**: Ubuntu 22.04, Linux kernel ≥ 5.10  
- **필수 패키지**: `iproute2`, `ethtool`, `tc`(taprio), `hping3`, `tcpdump`, Python 3.8+ (`pandas`, `matplotlib`, `numpy`, `tabulate`)

---

## ⚙️ 실험 개요

1. **네임스페이스 분리**  
   - `ns1` (송신기)와 `ns2` (수신기)로 NIC를 분리  
2. **TAS 스케줄**  
   - **2 µs 사이클**: 1 µs Open / 1 µs Close  
   - **10 µs 사이클**: 5 µs Open / 5 µs Close  
3. **Payload‑timestamp 방식**  
   - 송신 시각(`time.time_ns()`)을 UDP 페이로드에 기록  
   - 수신 시각 – 송신 시각 = 실제 패킷 지연(latency)  
4. **토글 테스트**  
   - 3회 반복, 각 스케줄 10초간 유지 → 60초 동안 자동 진행  
5. **데이터 분석**  
   - `latencies.csv` → 산점도, 히스토그램, CDF, 요약 통계  

---

## 📈 주요 결과

### 1. Scatter Plot (Payload TS)

- **X축**: `tx_time % cycle_length` (사이클 내 전송 위치)  
- **Y축**: 실제 지연(latency, ns)  

<div align="center">
  <img src="figures/scatter_2us.png" width="45%" alt="2µs cycle" />  
  <img src="figures/scatter_10us.png" width="45%" alt="10µs cycle" />  
</div>

> ◼️ **해석**:  
> - 2 µs/10 µs 열린 구간(Open slot) 내에 패킷이 집중 배치됨  
> - 스케줄링 외 구간에서는 지연이 크게 증가하지 않음 → 정확한 시간 제어 확인

---

### 2. Latency Histogram

- **Y축**: 패킷 수  
- **X축**: 지연(latency, ns)

![Latency Histogram](figures/latency_histogram.png)

> ◼️ **해석**:  
> - 두 사이클 모두 주요 지연 분포가 2–4만 ns 구간에 집중  
> - 10 µs보다 2 µs 사이클이 분포가 조금 더 넓게 퍼짐

---

### 3. Latency CDF

- 누적 분포 함수(CDF)로 지연 특성 시각화

![Latency CDF](figures/latency_cdf.png)

> ◼️ **해석**:  
> - 95% 패킷이 3만 ns 이내에 수신  
> - P99 지점에서도 5만 ns 이하 → 매우 우수한 지연 보장

---

### 4. 요약 통계

| Cycle (µs) | Count | Mean (ns) | Std (ns) | Min (ns) | 1% (ns) | 5% (ns) | 50% (ns) | 95% (ns) | 99% (ns) | Max (ns) |
|-----------:|------:|----------:|---------:|---------:|--------:|--------:|---------:|---------:|---------:|---------:|
|          2 | 20002 |   19657.2 |   6646.0 |    14055 |    14543 |    14861 |    18366 |    33866 |    52178 |   220533 |
|         10 | 20002 |   19657.2 |   6646.0 |    14055 |    14543 |    14861 |    18366 |    33866 |    52178 |   220533 |

> ◼️ **해석**:  
> - 평균 지연 ≈ 19.7 µs, 표준편차 ≈ 6.6 µs  
> - P99 지점 52.2 µs → 실시간 요구조건 하에서 충분히 낮은 지연 분포

---

## 🚀 활용 방법

1. **클론 & 의존성 설치**  
   ```bash
   git clone https://github.com/hwkim3330/Intel_tsn.git
   cd Intel_tsn
   ./scripts/install_deps.sh
````

2. **네임스페이스·TAS 설정 & 측정**

   ```bash
   bash scripts/setup_namespaces.sh
   sudo ip netns exec ns1 python3 scripts/sender.py 10 20000 169.254.59.100 &
   sudo ip netns exec ns2 python3 scripts/receiver.py latencies.csv &
   sudo scripts/toggle_tas.sh 3 10
   ```
3. **분석 & 그래프 생성**

   ```bash
   python3 analysis/plot_latency.py latencies.csv
   ```

---

## 📄 License

MIT © hwkim3330

```
::contentReference[oaicite:0]{index=0}
```
