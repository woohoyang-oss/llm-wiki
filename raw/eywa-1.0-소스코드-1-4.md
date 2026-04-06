# Eywa 1.0 소스코드 (1/4)

Files: server/package.json, server/src/db.js, server/src/deployer.js

### server/package.json
```json
{
  "name": "claude-team-server",
  "version": "1.0.0",
  "description": "Claude Team Setup Wizard - Backend",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js",
    "dev": "node --watch src/index.js"
  },
  "dependencies": {
    "better-sqlite3": "^11.7.0",
    "cors": "^2.8.5",
    "dotenv": "^16.4.7",
    "express": "^4.21.1",
    "multer": "^1.4.5-lts.1",
    "node-fetch": "^3.3.2",
    "ssh2": "^1.16.0",
    "ws": "^8.18.0"
  }
}
```

### server/src/db.js
```javascript
const Database = require('better-sqlite3');
const path = require('path');

const DB_PATH = path.join(__dirname, '..', 'data', 'claude-team.db');

// WAL 모드 + foreign keys
db.pragma('journal_mode = WAL');
db.pragma('foreign_keys = ON');

// Tables: teams, roles, members, logs, sessions, deploy_envs, deploy_env_vars
// Migrations for new columns
// Builtin roles seed: Project Lead, Team Lead, Coder, Reviewer, Tester, DevOps, Documentation
// Query helpers for all CRUD operations

module.exports = { db, queries };
```

### server/src/deployer.js
```javascript
const ssh = require('./ssh');
const fs = require('fs');
const path = require('path');

async function deployToClient(member, team, role) {
  // 1. 원격 디렉토리 생성
  // 2. bridge.py 업로드
  // 2.5. Auth method별 자격증명 배포 (OAuth Keychain 주입)
  // 3. .env.claude-team 생성
  // 4. 역할 MD 파일 업로드
  // 5. Python 의존성 설치
  // 6. Akasha MCP 등록
  // 7. bridge.py 시작
  // 8. 프로세스 확인
}

async function stopBridge(member) { /* pkill bridge.py */ }
async function restartBridge(member, team, role) { /* stop + deploy */ }
async function getBridgeStatus(member) { /* ps aux check */ }

module.exports = { deployToClient, stopBridge, restartBridge, getBridgeStatus };
```
