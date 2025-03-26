# Serv00-PM2-AutoRun

本项目借鉴了kuoihao仓库Serv00_Auto_Run的思路利用自动工作流进行远程运行脚本

原库地址https://github.com/kuoihao/Serv00_Auto_Run

## 3. 方法三：利用github的Actions中自动工作流脚本每5分钟check一下 https://memos.milaone.app 的运行状态，出错就ssh登录运行脚本

### 3.0 在Serv00中编写PM2恢复快照脚本
```
cd ~
touch run.sh
chmod +x run.sh


#run.sh中粘贴下面脚本

#!/bin/sh
~/.npm-global/bin/pm2 kill
/home/dino/.npm-global/bin/pm2 resurrect >/dev/null 2>&1
sleep 10
/home/dino/.npm-global/bin/pm2 restart all
~/.npm-global/bin/pm2 save

```
### 3.1 serv00生成密钥对，添加公钥到服务器、并且拿到私钥
serv00中运行，生成密钥对
```
ssh-keygen -t rsa -b 4096 -C "dino@milaone.app"
``` 
将公钥并添加到authorized_keys中
    
```
cat ~/.ssh/id_rsa.pub | tee -a ~/.ssh/authorized_keys
```

打印私钥内容，复制私钥全部内容
```
cat ~/.ssh/id_rsa
```
复制私钥内容作为SSH_PRIVATE_KEY的值,需要包括全部内容
```
-----BEGIN OPENSSH PRIVATE KEY-----
xxxxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxxxx
xxxxxxxxxxxxxxxxxxx
-----END OPENSSH PRIVATE KEY-----
```

- -----BEGIN OPENSSH PRIVATE KEY-----
- -----END OPENSSH PRIVATE KEY-----
这两行也要包含进来

### 3.2 fork本仓库到您的github账户下
### 3.3 配置密钥
在fork后的仓库中settings--> Security --> Secrets and variables --> Actions-->New repository secret按钮
```
Name：SSH_PRIVATE_KEY
Secret： 这里填刚复制出来的私钥
```
Add secret 按钮提交

### 3.4 修改仓库中
Serv00-PM2-Autorun/.github/workflows目录下的main.yml
其中需要修改的地方我都进行注释。
```
name: autorun

on:
  workflow_dispatch:
  schedule:
    # 5分钟运行一次
    - cron: '0/5 * * * *'
    

jobs:
  ssh-and-run:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout
      uses: actions/checkout@v2

    - name: Setup SSH
      uses: webfactory/ssh-agent@v0.5.3
      with:
        ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

    - name: Check if service is running
      id: check_service
      run: |
        if curl --head --silent --fail https://memos.milaone.app > /dev/null; then #这里改成你的Web服务的地址
          echo "service_status=running" >> $GITHUB_ENV
        else
          echo "service_status=not_running" >> $GITHUB_ENV
        fi

    - name: Run script on remote server if service is not running
      if: env.service_status == 'not_running'
      run: ssh -o StrictHostKeyChecking=no dino@s4.serv00.com "/home/dino/run.sh" #这里改成你的用户名@你的ssh服务器地址，以及/home/你的用户名/run.sh
```

### 3.5 打开Actions中的自动工作流即可
