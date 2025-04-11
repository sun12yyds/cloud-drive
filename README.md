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

