# Eywa 1.0 소스코드 (4/4)

Files: server/src/ssh.js, templates/bridge.py

### server/src/ssh.js
```
const { Client } = require('ssh2');
const fs = require('fs');

// SSH 연결 풀
const connectionPool = new Map();

/**
 * SSH 연결 설정 객체 생성
 */
function buildConfig(member) {
  const config = {
    host: member.host,
    port: member.port || 22,
    username: member.ssh_user,
    readyTimeout: 10000,
  };

  if (member.ssh_auth_type === 'password') {
    config.password = member.ssh_password;
  } else {
    const keyPath = member.ssh_key_path || `${process.env.HOME}/.ssh/id_rsa`;
    if (fs.existsSync(keyPath)) {
      config.privateKey = fs.readFileSync(keyPath);
    } else {
      throw new Error(`SSH 키 파일을 찾을 수 없습니다: ${keyPath}`);
    }
  }

  return config;
}

/**
 * SSH 연결
 */
function connect(member) {
  return new Promise((resolve, reject) => {
    // 기존 연결이 있으면 재사용
    const poolKey = `${member.host}:${member.port}:${member.ssh_user}`;
    const existing = connectionPool.get(poolKey);
    if (existing && existing._sock && !existing._sock.destroyed) {
      return resolve(existing);
    }

    const conn = new Client();
    let config;
    try {
      config = buildConfig(member);
    } catch (err) {
      return reject(err);
    }

    conn.on('ready', () => {
      connectionPool.set(poolKey, conn);
      resolve(conn);
    });

    conn.on('error', (err) => {
      connectionPool.delete(poolKey);
      reject(err);
    });

    conn.on('close', () => {
      connectionPool.delete(poolKey);
    });

    conn.connect(config);
  });
}

/**
 * SSH 명령 실행
 */
function exec(conn, command, options = {}) {
  return new Promise((resolve, reject) => {
    const timeout = options.timeout || 30000;
    let timer = setTimeout(() => {
      reject(new Error('SSH 명령 실행 타임아웃'));
    }, timeout);

    conn.exec(command, (err, stream) => {
      if (err) {
        clearTimeout(timer);
        return reject(err);
      }

      let stdout = '';
      let stderr = '';

      stream.on('close', (code) => {
        clearTimeout(timer);
        resolve({ stdout, stderr, code });
      });

      stream.on('data', (data) => {
        stdout += data.toString();
      });

      stream.stderr.on('data', (data) => {
        stderr += data.toString();
      });
    });
  });
}

/**
 * SSH 연결을 통한 실시간 로그 스트리밍
 */
function streamLogs(conn, logPath, onData, onError) {
  return new Promise((resolve, reject) => {
    conn.exec(`tail -f ${logPath} 2>/dev/null`, (err, stream) => {
      if (err) return reject(err);

      stream.on('data', (data) => {
        onData(data.toString());
      });

      stream.stderr.on('data', (data) => {
        if (onError) onError(data.toString());
      });

      stream.on('close', () => {
        resolve();
      });

      // stream 객체를 반환하여 외부에서 종료할 수 있게 함
      resolve(stream);
    });
  });
}

/**
 * SSH 연결 해제
 */
function disconnect(member) {
  const poolKey = `${member.host}:${member.port}:${member.ssh_user}`;
  const conn = connectionPool.get(poolKey);
  if (conn) {
    conn.end();
    connectionPool.delete(poolKey);
  }
}

/**
 * 모든 SSH 연결 종료
 */
function disconnectAll() {
  for (const [key, conn] of connectionPool) {
    conn.end();
  }
  connectionPool.clear();
}

module.exports = { connect, exec, disconnect, disconnectAll, streamLogs, connectionPool };

```

