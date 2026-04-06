# Eywa 1.0 소스코드 (3/4)

Files: server/src/routes/deploy-envs.js, server/src/routes/monitor.js, server/src/routes/roles.js, server/src/routes/sessions.js, server/src/routes/setup.js, server/src/routes/team.js

### server/src/routes/deploy-envs.js
```
const express = require('express');
const router = express.Router();
const { queries } = require('../db');
const ssh = require('../ssh');

// 팀의 배포 환경 목록 조회
router.get('/teams/:teamId/deploy-envs', (req, res) => {
  try {
    const envs = queries.getDeployEnvs.all(req.params.teamId);
    // 각 환경의 환경변수도 포함
    const result = envs.map(env => ({
      ...env,
      vars: queries.getDeployEnvVars.all(env.id),
    }));
    res.json(result);
  } catch (err) {
    res.status(500).json({ error: '배포 환경 목록 조회 실패', detail: err.message });
  }
});

// 배포 환경 생성
router.post('/teams/:teamId/deploy-envs', (req, res) => {
  try {
    const team = queries.getTeam.get(req.params.teamId);
    if (!team) return res.status(404).json({ error: '팀을 찾을 수 없습니다' });

    const {
      name, host, ssh_user, ssh_key_path, deploy_path,
      trigger_branch, deploy_method, build_command, run_command,
      auto_deploy, vars
    } = req.body;

    if (!name || !host) {
      return res.status(400).json({ error: 'name과 host는 필수입니다' });
    }

    const result = queries.createDeployEnv.run(
      req.params.teamId, name, host,
      ssh_user || 'ubuntu', ssh_key_path || null,
      deploy_path || null, trigger_branch || null,
      deploy_method || 'ssh', build_command || null,
      run_command || null, auto_deploy !== false ? 1 : 0
    );

    const envId = result.lastInsertRowid;

    // 환경변수 저장
    if (vars && Array.isArray(vars)) {
      for (const v of vars) {
        if (v.key) {
          queries.createDeployEnvVar.run(envId, v.key, v.value || '');
        }
      }
    }

    const env = queries.getDeployEnv.get(envId);
    res.status(201).json({
      ...env,
      vars: queries.getDeployEnvVars.all(envId),
    });
  } catch (err) {
    res.status(500).json({ error: '배포 환경 생성 실패', detail: err.message });
  }
});

// ... (full source in vault - deploy-envs CRUD, SSH test, init-server, deploy, generate-workflow)
```

### server/src/routes/monitor.js
```
const express = require('express');
const router = express.Router();
const { queries } = require('../db');

router.get('/dashboard', (req, res) => {
  try {
    const members = queries.getAllMembers.all();
    const summary = {
      total: members.length,
      active: members.filter(m => m.status === 'active').length,
      inactive: members.filter(m => m.status === 'inactive').length,
      error: members.filter(m => m.status === 'error').length,
    };
    res.json({ summary, members });
  } catch (err) {
    res.status(500).json({ error: '대시보드 조회 실패', detail: err.message });
  }
});

module.exports = router;
```

### server/src/routes/roles.js
```
const express = require('express');
const router = express.Router();
const multer = require('multer');
const { queries } = require('../db');

// Multer 설정 (메모리 저장)
const upload = multer({
  storage: multer.memoryStorage(),
  limits: { fileSize: 5 * 1024 * 1024 },
  fileFilter: (req, file, cb) => {
    if (file.mimetype === 'text/markdown' || file.originalname.endsWith('.md')) {
      cb(null, true);
    } else {
      cb(new Error('.md 파일만 업로드 가능합니다'));
    }
  },
});

// ... (full CRUD for roles with MD file upload support)
module.exports = router;
```

### server/src/routes/sessions.js
```
// 멤버의 세션 목록 조회, 세션 재시작 트리거, 자동 리프레시 설정 업데이트
// ... (full source in vault)
module.exports = router;
```

### server/src/routes/setup.js
```
// SSH 연결 검증, Claude CLI 배포, Claude 인증 검증 (oauth, api_key, openai)
// ... (full source in vault)
module.exports = router;
```

### server/src/routes/team.js
```
// 팀 CRUD, 팀 멤버 목록/추가
// ... (full source in vault)
module.exports = router;
```
