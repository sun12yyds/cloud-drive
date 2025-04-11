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

🔒 第二章：文件加密系统实现
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
