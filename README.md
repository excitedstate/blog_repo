# 按照以下方法重装`hexo`环境

```shell
# # install node/npm in your machine
# curl -fsSL https://fnm.vercel.app/install | bash
# fnm use --install-if-missing 20

curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt update && sudo apt install -y nodejs
# # npm install (generate node_modules)
npm install 
git clone -b main https://github.com/anzhiyu-c/hexo-theme-anzhiyu.git themes/anzhiyu
npm install hexo-renderer-pug hexo-renderer-stylus --save
npm run build
sudo npm install -g hexo
```

# 更新博客

```shell
hexo clean && hexo generat
# # preview
hexo server
# # deploy
hexo deploy 
```