### templates/bridge.py
```
#!/usr/bin/env python3
"""
Claude Team Bridge -- Slack <-> Claude CLI
Usage: python3 bridge.py <agent-name>

Each agent (team-lead, project-leader, coder, reviewer, tester) runs
as a separate bridge.py process connected to its own Slack bot.
"""

import json
import logging
import os
import re
import signal
import subprocess
import sys
import threading
import time
import urllib.request
from datetime import datetime
from pathlib import Path

# ---------------------------------------------------------------------------
# Config
# ---------------------------------------------------------------------------

AGENT_CONFIG = {
    "team-lead": {
        "bot_token_key": "SLACK_TEAM_LEAD_BOT_TOKEN",
        "app_token_key": "SLACK_TEAM_LEAD_APP_TOKEN",
        "agent_file": "team-lead.md",
        "model": "sonnet",
    },
    "project-leader": {
        "bot_token_key": "SLACK_PM_BOT_TOKEN",
        "app_token_key": "SLACK_PM_APP_TOKEN",
        "agent_file": "project-leader.md",
        "model": "opus",
    },
    "coder": {
        "bot_token_key": "SLACK_CODER_BOT_TOKEN",
        "app_token_key": "SLACK_CODER_APP_TOKEN",
        "agent_file": "coder.md",
        "model": "sonnet",
    },
    "reviewer": {
        "bot_token_key": "SLACK_REVIEWER_BOT_TOKEN",
        "app_token_key": "SLACK_REVIEWER_APP_TOKEN",
        "agent_file": "reviewer.md",
        "model": "sonnet",
    },
    "tester": {
        "bot_token_key": "SLACK_TESTER_BOT_TOKEN",
        "app_token_key": "SLACK_TESTER_APP_TOKEN",
        "agent_file": "tester.md",
        "model": "sonnet",
    },
}

# Session refresh thresholds
SESSION_MAX_CALLS = 20
SESSION_MAX_TIME = 1800  # 30 minutes
SESSION_SLOW_THRESHOLD = 120  # seconds

# Paths
PROJECT_DIR = Path(__file__).resolve().parent.parent  # project root
AGENTS_DIR = PROJECT_DIR / ".claude" / "agents"
LOGS_DIR = Path.home() / "claude-team" / "logs"
SESSIONS_DIR = Path.home() / "claude-team" / "sessions"
DASHBOARD_URL = os.environ.get("DASHBOARD_URL", "http://localhost:5555")

# Thread session tracking
# {thread_ts: {"session_id": str, "call_count": int, "started_at": float, "last_response_time": float}}
thread_sessions: dict = {}
session_lock = threading.Lock()

# Global logger — initialised in setup_logging()
logger: logging.Logger = logging.getLogger("bridge")

# Bot user ID — populated after Slack app starts
BOT_USER_ID: str = ""

# Shutdown flag
_shutdown = threading.Event()


# ---------------------------------------------------------------------------
# 1. Logging
# ---------------------------------------------------------------------------

def setup_logging(agent_name: str) -> logging.Logger:
    """파일 + 콘솔 로깅 설정. ~/claude-team/logs/{agent}.log 에 기록."""
    LOGS_DIR.mkdir(parents=True, exist_ok=True)
    log_file = LOGS_DIR / f"{agent_name}.log"

    fmt = logging.Formatter("[%(asctime)s] [%(levelname)s] %(message)s", datefmt="%Y-%m-%d %H:%M:%S")

    fh = logging.FileHandler(log_file, encoding="utf-8")
    fh.setLevel(logging.DEBUG)
    fh.setFormatter(fmt)

    ch = logging.StreamHandler(sys.stderr)
    ch.setLevel(logging.INFO)
    ch.setFormatter(fmt)

    log = logging.getLogger("bridge")
    log.setLevel(logging.DEBUG)
    log.addHandler(fh)
    log.addHandler(ch)

    # Custom log levels for clarity in file logs
    logging.addLevelName(25, "CALL")
    logging.addLevelName(26, "RESPONSE")
    logging.addLevelName(27, "SESSION")

    log.info(f"로그 초기화 완료 — 파일: {log_file}")
    return log


# ---------------------------------------------------------------------------
# 2. Dashboard helper
# ---------------------------------------------------------------------------

def dashboard_post(path: str, data: dict) -> None:
    """Dashboard API 에 HTTP POST. 실패 시 무시."""
    try:
        url = f"{DASHBOARD_URL}{path}"
        payload = json.dumps(data).encode("utf-8")
        req = urllib.request.Request(url, data=payload, headers={"Content-Type": "application/json"}, method="POST")
        with urllib.request.urlopen(req, timeout=5):
            pass
    except Exception:
        pass  # 대시보드 연결 실패는 무시


# ---------------------------------------------------------------------------
# 3. Agent MD loader
# ---------------------------------------------------------------------------

def load_agent_md(agent_file: str) -> str:
    """
    .claude/agents/{file} 을 읽어서 시스템 프롬프트로 반환.
    YAML frontmatter (---...---) 는 제거하고 본문만 반환.
    """
    md_path = AGENTS_DIR / agent_file
    if not md_path.exists():
        logger.warning(f"에이전트 파일 없음: {md_path}")
        return ""
    raw = md_path.read_text(encoding="utf-8")
    # Strip YAML frontmatter
    stripped = re.sub(r"^---\n.*?\n---\n?", "", raw, count=1, flags=re.DOTALL)
    logger.info(f"에이전트 프롬프트 로드 완료: {md_path} ({len(stripped)}자)")
    return stripped.strip()


# ---------------------------------------------------------------------------
# 4. Session refresh check
# ---------------------------------------------------------------------------

def should_refresh_session(session_info: dict) -> str | None:
    """
    세션 갱신 필요 여부를 체크. 사유 문자열 반환, 불필요 시 None.
    """
    if session_info.get("call_count", 0) >= SESSION_MAX_CALLS:
        return f"호출 횟수 초과 ({session_info['call_count']}/{SESSION_MAX_CALLS})"
    elapsed = time.time() - session_info.get("started_at", time.time())
    if elapsed >= SESSION_MAX_TIME:
        return f"세션 시간 초과 ({int(elapsed)}s / {SESSION_MAX_TIME}s)"
    if session_info.get("last_response_time", 0) >= SESSION_SLOW_THRESHOLD:
        return f"응답 느림 ({session_info['last_response_time']:.1f}s >= {SESSION_SLOW_THRESHOLD}s)"
    return None


# ---------------------------------------------------------------------------
# 5. Session refresh
# ---------------------------------------------------------------------------

def refresh_session(agent_name: str, session_id: str, client, channel: str, thread_ts: str) -> str | None:
    """
    기존 세션을 요약 후 새 세션으로 교체.
    요약본을 저장하고 새 session_id 를 반환 (None 이면 실패).
    """
    logger.log(27, f"세션 갱신 시작 — agent={agent_name}, session={session_id}")
    client.chat_postMessage(
        channel=channel,
        thread_ts=thread_ts,
        text=":arrows_counterclockwise: 세션을 새로고침합니다. 잠시만 기다려주세요...",
    )

    # Ask Claude to summarize current context
    env = _build_env()
    summary_cmd = [
        "claude",
        "--resume", session_id,
        "--output-format", "json",
        "--dangerously-skip-permissions",
        "--debug",
        "-p",
        "지금까지의 대화와 작업 내용을 간결하게 요약해줘. 핵심 결정사항, 현재 상태, 남은 작업 위주로.",
    ]
    try:
        result = subprocess.run(summary_cmd, capture_output=True, text=True, env=env, timeout=120, cwd=str(PROJECT_DIR))
        parsed = json.loads(result.stdout)
        summary = parsed.get("result", result.stdout[:3000])
    except Exception as e:
        logger.error(f"세션 요약 실패: {e}")
        summary = "(요약 생성 실패 — 이전 세션 컨텍스트)"

    # Save summary
    save_session_md(agent_name, session_id, summary)

    logger.log(27, f"세션 갱신 완료 — 이전 session={session_id}")
    return summary


# ---------------------------------------------------------------------------
# 6. Session MD saver
# ---------------------------------------------------------------------------

def save_session_md(agent_name: str, session_id: str, summary: str) -> Path:
    """세션 요약을 ~/claude-team/sessions/{agent}_{timestamp}.md 에 저장."""
    SESSIONS_DIR.mkdir(parents=True, exist_ok=True)
    ts = datetime.now().strftime("%Y%m%d_%H%M%S")
    filepath = SESSIONS_DIR / f"{agent_name}_{ts}.md"
    content = f"# Session Summary\n\n- **Agent**: {agent_name}\n- **Session ID**: {session_id}\n- **Saved at**: {ts}\n\n---\n\n{summary}\n"
    filepath.write_text(content, encoding="utf-8")
    logger.log(27, f"세션 요약 저장: {filepath}")
    return filepath


# ---------------------------------------------------------------------------
# 7a. OpenAI-compatible LLM call
# ---------------------------------------------------------------------------

AUTH_METHOD = os.getenv('AUTH_METHOD', 'oauth')  # oauth, api_key, openai


def call_openai_compatible(prompt: str, system_prompt: str = "") -> str:
    """OpenAI-compatible API (e.g. Ollama, vLLM) 호출. urllib만 사용."""
    endpoint = os.environ.get("OPENAI_ENDPOINT", "http://localhost:11434/v1")
    model = os.environ.get("OPENAI_MODEL", "llama3")
    api_key = os.environ.get("OPENAI_API_KEY", "")

    url = f"{endpoint.rstrip('/')}/chat/completions"
    messages = []
    if system_prompt:
        messages.append({"role": "system", "content": system_prompt})
    messages.append({"role": "user", "content": prompt})

    body = json.dumps({"model": model, "messages": messages}).encode("utf-8")
    headers = {"Content-Type": "application/json"}
    if api_key:
        headers["Authorization"] = f"Bearer {api_key}"

    req = urllib.request.Request(url, data=body, headers=headers, method="POST")
    try:
        with urllib.request.urlopen(req, timeout=300) as resp:
            data = json.loads(resp.read().decode("utf-8"))
            return data["choices"][0]["message"]["content"]
    except Exception as e:
        logger.error(f"OpenAI-compatible API 호출 실패: {e}")
        return f"(OpenAI API 오류: {e})"


# ---------------------------------------------------------------------------
# 7. Core: call Claude and respond
# ---------------------------------------------------------------------------

def _build_env() -> dict:
    """subprocess 용 환경변수. CLAUDECODE / CLAUDE_CODE_ENTRYPOINT 제거."""
    env = os.environ.copy()
    env.pop("CLAUDECODE", None)
    env.pop("CLAUDE_CODE_ENTRYPOINT", None)
    return env


def _split_message(text: str, limit: int = 3900) -> list[str]:
    """긴 메시지를 Slack 글자 수 제한에 맞게 분할."""
    if len(text) <= limit:
        return [text]
    parts = []
    while text:
        if len(text) <= limit:
            parts.append(text)
            break
        # Try to split at newline
        idx = text.rfind("\n", 0, limit)
        if idx == -1:
            idx = limit
        parts.append(text[:idx])
        text = text[idx:].lstrip("\n")
    return parts


def call_claude_and_respond(
    agent_name: str,
    config: dict,
    prompt: str,
    client,
    channel: str,
    thread_ts: str,
    event_ts: str,
) -> None:
    """Claude CLI 호출 후 결과를 Slack 스레드에 응답."""
    logger.log(25, f"[{agent_name}] 호출 시작 — channel={channel} thread={thread_ts}")
    logger.log(25, f"[{agent_name}] 프롬프트: {prompt[:200]}{'...' if len(prompt) > 200 else ''}")

    # Typing indicator
    try:
        client.reactions_add(channel=channel, name="hourglass_flowing_sand", timestamp=event_ts)
    except Exception:
        pass

    env = _build_env()
    session_id = None
    is_resume = False
    summary_context = None

    with session_lock:
        session_info = thread_sessions.get(thread_ts)
        if session_info and session_info.get("session_id"):
            # Check refresh
            reason = should_refresh_session(session_info)
            if reason:
                logger.log(27, f"세션 갱신 필요: {reason}")
                summary_context = refresh_session(
                    agent_name, session_info["session_id"], client, channel, thread_ts
                )
                # Reset session — will start fresh with summary
                thread_sessions.pop(thread_ts, None)
                session_info = None
            else:
                session_id = session_info["session_id"]
                is_resume = True

    # OpenAI-compatible path — skip Claude CLI entirely
    if AUTH_METHOD == 'openai':
        start_time = time.time()
        try:
            system_prompt = load_agent_md(config["agent_file"])
            response_text = call_openai_compatible(prompt, system_prompt)
            elapsed = time.time() - start_time
            logger.log(26, f"[{agent_name}] OpenAI 완료 — {elapsed:.1f}s")

            if not response_text:
                response_text = "(빈 응답)"

            parts = _split_message(response_text)
            for i, part in enumerate(parts):
                client.chat_postMessage(
                    channel=channel, thread_ts=thread_ts, text=part,
                    unfurl_links=False, unfurl_media=False,
                )
                if i < len(parts) - 1:
                    time.sleep(0.3)

            dashboard_post("/api/activity", {
                "agent": agent_name, "channel": channel, "thread_ts": thread_ts,
                "elapsed": elapsed, "prompt_len": len(prompt),
                "response_len": len(response_text), "session_id": None, "call_count": 1,
            })
        except Exception as e:
            logger.error(f"[{agent_name}] OpenAI 예외: {e}", exc_info=True)
            client.chat_postMessage(
                channel=channel, thread_ts=thread_ts,
                text=f":x: OpenAI API 오류:\n```\n{str(e)[:1000]}\n```",
            )
        finally:
            _remove_reaction(client, channel, event_ts)
        return

    # Build command (oauth / api_key — both use Claude CLI)
    cmd = ["claude"]

    if is_resume and session_id:
        cmd += ["--resume", session_id]
    else:
        # Load agent system prompt
        system_prompt = load_agent_md(config["agent_file"])
        if summary_context:
            system_prompt += f"\n\n---\n\n# 이전 세션 요약\n\n{summary_context}"
        if system_prompt:
            cmd += ["--append-system-prompt", system_prompt]
        cmd += ["--model", config.get("model", "sonnet")]

    cmd += [
        "--output-format", "json",
        "--dangerously-skip-permissions",
        "--debug",
        "-p",
        prompt,
    ]

    logger.log(25, f"[{agent_name}] CLI 명령: {' '.join(cmd[:8])}...")

    # Execute
    start_time = time.time()
    try:
        result = subprocess.run(
            cmd,
            capture_output=True,
            text=True,
            env=env,
            timeout=600,
            cwd=str(PROJECT_DIR),
        )
        elapsed = time.time() - start_time
        logger.log(26, f"[{agent_name}] CLI 완료 — {elapsed:.1f}s, exit={result.returncode}")

        if result.returncode != 0 and not result.stdout.strip():
            error_msg = result.stderr.strip() or "(알 수 없는 오류)"
            logger.error(f"[{agent_name}] CLI 에러: {error_msg}")
            client.chat_postMessage(
                channel=channel,
                thread_ts=thread_ts,
                text=f":x: Claude CLI 오류:\n```\n{error_msg[:2000]}\n```",
            )
            _remove_reaction(client, channel, event_ts)
            return

        # Parse JSON output
        response_text = ""
        new_session_id = None
        try:
            parsed = json.loads(result.stdout)
            response_text = parsed.get("result", "")
            new_session_id = parsed.get("session_id", session_id)
        except json.JSONDecodeError:
            # Fallback: treat stdout as plain text
            response_text = result.stdout.strip()
            logger.warning(f"[{agent_name}] JSON 파싱 실패, 원본 텍스트 사용")

        if not response_text:
            response_text = "(빈 응답)"

        logger.log(26, f"[{agent_name}] 응답 ({len(response_text)}자): {response_text[:200]}...")

        # Update session tracking
        with session_lock:
            if new_session_id:
                info = thread_sessions.get(thread_ts, {
                    "session_id": new_session_id,
                    "call_count": 0,
                    "started_at": time.time(),
                    "last_response_time": 0,
                })
                info["session_id"] = new_session_id
                info["call_count"] = info.get("call_count", 0) + 1
                info["last_response_time"] = elapsed
                thread_sessions[thread_ts] = info

        # Post response (split if needed)
        parts = _split_message(response_text)
        for i, part in enumerate(parts):
            client.chat_postMessage(
                channel=channel,
                thread_ts=thread_ts,
                text=part,
                unfurl_links=False,
                unfurl_media=False,
            )
            if i < len(parts) - 1:
                time.sleep(0.3)  # rate limit courtesy

        # Dashboard update
        dashboard_post("/api/activity", {
            "agent": agent_name,
            "channel": channel,
            "thread_ts": thread_ts,
            "elapsed": elapsed,
            "prompt_len": len(prompt),
            "response_len": len(response_text),
            "session_id": new_session_id,
            "call_count": thread_sessions.get(thread_ts, {}).get("call_count", 1),
        })

    except subprocess.TimeoutExpired:
        elapsed = time.time() - start_time
        logger.error(f"[{agent_name}] CLI 타임아웃 — {elapsed:.1f}s")
        client.chat_postMessage(
            channel=channel,
            thread_ts=thread_ts,
            text=":warning: Claude CLI 응답 시간 초과 (10분). 다시 시도해 주세요.",
        )
    except Exception as e:
        logger.error(f"[{agent_name}] 예외 발생: {e}", exc_info=True)
        client.chat_postMessage(
            channel=channel,
            thread_ts=thread_ts,
            text=f":x: 내부 오류가 발생했습니다:\n```\n{str(e)[:1000]}\n```",
        )
    finally:
        _remove_reaction(client, channel, event_ts)


def _remove_reaction(client, channel: str, timestamp: str) -> None:
    """Remove hourglass reaction."""
    try:
        client.reactions_remove(channel=channel, name="hourglass_flowing_sand", timestamp=timestamp)
    except Exception:
        pass


# ---------------------------------------------------------------------------
# Session persistence — save/restore thread_sessions to JSON
# ---------------------------------------------------------------------------

def _threads_file(agent_name: str) -> Path:
    return SESSIONS_DIR / f"{agent_name}_threads.json"


def save_thread_sessions(agent_name: str) -> None:
    """thread_sessions 를 파일에 저장."""
    SESSIONS_DIR.mkdir(parents=True, exist_ok=True)
    filepath = _threads_file(agent_name)
    with session_lock:
        data = dict(thread_sessions)
    filepath.write_text(json.dumps(data, indent=2), encoding="utf-8")
    logger.log(27, f"스레드 세션 저장 완료: {filepath} ({len(data)}개)")


def restore_thread_sessions(agent_name: str) -> None:
    """이전 thread_sessions 를 파일에서 복원."""
    filepath = _threads_file(agent_name)
    if not filepath.exists():
        return
    try:
        data = json.loads(filepath.read_text(encoding="utf-8"))
        with session_lock:
            thread_sessions.update(data)
        logger.log(27, f"스레드 세션 복원 완료: {len(data)}개")
    except Exception as e:
        logger.warning(f"스레드 세션 복원 실패: {e}")


# ---------------------------------------------------------------------------
# 8. Main
# ---------------------------------------------------------------------------

def main() -> None:
    global BOT_USER_ID, logger

    # Parse args
    if len(sys.argv) < 2 or sys.argv[1] not in AGENT_CONFIG:
        print(f"Usage: python3 bridge.py <{'|'.join(AGENT_CONFIG.keys())}>")
        sys.exit(1)

    agent_name = sys.argv[1]
    config = AGENT_CONFIG[agent_name]

    # Logging
    logger = setup_logging(agent_name)
    logger.info(f"=== Claude Team Bridge 시작 — 에이전트: {agent_name} ===")

    # Load .env.claude-team
    env_file = PROJECT_DIR / ".env.claude-team"
    try:
        from dotenv import load_dotenv
        if env_file.exists():
            load_dotenv(env_file)
            logger.info(f"환경 변수 로드: {env_file}")
        else:
            logger.warning(f"환경 변수 파일 없음: {env_file}")
    except ImportError:
        logger.warning("python-dotenv 미설치 — .env 파일 무시. pip install python-dotenv")

    # Validate tokens
    bot_token = os.environ.get(config["bot_token_key"])
    app_token = os.environ.get(config["app_token_key"])
    if not bot_token or not app_token:
        logger.error(f"토큰 누락: {config['bot_token_key']} 또는 {config['app_token_key']} 를 설정하세요.")
        sys.exit(1)

    # Import Slack
    try:
        from slack_bolt import App
        from slack_bolt.adapter.socket_mode import SocketModeHandler
    except ImportError:
        logger.error("slack_bolt 미설치. pip install slack_bolt")
        sys.exit(1)

    # Restore sessions
    restore_thread_sessions(agent_name)

    # Init Slack app
    app = App(token=bot_token)

    # Get bot user ID
    try:
        auth = app.client.auth_test()
        BOT_USER_ID = auth.get("user_id", "")
        logger.info(f"Slack 인증 완료 — bot_user_id={BOT_USER_ID}")
    except Exception as e:
        logger.error(f"Slack 인증 실패: {e}")
        sys.exit(1)

    # Event handler
    @app.event("app_mention")
    def handle_mention(event, client, say):
        user = event.get("user", "")
        text = event.get("text", "")
        channel = event.get("channel", "")
        event_ts = event.get("ts", "")
        thread_ts = event.get("thread_ts", event_ts)

        # Ignore self-mentions
        if user == BOT_USER_ID:
            return

        # Clean mention text
        prompt = re.sub(r"<@[A-Z0-9]+>", "", text).strip()
        if not prompt:
            client.chat_postMessage(
                channel=channel,
                thread_ts=thread_ts,
                text="메시지를 입력해 주세요.",
            )
            return

        logger.info(f"멘션 수신 — user={user}, channel={channel}, thread={thread_ts}")

        # Run CLI call in background thread
        t = threading.Thread(
            target=call_claude_and_respond,
            args=(agent_name, config, prompt, client, channel, thread_ts, event_ts),
            daemon=True,
        )
        t.start()

    # Graceful shutdown
    def shutdown_handler(signum, frame):
        sig_name = signal.Signals(signum).name
        logger.info(f"종료 신호 수신: {sig_name}")
        save_thread_sessions(agent_name)
        dashboard_post("/api/status", {"agent": agent_name, "status": "offline"})
        logger.info("=== Claude Team Bridge 종료 ===")
        _shutdown.set()
        sys.exit(0)

    signal.signal(signal.SIGINT, shutdown_handler)
    signal.signal(signal.SIGTERM, shutdown_handler)

    # Dashboard: online
    dashboard_post("/api/status", {"agent": agent_name, "status": "online"})

    # Start Socket Mode
    logger.info("Socket Mode 시작...")
    handler = SocketModeHandler(app, app_token)
    try:
        handler.start()
    except KeyboardInterrupt:
        shutdown_handler(signal.SIGINT, None)


if __name__ == "__main__":
    main()

```