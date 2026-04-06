# Claw Code 빌드 + OpenAI Codex 백엔드 연결

## Context
Claw Code(Claude Code의 Rust 오픈소스 클론)를 빌드하고, OpenAI의 모델을 백엔드로 사용하고자 함.
코드 분석 결과, 이미 OpenAI 호환 프로바이더가 구현되어 있어 코드 수정 없이 환경변수 설정만으로 가능.

## 실행 계획

### Step 1: Rust 툴체인 확인 및 설치
- `rustc --version` 확인, 없으면 rustup 설치 안내

### Step 2: Claw Code 빌드
```bash
cd /tmp/claw-code/rust
cargo build --workspace --release
```
- 바이너리: `./target/release/claw`

### Step 3: OpenAI API 키 설정
```bash
export OPENAI_API_KEY="sk-..."
```

### Step 4: OpenAI 모델로 테스트
```bash
./target/release/claw --model gpt-4o prompt "hello, what model are you?"
```

### Step 5: (선택) 로컬 경로에 바이너리 복사
```bash
cp ./target/release/claw /Users/wooho/claw-code/claw
```

## 주요 파일
- `/tmp/claw-code/rust/crates/api/src/providers/mod.rs` - 프로바이더 감지, 모델 레지스트리
- `/tmp/claw-code/rust/crates/api/src/providers/openai_compat.rs` - OpenAI 호환 클라이언트
- `/tmp/claw-code/rust/crates/api/src/client.rs` - ProviderClient 디스패처

## 동작 원리
1. `--model gpt-4o` 지정 → 모델명에서 프로바이더 감지 → OpenAI 선택
2. `OPENAI_API_KEY` 환경변수로 인증
3. OpenAI `/v1/chat/completions` 엔드포인트 호출
4. OpenAI 응답을 Anthropic 포맷으로 변환하여 동일한 에이전트 기능 제공

## 검증
- `claw --model gpt-4o prompt "hello"` 실행하여 응답 확인
- `claw doctor` 로 상태 체크
