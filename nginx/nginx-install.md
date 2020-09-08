Mac 下安装 Nginx

1. 更新 brew
```shell
brew update
```

1.1 如果 brew update 执行后，终端长时间没有响应，多半是网络连接问题，可以设置清华镜像(https://mirrors.tuna.tsinghua.edu.cn/help/homebrew/)
```shell
BREW_REPO="https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/brew.git"

git -C "$(brew --repo homebrew/core)" remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-core.git
git -C "$(brew --repo homebrew/cask)" remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-cask.git
git -C "$(brew --repo homebrew/cask-fonts)" remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-cask-fonts.git
git -C "$(brew --repo homebrew/cask-drivers)" remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-cask-drivers.git

echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.tuna.tsinghua.edu.cn/homebrew-bottles' >> ~/.bash_profile
source ~/.bash_profile

# 更换后测试工作是否正常
brew update
```

2. 安装 Nginx
```shell
brew install nginx
```
Nginx 安装信息如下，包含了一些重要的路径：
```
Docroot is: /usr/local/var/www

The default port has been set in /usr/local/etc/nginx/nginx.conf to 8080 so that
nginx can run without sudo.

nginx will load all files in /usr/local/etc/nginx/servers/.

To have launchd start nginx now and restart at login:
  brew services start nginx
Or, if you don't want/need a background service you can just run:
  nginx
```

3. 运行 Nginx
```
nginx
```
打开 http://localhost:8080/ 查看 Nginx 欢迎页，如果启动失败，提示端口被占用，可以找到 Nginx 配置文件，修改端口号，配置文件地址为 /usr/local/etc/nginx/nginx.conf
> 注意：nginx命令执行成功后不会有任何提示，不要误以为执行失败了
