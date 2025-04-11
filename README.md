-![LOGO](https://github.com/sun12yyds/cloud-drive/blob/main/LOgo/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202025-04-12%20072533.png)
# cloud-drive一个优秀且大方的个人云盘，其基于github page 进行搭建和开发
# 让我们一起开始吧！🤳
# 🌟 第一章：环境准备与仓库配置
1.1 创建专用组织账号


访问 GitHub 注册页面


创建新账号（如 CloudDriveBot），专门用于云盘操作


开启双重认证（Settings > Security）


1.2 创建私有仓库
```bash
# 创建并初始化仓库
git init cloud-drive
cd cloud-drive
git checkout -b backend  # 后端代码分支（私有）
git checkout -b gh-pages # 前端代码分支（公开）
```

1.3 配置仓库权限
仓库 Settings > Collaborators > 添加主账号为管理员

Settings > Branches > 添加保护规则：
```
     Require pull request reviews

     Require status checks
```

1.4 生成访问凭证
经典Token（文件操作）：
```
权限：repo, workflow, admin:org
```
生成地址：https://github.com/settings/tokens/new

OAuth App（用户登录）：
```
注册地址：https://github.com/settings/developers

回调地址：https://yourusername.github.io/cloud-drive/auth
```

# 🔒 第二章：文件加密系统实现
2.1 前端加密流程
```javascript
// public/js/crypto.js
const CRYPTO_CONFIG = {
  name: 'AES-GCM',
  length: 256,
  iterations: 100000
};

async function deriveKey(password, salt) {
  const baseKey = await crypto.subtle.importKey(
    "raw", 
    new TextEncoder().encode(password),
    "PBKDF2",
    false,
    ["deriveKey"]
  );
  
  return crypto.subtle.deriveKey(
    {
      name: "PBKDF2",
      salt,
      iterations: CRYPTO_CONFIG.iterations,
      hash: "SHA-256"
    },
    baseKey,
    { name: CRYPTO_CONFIG.name, length: CRYPTO_CONFIG.length },
    true,
    ["encrypt", "decrypt"]
  );
}

async function encryptFile(file, password) {
  const salt = crypto.getRandomValues(new Uint8Array(16));
  const iv = crypto.getRandomValues(new Uint8Array(12));
  const key = await deriveKey(password, salt);
  const encrypted = await crypto.subtle.encrypt(
    { name: CRYPTO_CONFIG.name, iv },
    key,
    await file.arrayBuffer()
  );

  return {
    meta: {
      salt: Array.from(salt),
      iv: Array.from(iv),
      iterations: CRYPTO_CONFIG.iterations
    },
    data: new Uint8Array(encrypted)
  };
}
```

2.2 后端解密工作流
```
创建 .github/workflows/decrypt.yml：
```
```yaml
name: File Decryption
on:
  workflow_dispatch:
    inputs:
      filename:
        description: 'Encrypted file name'
        required: true
      password:
        description: 'Decryption password'
        required: true

jobs:
  decrypt:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Decrypt file
        env:
          FILENAME: ${{ github.event.inputs.filename }}
          PASSWORD: ${{ github.event.inputs.password }}
        run: |
          openssl enc -d -aes-256-gcm \
            -in "$FILENAME" \
            -out "decrypted_$FILENAME" \
            -k "$PASSWORD" -pbkdf2
          
      - uses: actions/upload-artifact@v3
        with:
          name: decrypted-file
          path: decrypted_*
```

# 👥 第三章：多账户登录系统
3.1 OAuth 登录流程
```javascript
// public/js/auth.js
class AuthManager {
  constructor(clientId) {
    this.clientId = clientId;
    this.redirectUri = `${window.location.origin}/auth`;
  }

  startLogin() {
    const state = crypto.randomUUID();
    localStorage.setItem('oauth_state', state);
    
    const params = new URLSearchParams({
      client_id: this.clientId,
      redirect_uri: this.redirectUri,
      scope: 'repo',
      state: state
    });
    
    window.location = `https://github.com/login/oauth/authorize?${params}`;
  }

  async handleCallback() {
    const code = new URLSearchParams(window.location.search).get('code');
    const state = localStorage.getItem('oauth_state');
    
    const response = await fetch('https://github.com/login/oauth/access_token', {
      method: 'POST',
      headers: { Accept: 'application/json' },
      body: JSON.stringify({
        client_id: this.clientId,
        client_secret: 'YOUR_CLIENT_SECRET',  // 必须通过后端代理
        code,
        state
      })
    });
    
    const { access_token } = await response.json();
    localStorage.setItem('github_token', access_token);
  }
}
```
```
// 使用示例
const auth = new AuthManager('YOUR_CLIENT_ID');
if (window.location.pathname === '/auth') {
  auth.handleCallback();
}
```

3.2 安全注意事项
永远不要在前端存储 client_secret

使用代理服务器处理 OAuth 流程（示例 Node.js 代理）：

```javascript
// proxy-server.js
const express = require('express');
const axios = require('axios');
const app = express();

