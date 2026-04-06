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
```python
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
thread_sessions: dict = {}
session_lock = threading.Lock()

logger: logging.Logger = logging.getLogger("bridge")
BOT_USER_ID: str = ""
_shutdown = threading.Event()

# ... (full source in vault)
```
