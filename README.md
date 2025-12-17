# UNIVA 프로젝트 설명서

## 프로젝트 개요

- **목적**: 신약 개발 초기 단계(기초 연구–비임상 실험)에서 요구되는 ADMET(흡수·분포·대사·배설·독성) 특성 분석을 자동화하기 위해, 인지·추론 기반 AGI 에이전트 플랫폼을 구축한다. 본 플랫폼은 대규모 독성·약물동태 데이터와 온톨로지 기반 지식을 통합하여, 분자 수준의 ADMET 프로파일을 자율적으로 추론·해석할 수 있는 차세대 AI 시스템을 지향한다.
- **구현 내용**: 1차년도 1단계 목표로 **Generalized ADMET Inference 베이스라인**과 **Toxicity AI 프로토타입** 모델을 구축한다. 이를 통해 기존 신약개발 과정에서 반복되는 수작업 기반 독성 예측 및 ADMET 전주기 분석의 단절 문제를 해소하고, 능동적 의사결정이 가능한 self-evolving ADMET AI 에이전트 개발의 초석을 마련한다.

## 모델/런타임

- Generalized ADMET Inference Baseline: VLLM(OpenAI 호환) 엔드포인트(25TOXMC_Blowfish_v1.0.9-AWQ 기준)로 ChemCoTBench 평가 수행
- Toxicity AI Prototype: 동일 VLLM 엔드포인트로 독성 MMLU, Mobile-Eval-E 평가 수행
- `Toxicity AI Prototype/.env` 에서 `BASE_URL`(VLLM)과 `GPT_API_KEY`(GPT 채점) 관리

## 디렉터리 트리 & 파일 설명

```
Generalized ADMET Inference Baseline/
  ├─ chem_cot.py                   # ChemCoTBench 평가 스크립트
  └─ utils.py                      # JSONL 저장, EM 계산, 비동기 호출, <think> 제거 등

Toxicity AI Prototype/
  ├─ mmlu_toxic.py                 # 독성 MMLU 평가 스크립트
  ├─ mmlu_toxic.json               # 독성 MMLU 문제 200문항
  ├─ mobile_eval_e.py              # Mobile-Eval-E 플랜 생성(로컬 VLLM) + GPT 채점
  ├─ utils.py                      # JSONL 저장, EM 계산(MMLU), 비동기 호출, <think> 제거 등
  ├─ no_tool_chat_template_qwen3.jinja  # Qwen 채팅 템플릿(<think> 처리)
  └─ .env                          # BASE_URL(VLLM), GPT_API_KEY(GPT 채점) 환경 변수

run_vllm.sh                        # 25TOXMC_Blowfish_v1.0.9-AWQ VLLM 실행 예시(Volume/포트/GPU 수정 필요)
```

## 실행 및 평가 흐름

1) VLLM 서버 기동
   - 루트의 `run_vllm.sh`에서 마운트 경로(`-v`), 포트(`-p`), GPU(`CUDA_VISIBLE_DEVICES`)를 환경에 맞게 수정 후 실행(`chmod +x run_vllm.sh && ./run_vllm.sh`).
   - `Toxicity AI Prototype/.env`에 `BASE_URL`(예: `http://<host>:30002/v1/`), `GPT_API_KEY`를 채운다.

2) Toxicity MMLU 평가 (`Toxicity AI Prototype/mmlu_toxic.py`)
   - 작업 디렉터리를 `Toxicity AI Prototype/`로 두고 실행.
   - `mmlu_toxic.json`의 `system/prompt/answer`를 메시지로 보내고 EM 점수를 계산해 `MMLU_toxic_results.jsonl` 생성.

3) ChemCoTBench 평가 (`Generalized ADMET Inference Baseline/chem_cot.py`)
   - `Generalized ADMET Inference Baseline/` 아래에 `ChemCoTBench/`를 `datasets.load_from_disk`로 읽을 수 있게 배치.
   - 실행하면 응답 `"output"`과 `meta.reference`를 비교해 EM을 계산하고 `ChemCoT_results.jsonl` 생성.

4) Mobile-Eval-E 평가 (`Toxicity AI Prototype/mobile_eval_e.py`)
   - `GPT_API_KEY`를 `.env`에 채운 뒤 실행.
   - **액터**: `process_request_vl`이 로컬 VLLM 엔드포인트로 `plan/operations` JSON을 생성(필요 시 URL 수정).
   - **채점**: `judge_with_gpt`가 GPT로 루브릭·참고 동작을 비교해 점수 산출 후 `MobileEvalE_results.jsonl` 생성.

5) 결과 확인
   - 각 `*_results.jsonl`과 콘솔 `SUMMARY`/평균 점수를 확인.

## 환경/운영 참고

- `.env`는 `Toxicity AI Prototype/`에 있으며, 스크립트를 해당 디렉터리에서 실행하거나 `load_dotenv("<경로>")`로 직접 지정한다.
- VLLM 옵션(`gpu-memory-utilization`, `tensor-parallel-size`, `max-model-len`)은 GPU 자원에 맞춰 조정한다.
- `mobile_eval_e.py`의 `process_request_vl` 기본 URL(`http://192.168.0.202:25321/v1/`)이 다르면 실제 VLLM 주소로 수정한다.
- 응답에 `<think>` 블록이 포함돼도 `run_concurrent_worker`가 제거 후 JSON 파싱을 시도하므로 템플릿을 유지해도 된다.
