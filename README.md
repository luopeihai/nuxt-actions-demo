
**通过 github actions 持续集成发布 nuxt(vue)应用**

### [实例网站](http://nuxtdemo.wasaigutian.com/)

### 预期效果:本地通过tag,github自动编译代码,并通过ssh连接线上服务器进行自动部署

前置知识及涉及知识

* [GitHub Actions 入门教程](http://www.ruanyifeng.com/blog/2019/09/getting-started-with-github-actions.html)
* [nuxt 基础](https://www.nuxtjs.cn/)
* [Realworld 为提供给开发者通过不同语言实现的样例demo](https://github.com/gothinkster/realworld)

# github配置

### 生成可对仓库操作的token 步骤

1. 登录github --> 点击头像下拉选中Settings -->  Developer settings --> Personal access tokens --> Generate new token 
2. Note 输入框输入: token(自定义),权限选中第一个"repo",点击Generate token 生成token,***把生成token保存一份(生成的token只显示一次)***

###  在github上创建应用
创建栗子:[nuxt-actions-demo](https://github.com/luopeihai/nuxt-actions-demo)

### 生成持续集成需要的 secrets 信息(仓库推送的token和服务器ssh信息)
步骤:nuxt-actions-demo -->Settings -->Secrets --> New secret 
生成如下secrets
1. 生成TOKEN--> 名为:TOKEN,值为:第一步生成的token

2. 生成HOST(服务器地址)

3.生成PASSWORD(服务器ssh密码)

4.生成PORT(服务器ssh端口)

5.生成USERNAME(服务器ssh用户名)

# 项目配置

### 配置本地项目
1. 本地clone 下nuxt-actions-demo
2. 添加代码 并确 npm run build 和 npm run start 能运行nuxt 项目,这里需要先配置好web服务器,确保能运行node且外网可访问.

3.  根目录下创建 .github/workflows/main.yml, main.yml为github执行的脚本.也是自动部署的关键.文件需要已.yml格式, main.yml内容:
```
# 推送 tag触发脚本
name: Publish And Deploy nuxt Demo
on:
  push:
    tags:
      - "v*" 

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      # 下载源码
      - name: Checkout
        uses: actions/checkout@master

      # 打包构建
      - name: Build
        uses: actions/setup-node@master
      - run: npm install
      - run: npm run build
      - run: tar -zcvf release.tgz .nuxt static nuxt.config.js package.json package-lock.json pm2.config.json

      # 发布 Release
      - name: Create Release
        id: create_release
        uses: actions/create-release@master
        env:
          GITHUB_TOKEN: ${{ secrets.TOKEN }}
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ github.ref }}
          draft: false
          prerelease: false

      # 上传构建结果到 Release
      - name: Upload Release Asset
        id: upload-release-asset
        uses: actions/upload-release-asset@master
        env:
          GITHUB_TOKEN: ${{ secrets.TOKEN }}
        with:
          upload_url: ${{ steps.create_release.outputs.upload_url }}
          asset_path: ./release.tgz
          asset_name: release.tgz
          asset_content_type: application/x-tgz

      #shh 部署到服务器
      - name: Deploy
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          password: ${{ secrets.PASSWORD }}
          port: ${{ secrets.PORT }}
          script: |
            cd /var/www/node/nuxtdemo
            wget https://github.com/luopeihai/nuxt-actions-demo/releases/latest/download/release.tgz -O release.tgz
            tar zxvf release.tgz
            npm install --production
            pm2 reload pm2.config.json

```
其中 
```
script: | cd /var/www/node/nuxtdemo #为web服务器放项目地址
            wget https://github.com/luopeihai/nuxt-actions-demo/releases/latest/download/release.tgz -O release.tgz # github中 release地址
            tar zxvf release.tgz #解压
            npm install --production 
            pm2 reload pm2.config.json  #pm2 运行
```
提交代码
该github action只有当推送tag 时候才会触发.

### tag 推送 并查看
1. 提交 tag
```
git tag v0.1.0
git push origin v0.1.0
```

2.[查看部署情况](https://github.com/luopeihai/nuxt-actions-demo/actions),如果有报错查看build-and-deploy,报错信息可以在web服务器终端中输入运行,有利于查找原因


# 碰到的坑

在github actions 通过ssh 连接到服务器执行编译报错:
```
error: npm: command not found
```
感觉差异,服务器确认过安装npm,但是github actions报了这个找不到npm的指令,查资料发现 main.yml中
```
 # 部署到服务器
      - name: Deploy
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          password: ${{ secrets.PASSWORD }}
          port: ${{ secrets.PORT }}
          script: |
            cd /var/www/node/nuxtdemo
            wget https://github.com/luopeihai/nuxt-actions-demo/releases/latest/download/release.tgz -O release.tgz
            tar zxvf release.tgz
            npm install --production
            pm2 reload pm2.config.json
```
uses: appleboy/ssh-action@master 执行script时候是
```
sudo cd /var/www/...... npm install --production
```
在回到服务器sudo输入
```
sudo npm install --production
```
的确是 npm: command not found

***npm: command not found 原因:*** 服务端node环境 我是通过nvm 而nvm并不会把node环境安装到 /usr/local/bin/目录下,使得sudo找不到对应指令,最终创建软连接 解决
```
sudo ln -s "$NVM_DIR/versions/node/$(nvm version)/bin/node" "/usr/local/bin/node"
sudo ln -s "$NVM_DIR/versions/node/$(nvm version)/bin/npm" "/usr/local/bin/npm"
sudo ln -s "$NVM_DIR/versions/node/$(nvm version)/bin/pm2” "/usr/local/bin/pm2”
```













