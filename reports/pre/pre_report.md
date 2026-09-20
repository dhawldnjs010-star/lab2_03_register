# 실험 전 레포트: LAB2-03 레지스터: 데이터 저장과 이동

작성자: 상혁 (2025440084) / 작성일: 2026-09-20 / 소스 커밋: `<git rev-parse --short HEAD 결과 기입>` / workspace: `LAB1.code-workspace` (템플릿 v2.0.1) / OS: `<기입>` / Python: `<python --version 결과 기입>` / 시뮬레이터: Icarus Verilog `<iverilog -V 첫 줄 기입>`

> 이 레포트는 VS Code(Icarus) 시뮬레이션까지의 사전 검증이다. Vivado GUI와 실물 보드 결과는 실험 후 레포트([post](../post/post_report.md))에서 다룬다. 시각은 clk 상승 에지(5, 15, 25, … ns)의 1 ns 뒤, 즉 TB가 비교하는 시각으로 적었다.

## 목적과 예상 동작

입력을 저장하는 `stored`와 저장값을 전달받는 `value` 두 레지스터를 설계한다. load와 transfer가 동시에 1일 때 nonblocking 할당이 이전 값을 전달하는 과정을 계산한다.

### 포트 (`register_pair.v`의 `register_pair`)

| 포트 | 방향 | 비트 폭 | 설명 |
|---|---|---|---|
| clk | in | 1 | 상승 에지 기준 클록 |
| rst | in | 1 | 동기 active-high 리셋. load·transfer보다 우선 |
| load | in | 1 | 1이면 stored←data_in |
| transfer | in | 1 | 1이면 value←stored(에지 직전의 stored) |
| data_in | in | 4 | 저장할 입력 |
| stored | out (reg) | 4 | 저장 레지스터 |
| value | out (reg) | 4 | 전달 레지스터 |

최상위(`lab2_register.v`, `lab2_register`): `load = press && switches[0]`, `transfer = press && switches[1]`, `data_in = switches[7:4]`, `led = {stored, value}`. 스위치를 먼저 정한 뒤 N8을 눌러야 한다.

### 동작 규칙과 경계 입력

- 규칙: rst=1이면 stored=value=0. 아니면 load=1이면 stored←data_in, transfer=1이면 value←stored. 두 할당은 같은 에지에서 nonblocking이므로 value는 에지 직전의 stored를 받는다.
- 정상·경계: 동시 load·transfer: value는 이전 stored, stored는 새 입력. 다음 에지에서 value가 새 stored로 바뀐다. 두 제어가 모두 0이면 입력이 바뀌어도 유지. rst는 둘 다 1이어도 우선.
- 시간 기준: TB 클록은 10 ns 주기(상승 에지 5, 15, 25, … ns)이고 입력은 에지 1 ns 뒤에 바꾼다. 레지스터는 다음 상승 에지에서 갱신된다. 실제 보드 클록(1 kHz)은 시뮬레이션의 10 ns와 별개다.

## 소스와 테스트벤치

- 설계 top: `lab2_register` / 시뮬레이션 top: `tb_register_pair`
- 소스: [`src/register_pair.v`](../../src/register_pair.v), [`src/input_frontend.v`](../../src/input_frontend.v), [`src/lab2_register.v`](../../src/lab2_register.v)
- 테스트벤치: [`sim/tb_register_pair.sv`](../../sim/tb_register_pair.sv)
- 제약: [`constraints/lab2_register.xdc`](../../constraints/lab2_register.xdc)
- 설정: [`simulation.json`](../../simulation.json) (sources 3개, testbench `sim/tb_register_pair.sv`, simulation_top `tb_register_pair`)

| 파일 | 역할 |
|---|---|
| `src/register_pair.v` | 핵심 동작을 담은 코어 `register_pair`. TB가 이 모듈을 직접 검사한다. |
| `src/input_frontend.v` | 버튼·스위치를 클록에 맞추는 입력 회로. 리셋 2단 해제 동기화, 버튼·스위치 2단 동기화 플립플롭, STABLE_CYCLES=20(1 kHz에서 20 ms) 안정 확인 뒤 한 클록짜리 `press` 펄스를 만든다. |
| `src/lab2_register.v` | 보드 top. 프런트엔드와 코어를 연결하고 LED로 출력한다. |
| `sim/tb_register_pair.sv` | 입력 자극, 기대값 계산, 자동 비교(`check`), PASS/FAIL 출력, VCD 생성, watchdog. |
| `constraints/lab2_register.xdc` | 핀 번호·전압과 1 kHz 클록 정의. Icarus는 XDC를 읽지 않으므로 이 사전 시뮬레이션에는 사용되지 않는다. |
| `simulation.json` | VS Code 시뮬레이션 작업이 읽는 소스 목록·테스트벤치·시뮬레이션 top. |

### 테스트벤치 동작

