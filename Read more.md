# 以下是从GitHub 云盘下载更新方案，包含前端界面、API 调用和安全性处理：

# 📥 方法一：直接下载（公开文件）
1. 前端下载按钮实现
```javascript
// public/js/download.js
class FileDownloader {
  static async downloadFile(fileInfo) {
    if (fileInfo.encrypted) {
      return this._downloadEncrypted(fileInfo);
    } else {
      return this._downloadNormal(fileInfo);
    }
  }

  static _downloadNormal(fileInfo) {
    const link = document.createElement('a');
    link.href = fileInfo.download_url;
    link.download = fileInfo.name;
    link.target = '_blank';
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  }

  static async _downloadEncrypted(fileInfo) {
    // 获取加密文件内容
    const encryptedData = await fetch(fileInfo.download_url)
      .then(res => res.arrayBuffer());
    
    // 弹出密码输入框
    const password = prompt('请输入文件解密密码:');
    if (!password) return;

    // 解密文件
    const decrypted = await this._decryptFile(encryptedData, password, fileInfo.iv);
    
    // 创建下载链接
    const blob = new Blob([decrypted], { type: fileInfo.mimeType });
    const url = URL.createObjectURL(blob);
    this._downloadNormal({
      download_url: url,
      name: fileInfo.name.replace('.enc', '')
    });
    
    // 释放内存
    setTimeout(() => URL.revokeObjectURL(url), 1000);
  }

  static async _decryptFile(encryptedData, password, iv) {
    // 使用 Web Crypto API 解密
    const keyMaterial = await crypto.subtle.importKey(
      'raw',
      new TextEncoder().encode(password),
      { name: 'PBKDF2' },
      false,
      ['deriveKey']
    );
    
    const key = await crypto.subtle.deriveKey(
      {
        name: 'PBKDF2',
        salt: new TextEncoder().encode('StaticSalt'), // 生产环境应使用随机salt
        iterations: 100000,
        hash: 'SHA-256'
      },
      keyMaterial,
      { name: 'AES-GCM', length: 256 },
      false,
      ['decrypt']
    );
    
    return crypto.subtle.decrypt(
      { name: 'AES-GCM', iv: new Uint8Array(atob(iv).split(',')) },
      key,
      encryptedData
    );
  }
}
```
2. 文件列表渲染
```javascript
// 示例文件列表数据
const files = [
  {
    name: 'document.pdf',
    download_url: 'https://github.com/user/repo/raw/main/document.pdf',
    encrypted: false
  },
  {
    name: 'secret_file.enc',
    download_url: 'https://github.com/user/repo/raw/main/secret_file.enc',
    encrypted: true,
    iv: '1,2,3,4,5,6,7,8,9,10,11,12' // 加密时使用的初始化向量
  }
];

// 渲染下载按钮
function renderFileList() {
  const container = document.getElementById('file-list');
  
  files.forEach(file => {
    const item = document.createElement('div');
    item.className = 'file-item';
    
    const downloadBtn = document.createElement('button');
    downloadBtn.textContent = `下载 ${file.name}`;
    downloadBtn.onclick = () => FileDownloader.downloadFile(file);
    
    item.appendChild(downloadBtn);
    container.appendChild(item);
  });
}
```
# 🔐 方法二：通过 GitHub API 下载（私有仓库）
1. 获取文件内容 API
```javascript
async function getFileContent(token, repoPath) {
  const response = await fetch(
    `https://api.github.com/repos/${repoPath}`,
    {
      headers: {
        'Authorization': `token ${token}`,
        'Accept': 'application/vnd.github.v3.raw'
      }
    }
  );
  
  if (!response.ok) {
    throw new Error(`下载失败: ${response.status}`);
  }
  
  return response.blob();
}
```
2. 使用示例
```javascript
// 从用户登录获取的token
const userToken = localStorage.getItem('github_token'); 

// 下载私有文件
document.getElementById('download-btn').addEventListener('click', async () => {
  try {
    const blob = await getFileContent(
      userToken,
      'yourname/repo/main/private_file.txt'
    );
    
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'private_file.txt';
    a.click();
    
    setTimeout(() => URL.revokeObjectURL(url), 1000);
  } catch (error) {
    alert(error.message);
  }
});
```
# 🗂 方法三：批量下载（文件夹打包）
1. 创建服务器端打包脚本
```python
# download_server.py
from flask import Flask, send_file
import zipfile
import io
import requests

app = Flask(__name__)

