# Eywa 1.0 소스코드 (2/4)

Files: server/src/index.js, server/src/log-streamer.js, server/src/monitor.js, server/src/routes/akasha.js

### server/src/index.js
```
require('dotenv').config();

const express = require('express');
const cors = require('cors');
const http = require('http');
const { WebSocketServer } = require('ws');
const url = require('url');

const { queries } = require('./db');
const ssh = require('./ssh');
const monitor = require('./monitor');
const logStreamer = require('./log-streamer');

// 라우터
const teamRoutes = require('./routes/team');
const roleRoutes = require('./routes/roles');
const monitorRoutes = require('./routes/monitor');
const setupRoutes = require('./routes/setup');
const sessionRoutes = require('./routes/sessions');
const deployEnvRoutes = require('./routes/deploy-envs');
const akashaRoutes = require('./routes/akasha');

const app = express();
const server = http.createServer(app);

// 미들웨어
app.use(cors());
app.use(express.json());

// API 라우트
app.use('/api/teams', teamRoutes);
app.use('/api/roles', roleRoutes);
app.use('/api/monitor', monitorRoutes);
app.use('/api/setup', setupRoutes);
app.use('/api', sessionRoutes);
app.use('/api', deployEnvRoutes);
app.use('/api/akasha', akashaRoutes);

// ... (full source: member CRUD, start/stop, WebSocket setup for logs/status, monitor start)

const PORT = process.env.PORT || 3001;
server.listen(PORT, () => {
  console.log(`[Server] Claude Team Setup Wizard 서버 시작: http://localhost:${PORT}`);
});
```

### server/src/log-streamer.js
```
const ssh = require('./ssh');

function streamBridgeLog(member, onData, onError) { /* tail -f bridge log */ }
function streamCliDebugLog(member, onData, onError) { /* tail -f latest debug log */ }
async function listDebugLogs(member) { /* ls -lt ~/.claude/debug/*.txt */ }
async function readDebugLog(member, filename) { /* cat specific debug log */ }

module.exports = { streamBridgeLog, streamCliDebugLog, listDebugLogs, readDebugLog };
```

### server/src/monitor.js
```
const ssh = require('./ssh');
const { queries } = require('./db');

let monitorInterval = null;
let wsBroadcast = null;

function start(broadcastFn) { /* 30초 간격 모니터링 */ }
function stop() { /* 모니터링 중지 */ }
async function checkAllMembers() { /* 전체 멤버 상태 확인 */ }
async function checkMember(member) { /* 개별 SSH 체크 */ }

module.exports = { start, stop, checkAllMembers, checkMember };
```

### server/src/routes/akasha.js
```
const express = require('express');
const router = express.Router();
const { queries } = require('../db');

const AKASHA_BASE = 'https://akaxa.space/api/v1';

// POST /auth - 이메일 + PIN 인증
// GET /list - Vault 목록 조회
// POST /load - Vault 로드
// POST /save - Vault 저장

module.exports = router;
```