- 자극 순서: 리셋 → data_in=A, load=1 → load=0, data_in=3, transfer=1 → load=1(transfer=1 유지) → 한 에지 더 → load=transfer=0, data_in=F → rst=1, load=transfer=1. 총 7회 검사.
- 검사 횟수: 7 (reset, load does not transfer, transfer stored not live input, simultaneous uses old stored, next edge transfers new stored, both hold, reset beats both controls)
- 종료·watchdog: 마지막 검사 뒤 `finish` task가 `LAB2_PASS register_pair checks=N`을 출력하고 `$finish`한다. 별도로 100000 ns(100 µs) 뒤에 `watchdog timeout`으로 `$fatal` 처리한다. 예상 종료 시각은 66 ns이다.
- 이 TB는 코어 `register_pair`만 시험한다. 입력 동기화·디바운스와 실제 핀·타이밍이 통과했다는 뜻은 아니다.

### XDC 설명

`lab2_register.xdc`은 포트 이름을 `lab2_register.v`과 맞춰 핀을 지정한다. 모든 I/O는 `LVCMOS33`이고 `create_clock -name trainer_1khz -period 1000000.000 [get_ports clk]`로 주 클록을 1 kHz(주기 1,000,000 ns)로 정의하며 `set_false_path -from [get_ports {rst button sw[*]}]`로 비동기 입력을 타이밍 경로에서 제외한다.

| 포트 | 핀 | 보드 대응 |
|---|---|---|
| clk | B6 | 1 kHz 주 클록 |
| rst | K4 | 리셋 (active-high) |
| button | N8 | 스텝 버튼 |
| sw[7:0] | U4(sw[0]), V4, W1, W4, T1, U2, W3, Y1(sw[7]) | DIPSW8..DIPSW1 (DIPSW1..8 = sw[7]..sw[0]) |
| led[7:0] | N5(led[0]), M1, M3, M7, N7, M2, M4, L4(led[7]) | LED0..LED7 |

## VS Code 실행 과정

1. File → New Window → File → Open Workspace from File...로 `LAB1.code-workspace`를 연다. 확장(slang, VaporView, vscode-pdf)을 설치한다.
2. RTL·TB·XDC·`simulation.json`을 직접 입력하고 File → Save All.
3. Terminal → Run Task... → `01 Check tools`로 Git·Python·iverilog·vvp 버전을 확인한다.
4. `02 Simulate`를 실행해 `LAB2_PASS`와 종료 시각을 확인한다.
5. `03 Open waveform`으로 `build/sim/wave.vcd`를 VaporView로 연다. 신호: clk, rst, load, transfer, data_in, stored, value (모두 16진수).

정상 실행 로그(본인 로그로 교체하고 `evidence/pre/`에 복사):

```text
LAB2_PASS register_pair checks=7
sim/tb_register_pair.sv:19: $finish called at 66000 (1ps)
```

- 본인 실행 로그: [`../../evidence/pre/lab2_03_normal.log`](../../evidence/pre/lab2_03_normal.log)
- VCD: [`../../evidence/pre/lab2_03_wave_normal.vcd`](../../evidence/pre/lab2_03_wave_normal.vcd)
- 파형 캡처(VaporView): `evidence/pre/lab2_03_wave_full.png`(전체 Zoom Fit), `evidence/pre/lab2_03_wave_zoom.png`(상승 에지 확대)
- 오류: 첫 실행에서 발생한 오류가 있으면 첫 오류 → 수정 → 재실행 로그 순서로 기록한다. (없으면 "없음", Python 실행 경로를 고쳤다면 그 내용 기입)

### 사전 파형 해석

| 시간 구간 | 입력 | 예상 | 실제 파형 | 해석 |
|---|---|---|---|---|
| 6 ns | rst=1 | stored=0, value=0 | stored=0, value=0 | 동기 리셋. |
| 16 ns | rst=0, data_in=A, load=1 · 에지 15 ns | stored=A, value=0 | stored=A, value=0 | load만: stored에 A 저장, value는 그대로(load가 transfer하지 않음). |
| 26 ns | load=0, data_in=3, transfer=1 · 에지 25 ns | stored=A, value=A | stored=A, value=A | transfer는 현재 입력 3이 아니라 stored A를 전달. |
| 36 ns | load=1, transfer=1 · 에지 35 ns | stored=3, value=A | stored=3, value=A | 경계: 동시 동작에서 value는 이전 stored(A), stored는 새 입력 3. |
| 46 ns | 같은 제어 유지 · 에지 45 ns | stored=3, value=3 | stored=3, value=3 | 다음 에지에서 value가 새 stored 3을 받음. |
| 56 ns | load=0, transfer=0, data_in=F · 에지 55 ns | stored=3, value=3 | stored=3, value=3 | 두 제어가 0이면 입력 F를 무시하고 유지. |
| 66 ns | rst=1, load=1, transfer=1 · 에지 65 ns | stored=0, value=0 | stored=0, value=0 | 리셋이 두 제어보다 우선. |

