# UART FIFO Loopback

PC에서 보낸 직렬 데이터를 UART로 받아 FIFO 두 개를 거쳐 다시 PC로 돌려보내는 루프백 시스템입니다.
RTL 설계부터 테스트벤치 기반 검증, Basys-3 보드 시연까지 혼자 진행했습니다.

- **기간**: 2026.10 (대한상공회의소 AI 시스템반도체 설계 2기 미니 프로젝트)
- **인원**: 개인
- **툴/보드**: Verilog HDL, Vivado 2020.2, XSim, Digilent Basys-3 (Artix-7 XC7A35T)

## 결과 요약

| 검증 항목 | 결과 |
|---|---|
| FIFO 단독 (full · empty 경계 조건) | 4개 적재 후 full, 5번째 push 무시, 순서대로 출력 후 empty |
| Loopback 1 byte | `rx_done` → UART_TX 송신 시작까지 3 clk (30 ns) |
| Loopback 랜덤 16 byte | **PASS 16 / FAIL 0** (scoreboard 자동 판정) |
| 보드 시연 | ComPort Master로 보낸 문자 · 16진수가 그대로 에코 |

## 구조

```
PC ──rx──▶ uart_rx ──▶ FIFO_RX ──▶ FIFO_TX ──▶ uart_tx ──tx──▶ PC
              ▲                                    ▲
        baud_tick_gen (9600 bps × 16 오버샘플링)  ─┘
```

| 모듈 | 역할 |
|---|---|
| `baud_tick_gen` | 100 MHz → 9600 × 16 Hz tick (F_COUNT = 651, 오차 +0.006%) |
| `uart_rx` | 직렬 → 병렬, 비트 중앙(8 tick)에서 샘플 |
| `fifo` × 2 | 4-depth, 8 bit 원형 버퍼 |
| `uart_tx` | 병렬 → 직렬, `tx_busy` 출력 |

핸드셰이크: `push = rx_done`, FIFO 간 `push = ~empty` · `pop = ~full`, 송신부 `tx_start = ~empty` · `pop = ~tx_busy`

## 검증

| No | 테스트 | 자극 | 기대 결과 |
|---|---|---|---|
| 1 | FIFO 단독 (`tb/tb_fifo.v`) | push 5회 → pop 5회 → push · pop 동시 | full 시 push 무시, empty 시 pop 무시, 순서 보장 |
| 2 | Loopback 1 byte (`tb/tb_uart_fifo_loopback.v`) | `8'h24` 1 frame | 3 clk 안에 UART_TX로 전달, 같은 프레임 송신 |
| 3 | Loopback 랜덤 | `$random` 16 byte | 송신값 = 수신값 16회 |

테스트벤치 구성
- `sender_uart`: Start · Data 8 bit (LSB first) · Stop을 1 bit 간격(104,160 ns)으로 인가
- `receive_uart`: Start 하강 감지 → 반 비트 후 확인 → 8회 샘플 → Stop 확인
- `score_board`: 기대값과 수신값 비교 후 pass / fail 카운트

## 발견한 문제와 남은 과제

- **`tx_busy` 1클럭 지연**: `uart_tx`가 IDLE에서 `tx_start`를 받은 뒤 START 상태에 들어가서야 `tx_busy`가 1이 됩니다. 그 1클럭 동안 FIFO_TX의 `pop = ~tx_busy`가 열려 있어, FIFO_TX에 2 byte 이상 쌓여 있으면 데이터 하나가 유실될 수 있습니다. (코드 분석 결과, 시뮬레이션 재현 예정)
  - 수정안: IDLE에서 `tx_start`를 받는 순간 `tx_busy_next = 1`
- **검증 공백**: 현재 테스트벤치는 1 byte 송신 → 수신 완료 → 다음 송신 순서라 FIFO에 최대 1 byte만 머뭅니다. full · overflow와 위 버그를 재현하려면 연속 송신 테스트가 필요합니다.

## 파일

```
rtl/   fifo.v, uart_fifo_loopback.v (top), uart_rx.v, uart_tx.v
tb/    tb_fifo.v, tb_uart_fifo_loopback.v
docs/  report.pdf (결과 보고서), demo.mp4 (보드 동작 영상)
```