@app.route('/download-folder')
def download_folder():
    # 获取文件列表
    files = [
        {'name': 'file1.txt', 'url': 'https://github.com/...'},
        {'name': 'file2.jpg', 'url': 'https://github.com/...'}
    ]
    
    # 创建内存中的ZIP文件
    zip_buffer = io.BytesIO()
    with zipfile.ZipFile(zip_buffer, 'w') as zip_file:
        for file in files:
            response = requests.get(file['url'])
            zip_file.writestr(file['name'], response.content)
    
    zip_buffer.seek(0)
    return send_file(
        zip_buffer,
        mimetype='application/zip',
        as_attachment=True,
        download_name='folder.zip'
    )

if __name__ == '__main__':
    app.run()
```
2. 前端调用
```javascript
// 下载整个文件夹
async function downloadFolder(folderId) {
  const response = await fetch(`/download-folder?id=${folderId}`);
  const blob = await response.blob();
  
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = 'files.zip';
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  
  setTimeout(() => URL.revokeObjectURL(url), 1000);
}
```
# 🛡 安全增强措施
下载限速

```javascript
// 限制下载速度 (KB/s)
async function downloadWithSpeedLimit(url, maxSpeed = 100) {
  const response = await fetch(url);
  const reader = response.body.getReader();
  let received = 0;
  let chunks = [];
  
  while(true) {
    const { done, value } = await reader.read();
    if (done) break;
    
    chunks.push(value);
    received += value.length;
    
    // 计算需要等待的时间
    const expectedTime = (received / (maxSpeed * 1024)) * 1000;
    const actualTime = performance.now() - startTime;
    if (actualTime < expectedTime) {
      await new Promise(r => setTimeout(r, expectedTime - actualTime));
    }
  }
  
  return new Blob(chunks);
}
```
下载次数限制

```python
# 服务器端计数
from collections import defaultdict
download_count = defaultdict(int)

@app.route('/download')
def download_file():
    file_id = request.args.get('id')
    if download_count[file_id] > 10:
        return "超过下载次数限制", 403
    
    download_count[file_id] += 1
    return send_file(...)
```
# 🌐 完整前端页面示例
```html
<!DOCTYPE html>
<html>
<head>
    <title>GitHub 云盘下载器</title>
    <style>
        .file-item {
            padding: 10px;
            margin: 5px;
            border: 1px solid #ddd;
            border-radius: 4px;
            display: flex;
            justify-content: space-between;
        }
        .progress-bar {
            height: 5px;
            background: #eee;
            margin-top: 5px;
        }
        .progress {
            height: 100%;
            background: #4CAF50;
            width: 0%;
        }
    </style>
</head>
<body>
    <h1>文件下载中心</h1>
    <div id="file-list"></div>

    <script src="download.js"></script>
    <script>
        // 模拟文件数据
        const files = [
            {
                name: "项目文档.pdf",
                download_url: "https://example.com/files/doc.pdf",
                size: "2.5MB",
                encrypted: false
            },
            {
                name: "机密数据.enc",
                download_url: "https://example.com/files/secret.enc",
                size: "1.8MB",
                encrypted: true,
                iv: "1,2,3,4,5,6,7,8,9,10,11,12"
            }
        ];

        // 渲染文件列表
        function renderFiles() {
            const container = document.getElementById('file-list');
            
            files.forEach(file => {
                const item = document.createElement('div');
                item.className = 'file-item';
                
                const info = document.createElement('span');
                info.textContent = `${file.name} (${file.size})`;
                
                const btn = document.createElement('button');
                btn.textContent = '下载';
                btn.onclick = () => {
                    FileDownloader.downloadFile(file)
                        .catch(err => alert(err.message));
                };
                
                item.appendChild(info);
                item.appendChild(btn);
                container.appendChild(item);
            });
        }

        // 初始化
        document.addEventListener('DOMContentLoaded', renderFiles);
    </script>
</body>
</html>
```
运行 HTML
# ⚠️ 重要注意事项
GitHub API 速率限制：

1.未认证用户：60 次/小时

2.认证用户：5000 次/小时

3.解决方案：使用 localStorage 缓存文件列表

4.大文件处理：

```javascript
// 分块下载大文件
async function downloadLargeFile(url, chunkSize = 5 * 1024 * 1024) {
  const fileSize = await getFileSize(url);
  let chunks = [];
  
  for (let start = 0; start < fileSize; start += chunkSize) {
    const end = Math.min(start + chunkSize - 1, fileSize - 1);
    const chunk = await fetch(url, {
      headers: { 'Range': `bytes=${start}-${end}` }
    });
    chunks.push(await chunk.arrayBuffer());
  }
  
  return new Blob(chunks);
}
```
移动端适配：

```javascript
// 检测移动端
if (/Mobi|Android/i.test(navigator.userAgent)) {
  alert('建议在WiFi环境下下载大文件');
}
```
# 至此，你已经学会了下载模块的管理