표의 값은 상승 에지 직후 안정된 값이다. `LAB2_PASS`만 적지 않고 각 행에서 입력, 이전 상태, 다음 상태를 비교한다. 파형 캡처에서 위 시각을 확대해 본인 화면으로 확인한다.

## 코드 수정·실패·복구 실험

- 변경: transfer의 전달 대상을 `stored`에서 `data_in`으로 바꾼다(`value <= stored` → `value <= data_in`).
- 변경한 파일과 위치: `src/register_pair.v` 13행
- 테스트벤치 기대값은 바꾸지 않는다.

```diff
-    if (transfer) value <= stored;
+    if (transfer) value <= data_in;
```

실행 전 계산: 26 ns 에지(25 ns)에서 stored=A, data_in=3이므로 정상 회로는 value=A를 받는다. 변경 회로는 현재 입력 3을 받아 {stored,value}=A3이 되고, TB의 기대값 AA와 어긋난다.

| 단계 | 소스 커밋 또는 해시 | 실행 폴더·로그 링크 | 입력·기대값·실제값 | 해석 |
|---|---|---|---|---|
| 정상 코드 | `<커밋/해시 기입>` | [normal.log](../../evidence/pre/lab2_03_normal.log) | 26 ns 기대 value=A, 실제 value=A. `LAB2_PASS register_pair checks=7` | 모든 검사 통과, 66 ns 종료. |
| 지정한 RTL 변경 | `<커밋/해시 기입>` | [mod.log](../../evidence/pre/lab2_03_mod.log) | 26 ns 기대 value=A, 실제 value=3. `LAB2_FAIL transfer stored not live input time=26000`, `FATAL: sim/tb_register_pair.sv:12: check failed` | `transfer stored not live input` 검사가 변경을 발견했다(로그의 time은 ps 단위, 26000 ps = 26 ns). |
| 원래 코드로 복구 | `<커밋/해시 기입>` | [recover.log](../../evidence/pre/lab2_03_recover.log) | 복구 후 전체 검사 재실행. `LAB2_PASS register_pair checks=7`, `$finish called at 66000 (1ps)` | PASS와 종료 시각이 정상 실행과 같고 새 VCD를 확인한다. |

- 첫 실패 이후에는 `$fatal`로 시뮬레이션이 끝나므로 뒤의 검사는 실행되지 않는다. 변경 전후 파형은 각각 별도 폴더에 보관한다.
- 문법 오류를 경험했다면 오류 위치로 이동한 화면, 원인, 수정 내용과 재실행 로그도 이 절에 연결한다.

## 보드 실험 계획

- 부품·프로젝트: Vivado 2026.1, RTL Project `lab2_register`, 부품 `xc7s75fgga484-1`(정확히 -1). Design Sources: `register_pair.v`, `input_frontend.v`, `lab2_register.v`(Copy sources 해제). Simulation Sources: `tb_register_pair.sv`(Set as Top: `tb_register_pair`). Constraints: `lab2_register.xdc`. Project Summary의 Top module name은 `lab2_register`이다.
- 장비 클록·제약: Combo II-DLD S75 주 클록 B6을 1 kHz로 맞추고 XDC의 `trainer_1khz`(1,000,000 ns)와 일치시킨다.
- 입력: DIPSW1..4=`sw[7:4]`=data_in, DIPSW7=`sw[1]`=transfer, DIPSW8=`sw[0]`=load, N8=press, K4=리셋. 주 클록은 1 kHz. 스위치를 먼저 안정시킨 뒤 N8을 누른다.
- 출력: LED[7:4]=stored, LED[3:0]=value.

| 조작 | 예상 LED / 동작 |
|---|---|
| sw=A1, N8 한 번 | LED=A0 (load만) |
| sw=32, N8 한 번 | LED=AA (transfer만) |
| sw=33, N8 한 번 | LED=3A (동시 동작) |
| sw=33 그대로 다시 누름 | LED=33 |
| sw=F0, N8 한 번 | LED=33 유지 (제어 0) |
| K4 초기화 | LED=00 |

sw 값은 16진수 표기이며 상위 4비트가 data_in, 하위 2비트가 transfer·load이다.

촬영할 장면: 보드 전체(배선·입력·출력이 함께 보이는 사진)와 위 표의 조작별 LED 상태 사진·영상. 예상되는 차이: 버튼·스위치는 동기화·안정 확인 지연이 있고, 빠른 신호는 눈으로 구분되지 않을 수 있다.

이 단계에서는 Vivado GUI와 실물 보드의 결과를 수행한 것처럼 기록하지 않는다. 합성·구현·bit 생성과 실제 장치 기록은 실험 후 레포트에서 다룬다.
