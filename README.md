# claude-ray.github.io
my own homepage

### 安装
这里只列出 hexo 的简明搭建方法，不讲“是什么”和“为什么”，关于 hexo 的更多配置和用法请前往 [Hexo Docs](https://hexo.io/docs/)  了解。

1. 安装 Node.js 环境
2. 通过 Node 的 npm 安装 Hexo，命令行输入
```
$ npm install -g hexo-cli
```
3. 初始化
```
$ hexo init <folder> # 初始化项目名称
$ cd <folder>
$ npm install # 安装模块
```
4. 预览
```
$ hexo clean # 必要时，清除上次生成页面时造成的缓存
$ hexo g  # 生成静态页面
$ hexo s  # 在本地启动Hexo，可以在浏览器访问 localhost:4000 来预览
```
5. github 支持
首先在 github 按照 username.github.io 的格式新建仓库，
之后在项目根目录下执行
```
$ npm i hexo-deployer-git --save  
```
打开_config.yml，编辑deploy信息，注意替换用户名
```
deploy:
  type: git
  repo: git@github.com:yourusername/yourusername.github.io.git
  branch: master
```
配置完成后，只需执行
```
$ hexo d
```
即可将本地内容部署在 github 中，输入域名username.github.io进行访问
