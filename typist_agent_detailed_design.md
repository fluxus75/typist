# Typist Agent 상세 설계 (Windows, 완전 로컬, Qwen3-ASR-1.7B)

## 1. 목표 및 제약

### 목표
- Windows PC에서 사용자가 커서를 두고 **특정 모드(단축키/트레이) ON** 시 마이크 입력을 받아 텍스트로 변환한다.
- **모드 OFF** 시 즉시 입력을 종료한다.
- 한국어/영문 혼합 환경에서 안정적으로 동작한다.

### 제약 조건
- ASR 모델은 **Hugging Face에서 사전 다운로드** 후 사내망으로 반입하여 로컬 경로에서 로드한다.
- 전체 시스템은 **완전 오프라인 로컬 동작**을 원칙으로 한다.
- 대상 OS는 Windows 10/11.

---

## 2. 시스템 아키텍처

```text
[Tray/Hotkey Controller]
          |
          v
 [Session State Machine] <-----> [Config Manager]
          |
          v
   [Audio Capture (WASAPI)]
          |
          v
   [Silero VAD Segmenter]
          |
          v
   [ASR Engine (Qwen3-ASR-1.7B)]
      |                 |
      |                 +--> Draft Text Stream (UI only)
      v
 Finalized Utterance
          |
          v
 [Text Injector (Clipboard Main)]
          |
          v
    Active App Caret
```

핵심 설계 원칙:
1. **무음 기반 동적 분할**(VAD)로 발화 단위를 생성한다.
2. 실시간성은 Draft로 제공하고, 최종 텍스트는 Accept 시점에 확정한다.
3. 한글 안정성을 위해 텍스트 주입의 기본 경로는 Clipboard + Paste로 한다.

---

## 3. 상태 머신 설계

세션 상태:
- `IDLE`: 대기
- `ARMED`: 모드 ON, 마이크 준비
- `LISTENING`: 음성 감지 전/후 수집 중
- `TRANSCRIBING_DRAFT`: 중간 결과 생성
- `FINALIZING`: 무음 감지 후 최종 확정
- `INJECTING`: 텍스트 주입
- `STOPPING`: 모드 OFF 처리
- `ERROR`: 예외 상태

주요 이벤트:
- `HOTKEY_TOGGLE_ON`
- `HOTKEY_TOGGLE_OFF`
- `VAD_SPEECH_START`
- `VAD_SPEECH_END`
- `DRAFT_READY`
- `FINAL_READY`
- `INJECT_DONE`
- `EXCEPTION`

핵심 전이:
- `IDLE -> ARMED -> LISTENING`
- `LISTENING -> TRANSCRIBING_DRAFT` (음성 구간 존재)
- `TRANSCRIBING_DRAFT -> FINALIZING` (무음 기준 충족)
- `FINALIZING -> INJECTING -> LISTENING`
- 모든 상태에서 `HOTKEY_TOGGLE_OFF -> STOPPING -> IDLE`

---

## 4. 오디오 파이프라인 (Silero VAD 기반)

### 4.1 캡처
- Windows WASAPI loop: 마이크 입력을 16kHz mono PCM으로 정규화.
- 입력 버퍼 단위: 20~30ms 프레임.

### 4.2 VAD
- 기본 VAD: **Silero VAD(PyTorch)**.
- 임계치 + hysteresis + hangover를 사용해 구간 안정화.

초기 권장값:
- `min_speech_ms`: 200
- `end_silence_ms`: 700
- `max_utterance_s`: 15
- `pre_roll_ms`: 150 (발화 시작 직전 보존)

### 4.3 세그먼트 전략
- 고정 길이 청크 금지.
- `speech_start`부터 `end_silence_ms` 만족 시점까지 하나의 발화로 처리.
- 장문 발화는 `max_utterance_s` 도달 시 강제 finalize 후 다음 발화로 이어감.

---

## 5. ASR 추론 전략 (Draft & Accept)

### 5.1 Draft
- 발화 중 주기적으로(300~500ms) 임시 텍스트 생성.
- Draft는 UI 내부 표시용이며 외부 앱에 주입하지 않는다.

### 5.2 Accept
- 무음 구간이 종료 조건을 만족하면 해당 발화를 최종 재추론.
- 최종 텍스트가 이전 Draft를 덮어쓴다.
- 외부 앱 주입은 **Accept 결과만** 수행.

### 5.3 환각 완화
- 지나치게 짧은 발화(예: < 250ms) 버림.
- 반복 토큰/비문 패턴 필터(후처리 단계).
- 발화 단위 컨텍스트만 유지(세션 전체 컨텍스트 누적 최소화).

---

