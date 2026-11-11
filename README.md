## 1. 실행 환경

| 항목 | 요구 사항 |
| --- | --- |
| OS | Linux x86_64 (Ubuntu 권장) |
| Python | 3.10 |

---

## 2. 설치

```bash
git clone https://github.com/LeeJH0904/CrewACI.git
cd CrewACI

sudo apt update
sudo apt install -y python3 python3-pip python3-venv

python3 -m venv .aciarena
source .aciarena/bin/activate

pip install -e .
pip install torch python-dotenv

#가상환경 종료 시
deactivate

```

---

## 3. 모델·API 설정

### 3.1 API 키

저장소 최상위에 `.env` 파일 생.

```
OPENAI_API_KEY=sk-...
```

### 3.2 모델 설정 파일

```yaml
# configs/g7_model.yaml
provider: openai
base_url: https://api.openai.com/v1
model_name: gpt-4o-mini-2024-07-18
temperature: 0.0
max_tokens: 1024
seed: 42


# configs/g7_judge.yaml
provider: openai
base_url: https://api.openai.com/v1
model_name: gpt-4o-mini-2024-07-18
temperature: 0.0
max_tokens: 256 
seed: 42

```



---

## 4. 벤치마크 실행 순서

### 4.1 실행 계획 확인 (API 호출 없음)
```bash

# --execute 를 붙이지 않으면 모델을 호출하지 않고 실행 계획만 출력.
python benchmark.py --mas camel --task_domain math --suite hijacking

```

### 4.2 소규모 실행
```bash

# --limit 로 task 수를 제한하여 실행해 API 설정과 기록이 정상인지 먼저 확인. 
python benchmark.py --mas camel --task_domain math --suite hijacking --limit 1 --execute --experiment_id smoke-test

```

### 4.3 정상 성능(BU) 측정

```bash
python benchmark.py --mas <MAS> --task_domain <math|code> --suite benign --execute --experiment_id <실험ID>
```

### 4.4 공격 평가(UA·ASR) 측정

```bash
python benchmark.py --mas <MAS> --task_domain <math|code> \
  --suite <hijacking|disruption|disclosure> \
  --malicious_agents <대상 에이전트 1명> \
  --execute --experiment_id <실험ID>
```

예시:

```bash
# CrewAI, code 도메인, hijacking, finalizer 대상
python benchmark.py --mas crewai_seq_nodeleg --task_domain code --suite hijacking  --malicious_agents finalizer --execute --experiment_id crewai_exp

```

### 4.5 중단된 실행 이어서 진행

```bash
# 완료된 run은 건너뛰고 남은 run만 실행
python benchmark.py ... --execute --experiment_id <실험ID> --resume

# 일시적 오류로 실패한 run까지 재시도
python benchmark.py ... --execute --experiment_id <실험ID> --resume --retry_errors
```

`--retry_errors`의 재시도 대상 오류는 `APITimeoutError`, `APIConnectionError`, `RateLimitError`, `TimeoutError`로 한정. 
---

## 5. MAS별 에이전트

| `--mas` | 지원 도메인 | 선택 가능 에이전트 | 
| --- | --- | --- |
| `crewai_seq_nodeleg` | math, code | `solver`, `reviewer`, `finalizer` |
| `metagpt` | code | `product_manager`, `architect`, `project_manager`, `engineer`, `qa_engineer` |
| `autogen` | math, code | `assistant`, `user_proxy` |
| `camel` | math, code | `assistant`, `user_proxy`, `critic`, `task_specifier` |
| `sc` | math, code | `sc1`, `sc2`, `sc3`, `sc4`, `sc5`, `aggregate` |
| `llm_debate` | math, code | `debater_0`, `debater_1`, `debater_2`, `aggregator` |
| `agentverse` | math, code | `role_assigner`, `solver`, `critic_0`, `evaluator` |

---

## 6. 명령행 옵션

| 옵션 | 값 / 기본값 | 설명 |
| --- | --- | --- |
| `--mas` | 5절의 7종 | 평가할 MAS |
| `--task_domain` | `math`, `code` · `code` | 평가 도메인 |
| `--suite` | `benign`, `hijacking`, `disruption`, `disclosure` · `hijacking` | 공격 목표. `benign`은 공격 없음 |
| `--malicious_agents` | 에이전트 이름 1개 | 공격 대상 에이전트 |
| `--execute` | 플래그 | 실제 모델 호출 허용. 생략 시 계획만 출력 |
| `--experiment_id` | 사용자 지정 | 기록 폴더 이름 |
| `--output_dir` | `outputs/~` | 기록 루트 폴더 |
| `--limit` | 양의 정수 · 없음 | 앞에서부터 N개 task만 실행 |
| `--resume` | 플래그 · off | 완료된 run 재사용 |
| `--retry_errors` | 플래그 · off | 일시적 오류 run만 재시도 (최대 3회 시도) |

---

## 7. 출력 구조

```text
<output_dir>/<experiment_id>/
├── runs.jsonl        # run별 결과 (utility/attack 판정, 상태, config_hash 등)
├── messages.jsonl    # run별 에이전트 메시지와 공격 주입 기록
├── configs/<config_hash>.json   # 실행 설정 스냅샷
├── .writer.lock
└── .run_locks/
```

- `runs.jsonl`과 `messages.jsonl`은 `run_id`로 연결됨.
- 같은 `--experiment_id`로 실행하면 기존 파일에 누적.
- 실행이 끝나면 계획한 run과 기록이 일치하는지 자동 감사 진행 후 실패 시 오류로 종료.

---

## 9. 폴더 구조

```text
CrewACI/
├── benchmark.py                 # 벤치마크 실행 진입점
├── configs/
│   ├── g7_model.yaml            # 에이전트 모델 설정
│   └── g7_judge.yaml            # Judge 모델 설정
├── manifests/g7/                # task·공격·MAS·대상 에이전트 고정 목록
├── aciarena/
│   ├── mas/                     # MAS 구현 (crewai 포함)
│   ├── attacks/                 # 공격 구현과 공격 카탈로그
│   ├── agent_components/        # 에이전트 / LLM 공통 구성요소
│   ├── evaluation/              # 실행기, 기록, 감사, 채점, 데이터셋
│   ├── defenses/                # 방어 모듈 (본 벤치마크에서는 미사용)
│   └── utils/
├── scripts/g4/run_g4_pilot.py   # config_hash 계산용 (실행 X)
├── requirements.lock
└── setup.py
```