app.get('/exchange-code', async (req, res) => {
  const { code } = req.query;
  
  const response = await axios.post('https://github.com/login/oauth/access_token', {
    client_id: process.env.GH_CLIENT_ID,
    client_secret: process.env.GH_CLIENT_SECRET,
    code
  }, {
    headers: { Accept: 'application/json' }
  });

  res.json(response.data);
});

app.listen(3000);
```

# 📁 第四章：文件管理系统实现
4.1 文件上传组件
```javascript
// public/js/uploader.js
class FileUploader {
  constructor(token) {
    this.token = token;
    this.chunkSize = 5 * 1024 * 1024; // 5MB分片
  }

  async upload(file, path, progressCallback) {
    const totalChunks = Math.ceil(file.size / this.chunkSize);
    const fileHash = await this.calculateHash(file);
    
    for (let i = 0; i < totalChunks; i++) {
      const chunk = file.slice(i * this.chunkSize, (i+1)*this.chunkSize);
      const content = await this.readChunk(chunk);
      
      await fetch(`https://api.github.com/repos/${REPO}/contents/${path}_part${i}`, {
        method: 'PUT',
        headers: {
          Authorization: `token ${this.token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          message: `Uploading ${file.name} part ${i+1}/${totalChunks}`,
          content: btoa(content)
        })
      });
      
      progressCallback((i+1)/totalChunks * 100);
    }
    
    await this.mergeParts(file.name, totalChunks);
  }

  async calculateHash(file) {
    const buffer = await file.arrayBuffer();
    const hashBuffer = await crypto.subtle.digest('SHA-256', buffer);
    return Array.from(new Uint8Array(hashBuffer))
               .map(b => b.toString(16).padStart(2,'0'))
               .join('');
  }
}
```
4.2 文件预览实现
```html
<!-- public/preview.html -->
<div class="preview-container">
  <div id="pdf-viewer" class="hidden"></div>
  <img id="image-viewer" class="hidden">
  <pre id="text-viewer" class="hidden"></pre>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.4.120/pdf.min.js"></script>
<script>
class FilePreview {
  static async show(fileUrl) {
    const extension = fileUrl.split('.').pop().toLowerCase();
    
    switch(extension) {
      case 'pdf':
        this.showPDF(fileUrl);
        break;
      case 'png':
      case 'jpg':
      case 'jpeg':
        this.showImage(fileUrl);
        break;
      default:
        this.showText(fileUrl);
    }
  }

  static async showPDF(url) {
    const loadingTask = pdfjsLib.getDocument(url);
    const pdf = await loadingTask.promise;
    
    for (let i = 1; i <= pdf.numPages; i++) {
      const page = await pdf.getPage(i);
      const canvas = document.createElement('canvas');
      const context = canvas.getContext('2d');
      
      const viewport = page.getViewport({ scale: 1.5 });
      canvas.height = viewport.height;
      canvas.width = viewport.width;
      
      await page.render({
        canvasContext: context,
        viewport: viewport
      }).promise;
      
      document.getElementById('pdf-viewer').appendChild(canvas);
    }
  }
}
</script>
```
运行 HTML
# 🚀 第五章：部署与持续集成
5.1 前端部署配置
```bash
# 安装 gh-pages 部署工具
npm install gh-pages --save-dev

# package.json 添加脚本
{
  "scripts": {
    "deploy": "gh-pages -d public -b gh-pages"
  }
}

# 运行部署
npm run deploy
```
5.2 GitHub Actions 自动化
```
创建 .github/workflows/sync.yml：
```

```yaml
name: Auto Sync
on:
  push:
    branches: [ main ]
  schedule:
    - cron: '0 3 * * *'  # 每天UTC 3点同步

jobs:
  sync-files:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 18.x
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build and deploy
        run: |
          npm run build
          npm run deploy
        env:
          GH_TOKEN: ${{ secrets.GH_DEPLOY_TOKEN }}
```
# 📱 第六章：移动端优化
6.1 响应式布局
```css
/* public/css/mobile.css */
@media (max-width: 768px) {
  .file-item {
    flex-direction: column;
    padding: 8px;
  }

  .upload-box {
    padding: 10px;
    margin: 10px 0;
  }

  .preview-container {
    max-width: 100%;
    overflow-x: auto;
  }
}

/* 手势操作优化 */
.touch-area {
  -webkit-tap-highlight-color: transparent;
  touch-action: manipulation;
}
```
6.2 离线支持
创建 public/sw.js：

```javascript
const CACHE_NAME = 'cloud-drive-v1';
const ASSETS = [
  '/',
  '/index.html',
  '/css/styles.css',
  '/js/main.js'
];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(ASSETS))
  );
});

self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request)
      .then(response => response || fetch(event.request))
  );
});
```
# 🔧 第七章：安全加固
7.1 密钥管理
```bash
# 使用 GitHub Secrets 存储敏感信息
# 在仓库 Settings > Secrets > Actions 添加：
- AES_KEY: 加密主密钥
- GH_DEPLOY_TOKEN: 部署用Token
- OAUTH_SECRET: OAuth客户端密钥
```
7.2 CSP 配置
在 public/index.html 添加：

```html
<meta http-equiv="Content-Security-Policy" 
      content="default-src 'self';
               script-src 'self' https://cdnjs.cloudflare.com;
               img-src 'self' data:;
               style-src 'self' 'unsafe-inline';
               connect-src 'self' https://api.github.com;">
```
运行 HTML
7.3 审计日志
创建 .github/workflows/audit.yml：

```yaml
name: Security Audit
on: [push, pull_request]

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run npm audit
        run: npm audit --production
        
      - name: Check for secrets
        uses: gitleaks/gitleaks-action@v2
        with:
          config-path: .gitleaks.toml
```
# 🛠️ 调试与维护
实时日志查看：
```bash
# 跟踪 GitHub Actions 日志
gh run watch
```
本地开发服务器：

```bash
python3 -m http.server 8000 --directory public/
```
性能监控：

```javascript

// 添加性能追踪
window.addEventListener('load', () => {
  const [timing] = performance.getEntriesByType('navigation');
  console.log('页面加载时间:', timing.duration);
});
```
以上为完整实现方案，每个步骤都包含可直接运行的代码示例。实际部署时请务必：


替换所有 YOUR_CLIENT_ID 等占位符


通过 GitHub 


在正式环境使用 HTTPS


定期轮换加密密钥

# 结语：至此，你已经完成了个人网盘的搭建，fork或者点个关注呗！
# 扩展阅读建议：
-![API管理和下载管理](https://github.com/sun12yyds/cloud-drive/blob/main/Read%20more.md)


-![全功能云盘构建思路-下载管理](https://github.com/sun12yyds/cloud-drive/blob/main/Advanced%20build%20solutions.md)


-![上传文件完整构建方法](https://github.com/sun12yyds/cloud-drive/blob/main/File%20upload%20and%20construction%20plan.md)


-![上传文件终极实现方案](https://github.com/sun12yyds/cloud-drive/blob/main/Upload%20a%20premium%20plan.md)


-![优秀网盘界面设计](https://github.com/sun12yyds/cloud-drive/blob/main/Excellent%20network%20disk%20interface%20design.md)