## 6. 텍스트 주입 전략 (Windows 한글 안정성 중심)

### 6.1 기본 경로: Clipboard + Ctrl+V
1. 현재 클립보드 백업
2. 최종 텍스트를 Unicode로 클립보드에 설정
3. 활성 윈도우로 `Ctrl+V` 주입
4. 짧은 지연 후 기존 클립보드 복구

### 6.2 대체 경로
- `SendInput` 직접 타이핑: 영문 위주/일부 앱 전용 옵션
- `WM_IME_CHAR`: 고급 옵션(앱 호환성 검증 후)

### 6.3 예외 처리
- 붙여넣기 금지 앱/보안 필드 감지 시 사용자 알림
- 관리자 권한 앱과 권한 불일치 시 실패 로그 + 가이드
- 클립보드 경합 시 재시도 및 타임아웃 처리

---

## 7. 모델 관리 (오프라인 반입)

### 7.1 배포 절차
1. 외부망에서 HF snapshot 다운로드
2. SHA256 체크섬 생성
3. 사내 보안 절차에 따라 반입
4. 로컬 경로 고정 (예: `D:\models\Qwen3-ASR-1.7B`)

### 7.2 앱 시작 시 검증
- 필수 파일 존재 여부(config/tokenizer/weights)
- 체크섬 검증(옵션)
- GPU 사용 가능 여부 확인 후 fallback 정책 적용

### 7.3 런타임 정책
- GPU(CUDA) 우선
- 실패 시 CPU fallback
- 모델 로드 실패 시 세션 시작 금지 및 명확한 오류 메시지

---

## 8. 패키징/배포 전략

### 8.1 비권장
- PyInstaller 단일 대형 exe(용량/시작시간/업데이트 비효율)

### 8.2 권장 (Portable)
```text
/typist-portable
  /runtime        # Python venv 또는 임베디드 런타임
  /app            # agent 코드
  /models         # 분리된 모델 파일(반입)
  launcher.exe    # 경로 검증 후 실행
  config.yaml
```

### 8.3 업데이트 단위
- 앱 코드 업데이트
- 런타임 업데이트
- 모델 업데이트

각각 독립 교체 가능하도록 설계.

---

## 9. 로깅/관측성

로그 채널:
- `session.log`: 상태 전이, ON/OFF, 오류
- `audio.log`: 샘플링/VAD 이벤트
- `asr.log`: draft/final 지연, 토큰 길이
- `inject.log`: 대상 앱, paste 성공/실패

민감정보 정책:
- 원시 오디오 기본 비저장
- 텍스트 로그 비활성 기본값(디버그 모드에서만)

---

## 10. 초기 튜닝 파라미터

```yaml
audio:
  sample_rate: 16000
  frame_ms: 30
vad:
  provider: silero
  threshold: 0.5
  min_speech_ms: 200
  end_silence_ms: 700
  max_utterance_s: 15
asr:
  draft_interval_ms: 400
  finalize_on_pause_ms: 700
injector:
  method: clipboard_paste
  restore_clipboard: true
  restore_timeout_ms: 1500
```

---

## 11. 개발 마일스톤

### M1. 오프라인 추론 검증
- 로컬 모델 경로 로드 성공
- 샘플 WAV 추론 성공(GPU/CPU fallback 확인)

### M2. 실시간 파이프라인
- WASAPI 캡처 + Silero VAD 세그먼트 완성
- Draft/Accept 분리 동작 확인

### M3. 주입 안정화
- 메모장/Word/브라우저/메신저에서 Clipboard paste 검증
- 한글/영문 혼합 입력 검증

### M4. UX/운영
- 트레이 + 전역 핫키 + 상태 표시
- 오류 코드/로그/진단 수집 정비

### M5. 배포
- Portable 패키지 구성
- 사내 반입/설치/업데이트 가이드 문서화

---

## 12. 수용 기준 (Acceptance Criteria)

1. 인터넷 없이 앱 시작부터 입력 종료까지 동작한다.
2. 모드 ON/OFF 전환이 1초 이내 반응한다.
3. 일반 사무실 소음에서 오탐 발화율이 기준 이하(내부 정의 KPI).
4. 한국어 문장 입력 시 주요 앱에서 자모 분리 없이 정상 입력된다.
5. 모델 경로 오류/권한 오류/마이크 오류에 대해 사용자 친화적 안내를 제공한다.

---

## 13. 오픈 이슈

- Qwen3-ASR-1.7B의 정확한 런타임 권장 옵션(precision/quantization) 최적점 검증 필요
- 장시간 연속 사용 시 메모리 단편화 모니터링 필요
- 기업 보안 정책에 따른 클립보드 사용 허용 범위 사전 확인 필요

