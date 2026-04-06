```python
"""
Claude Team Bridge — Slack ↔ Claude CLI 연결
각 봇이 멘션되면 Claude CLI를 호출하고 결과를 Slack에 전송합니다.
스레드 단위로 세션을 유지하여 대화 맥락이 이어집니다.

사용법:
  python3 bridge.py team-lead
  python3 bridge.py coder
  python3 bridge.py reviewer
  python3 bridge.py tester
"""

import json
import logging
import os
import re
import subprocess
import sys
import threading
import time
import urllib.request
from pathlib import Path

from dotenv import load_dotenv
from slack_sdk import WebClient

DASHBOARD_URL = os.environ.get("DASHBOARD_URL", "http://localhost:5555")


def dashboard_post(path, data):
    """대시보드 API에 업데이트 전송 (실패해도 무시)"""
    try:
        req = urllib.request.Request(
            f"{DASHBOARD_URL}{path}",
            data=json.dumps(data).encode(),
            headers={"Content-Type": "application/json"},
            method="POST",
        )
        urllib.request.urlopen(req, timeout=3)
    except Exception:
        pass


logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(name)s] %(message)s",
    datefmt="%H:%M:%S",
)
logger = logging.getLogger("bridge")

env_path = Path(__file__).parent.parent / ".env.claude-team"
load_dotenv(env_path)

AGENT_CONFIG = {
    "team-lead": {
        "bot_token_key": "SLACK_TEAM_LEAD_BOT_TOKEN",
        "app_token_key": "SLACK_TEAM_LEAD_APP_TOKEN",
        "agent_file": "team-lead.md",
        "model": "sonnet",
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
    "project-leader": {
        "bot_token_key": "SLACK_PM_BOT_TOKEN",
        "app_token_key": "SLACK_PM_APP_TOKEN",
        "agent_file": "project-leader.md",
        "model": "opus",
    },
}

PROJECT_DIR = Path(__file__).parent.parent
AGENTS_DIR = PROJECT_DIR / ".claude" / "agents"

thread_sessions = {}
thread_sessions_lock = threading.Lock()


def call_claude_and_respond(prompt, model, bot_token, channel, thread_ts, agent_file_name):
    agent_name = os.path.splitext(agent_file_name)[0]
    dashboard_post("/api/agent", {"name": agent_name, "status": "busy", "current_task": prompt[:50]})
    dashboard_post("/api/log", {"agent": agent_name, "message": f"작업 시작: {prompt[:60]}"})

    try:
        logger.info(f"Claude CLI 호출: {prompt[:50]}...")
        env = os.environ.copy()
        env.pop("CLAUDECODE", None)
        env.pop("CLAUDE_CODE_ENTRYPOINT", None)

        agent_file = AGENTS_DIR / agent_file_name
        system_prompt = None
        if agent_file.exists():
            with open(agent_file, "r") as f:
                content = f.read()
                parts = content.split("---", 2)
                system_prompt = parts[2].strip() if len(parts) >= 3 else content

        with thread_sessions_lock:
            session_id = thread_sessions.get(thread_ts)

        if session_id:
            cmd = [
                "claude", "-p", prompt,
                "--model", model,
                "--output-format", "json",
                "--dangerously-skip-permissions",
                "--resume", session_id,
            ]
        else:
            cmd = [
                "claude", "-p", prompt,
                "--model", model,
                "--output-format", "json",
                "--dangerously-skip-permissions",
            ]
            if system_prompt:
                cmd.extend(["--append-system-prompt", system_prompt])

        result = subprocess.run(
            cmd, stdin=subprocess.DEVNULL, stdout=subprocess.PIPE,
            stderr=subprocess.PIPE, text=True, timeout=600,
            cwd=str(PROJECT_DIR), env=env,
        )

        if result.returncode != 0:
            output = f"CLI 오류: {result.stderr.strip()[:500]}"
        else:
            try:
                data = json.loads(result.stdout)
                output = data.get("result", result.stdout.strip())
                new_session_id = data.get("session_id")
                if new_session_id:
                    with thread_sessions_lock:
                        thread_sessions[thread_ts] = new_session_id
            except json.JSONDecodeError:
                output = result.stdout.strip()
            if not output:
                output = "응답이 비어있습니다."

    except subprocess.TimeoutExpired:
        output = "작업이 5분을 초과했습니다."
    except Exception as e:
        output = f"오류: {str(e)[:200]}"

    try:
        client = WebClient(token=bot_token)
        if len(output) > 3900:
            chunks = [output[i:i+3900] for i in range(0, len(output), 3900)]
            for i, chunk in enumerate(chunks):
                client.chat_postMessage(
                    channel=channel, thread_ts=thread_ts,
                    text=f"({i+1}/{len(chunks)})\n{chunk}" if len(chunks) > 1 else chunk,
                )
                time.sleep(1)
        else:
            client.chat_postMessage(channel=channel, thread_ts=thread_ts, text=output)
        dashboard_post("/api/agent", {"name": agent_name, "status": "online", "current_task": ""})
        dashboard_post("/api/log", {"agent": agent_name, "message": f"작업 완료: {output[:60]}"})
    except Exception as e:
        logger.error(f"Slack 전송 실패: {e}")


def main(agent_name: str):
    if agent_name not in AGENT_CONFIG:
        print(f"알 수 없는 에이전트: {agent_name}")
        sys.exit(1)

    config = AGENT_CONFIG[agent_name]
    bot_token = os.environ[config["bot_token_key"]]
    app_token = os.environ[config["app_token_key"]]
    model = config["model"]
    agent_file_name = config["agent_file"]

    from slack_bolt import App
    from slack_bolt.adapter.socket_mode import SocketModeHandler

    app = App(token=bot_token)
    bot_user_id = app.client.auth_test()["user_id"]

    @app.event("app_mention")
    def handle_mention(event, say):
        user = event.get("user", "")
        text = event.get("text", "")
        channel = event.get("channel", "")
        thread_ts = event.get("thread_ts") or event.get("ts")
        if user == bot_user_id:
            return
        clean_text = re.sub(r"<@[A-Z0-9]+>", "", text).strip()
        if not clean_text:
            return
        t = threading.Thread(
            target=call_claude_and_respond,
            args=(clean_text, model, bot_token, channel, thread_ts, agent_file_name),
            daemon=True,
        )
        t.start()

    @app.event("message")
    def handle_message(event):
        pass

    handler = SocketModeHandler(app, app_token)
    handler.start()


if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("사용법: python3 bridge.py <agent-name>")
        sys.exit(1)
    main(sys.argv[1])
```

## 요약
- 280줄, Slack Socket Mode + Claude CLI 브릿지
- 5개 에이전트: team-lead, coder, reviewer, tester, project-leader
- 스레드별 세션 유지 (thread_ts → session_id)
- 대시보드 API 연동 (상태/로그)
- 타임아웃: 600초 (10분)
- 3900자 초과 시 분할 전송
