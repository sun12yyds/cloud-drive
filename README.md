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
```
OAuth App（用户登录）：
```
注册地址：https://github.com/settings/developers

回调地址：https://yourusername.github.io/cloud-drive/auth
```
