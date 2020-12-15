---
title: Travis-CI持续集成Hexo博客
date: 2020-12-15T12:15:39+08:00
tags: hexo
categories: blog
---
> Hexo is a fast, simple & powerful blog framework, powered by Node.js.

hexo不仅可以作为博客框架生成博客，更能够广泛地快捷生成任意静态页面。

# hexo quick start
1. 准备工作：安装nodejs
```bash
$ yum install -y nodejs
```

2. 安装hexo
```bash
$ npm install hexo-cli -g
```

3. 创建博客空目录，使用hexo初始化设置
```bash
$ mkdir -p blog && cd blog && hexo init .
```

博客目录必须为空，不能包含任何文件，否则hexo init会初始化失败

4. 启动服务器，查看博客是否初始化成功
```bash
$ hexo server
```

5. 若要创建新的博文
```bash
$ hexo new [layout] <title>
```

post模板是默认的布局，通过修改_config.yml中的default_layout项修改文章的默认布局。如果不想使用任何布局，可以在文章的[Front-matter](https://hexo.io/docs/front-matter#Layout)中disable layout。

使用不同模板创建的文章保存在source目录下不同的路径：
| layout | path |
|  ----  | ----  |
| post | source/_posts |
| page | source |
| draft	| source/_drafts |

默认情况下，会使用文章名字作为文件名称。也可以修改_config.yml中new_post_name项定义文件命名格式。

hexo默认安装了hexo-renderer-marked和hexo-renderer-ejs，因此hexo支持渲染markdown以及ejs格式的文本，若安装了hexo-renderer-pug，也支持使用pug模板进行文章[书写](https://hexo.io/docs/writing)
除了markdown语法格式之外，若想在文章中添加引用块、代码块、youtube视频，为了让这些内容也能够很好地渲染，hexo提供了标签插件[tag plugins](https://hexo.io/docs/tag-plugins)，无论使用markdown还是其他格式进行文章书写，标签插件的格式都是不变的。
在写文章的时候，往往会在文章中插入其他附件，如图片，为了让页面渲染更出色，有时候甚至会在文章中添加css、js文件。hexo中的资源（Assets）代表的就是source目录下非文章（non-post）文件。
为了区分每篇文章的资源，使得文章资源更为合理地管理。需要将_config.yml中的[post_asset_folder](https://hexo.io/docs/asset-folders.html)设置为true，如此，通过hexo new命令生成的每篇新文章就会创建同名的资源文件夹，将文章相关的资源都放到这个同名文件夹中。在文章中通过相对路劲即可高效地引用这些资源。

6. 使用markdown编辑好博文之后，生成静态页面
```bash
$ hexo generate
```

# hexo目录
初始化完成之后，hexo目录结构如下：
```bash
[root@localhost blog]# ls -alh
total 108K
drwxr-xr-x.   6 root root  168 Nov 20 22:46 .
dr-xr-x---.  30 root root 4.0K Nov 20 22:44 ..
-rw-r--r--.   1 root root 2.4K Nov 19 20:34 _config.yml
-rw-r--r--.   1 root root  21K Nov 20 03:37 db.json
-rw-r--r--.   1 root root   65 Nov 19 20:34 .gitignore
drwxr-xr-x. 164 root root 8.0K Nov 19 20:36 node_modules
-rw-r--r--.   1 root root  581 Nov 19 20:36 package.json
-rw-r--r--.   1 root root  55K Nov 19 20:36 package-lock.json
drwxr-xr-x.   2 root root   52 Nov 19 20:34 scaffolds
drwxr-xr-x.   3 root root   20 Nov 19 20:34 source
drwxr-xr-x.   3 root root   23 Nov 19 20:34 themes
```
各目录解释如下：
| 目录/文件 | 用途 |
|  ----  | ----  |
| node_modules(D) |	npm安装的各种依赖包 |
| scaffolds(D) | 模板文件夹，初始包括page、post、draft三种模板，模板定义了写作内容的布局 |
| source(D) |	生成静态页面的源文件，包括markdown文档、图片等 |
| themes(D) |	主题文件夹，themes目录下，每个主题都是一个文件夹，默认主题为landscape |
| _config.yml(T) | 博客配置文件
| package.json(T) | 应用程序信息，罗列了Hexo版本及其依赖程序的版本 |
| package-lock.json(T) | hexo init执行过程的npm install所生成，用以记录当前状态下实际安装的各个npm package的具体来源和版本号。https://docs.npmjs.com/cli/v6/configuring-npm/package-lock-json |

# hexo blog部署
hexo提供了便捷的一键部署命令：
```bash
$ hexo deploy
```
hexo将网站部署的目的地有多种，主要分为服务器和远端仓库。
远端仓库如github page，该仓库包含静态站点的HTML、CSS和js文件，并自动地对外发布网站。
> GitHub Pages 是一项静态站点托管服务，它直接从 GitHub 上的仓库获取 HTML、CSS 和 JavaScript 文件，（可选）通过构建过程运行文件，然后发布网站。

[Coding.net](http://coding.net/)、gitee、gitlab都有同样的服务，部署方式参考github page即可。另外一种方式，就是将静态站点部署到自己的服务器上，可以通过ftp、sftp等方式一键将静态站点发布到自己的服务器上，这样需要安装hexo-deployer-ftpsync、hexo-deployer-sftp插件，每种插件都有相关的部署配置。其他小众的部署方式参考：[One-Command Deployment](https://hexo.io/docs/one-command-deployment.html)

本文以将静态站点部署到github page为例。
1. 在github上创建带有自己用户名的代码仓：`<username>.github.io`
2. 安装hexo-deployer-git插件
```bash
$ npm install hexo-deployer-git --save
```
3. 配置_config.yml
```yaml
deploy:
  type: git
  repo: <repository url> # https://github.com/system-thoughts/system-thoughts.github.io.git
  branch: [branch]
  message: [message]
```
`deploy`的各项解释如下：
| 选项 | 描述 | 默认值 |
|  ----  | ----  | ---- |
| repo | 代码仓地址	| |
| branch |部署的分支名称 | master (GitHub)coding-pages (Coding.net) master (others) |
| message | 自定义部署提交的信息 | Site updated: 当前时间，格式：'YYYY-MM-DD HH:mm:ss' |
| token	| 验证代码仓的token	| |

关于hexo-deploy-git相关的配置选项更多的介绍可参考：[hexo-deployer-git](https://github.com/hexojs/hexo-deployer-git)。
此处，配置type和repo即可，其他项使用默认值。
4. 部署网站
```bash
$ hexo clean && hexo deploy
```
`hexo clean`用来清除缓存文件(db.json) 和已生成的静态文件 (public)。在某些情况（尤其是更换主题后），如果发现您对站点的更改无论如何也不生效，您可能需要运行该命令。一般情况下，不需要运行此命令。
`hexo deploy`命令就是部署静态站点，当然之前要使用hexo generate生成静态页面。可以将这两条命令简化为：
```bash
$ hexo d -g // 在部署之前生成一把
or
$ hexo g -d // 生成之后立马部署
or
$ hexo generate && hexo deploy
```
此时会提示你输入github的用户名和密码，除非你使用token或者ssh-key的方式认证。
此处简单配置下，git用户名和密码：
```bash
$ git config --global user.name "lvying"
$ git config --global user.email "lvying.system.thoughts@gmail.com"
```
生成sshkey就不介绍，此处做下测试，当前系统中的sshkey是否是自己的github账户：
```bash
[root@localhost blog]# ssh -T git@github.com
Warning: Permanently added the RSA host key for IP address '192.30.255.112' to the list of known hosts.
Hi **system-thoughts**! You've successfully authenticated, but GitHub does not provide shell access.
```
之前的sshkey因为不是我自己的github账户，因此我又重新生成了一个。
在_config.yml中我是用https的仓库地址，提交会遇到错误：
```bash
remote: Permission to system-thoughts/system-thoughts.github.io.git denied to **buweilv**.
fatal: unable to access 'https://github.com/system-thoughts/system-thoughts.github.io.git/': The requested URL returned error: 403
FATAL { err:
   { Error: Spawn failed
       at ChildProcess.task.on.code (/root/blog/node_modules/hexo-deployer-git/node_modules/hexo-util/lib/spawn.js:51:21)
       at ChildProcess.emit (events.js:198:13)
       at Process.ChildProcess._handle.onexit (internal/child_process.js:248:12) code: 128 } } 'Something\'s wrong. Maybe you can find the solution here: %s' '\u001b[4mhttps://hexo.io/docs/troubleshooting.html\u001b[24m'
```
这个是我另外一个github账户，不知道为何会默认识别使用这个账户，我猜想这个可能是因为缓存的缘故吧。将repo配置为ssh的仓库地址，便可以正确部署站点。


# Travis CI部署hexo blog
当前每次部署hexo blog，都需要手动输入`hexo g -d`生成静态页面，并部署到github page上。这需要在当前环境安装hexo及npm，部署到github page上之后，`<username>.github.io`代码仓托管的是静态页面，如果要在不同设备上进行写作，需要拷贝最新源文件。
为了能够在本地专注blog写作，不用担心环境配置、源文件托管等杂项，使用Travis CI部署hexo blog。

## 源文件git管理
`hexo generate`和`hexo deploy`会新增文件和目录，这些目录的解释如下：
| 目录/文件 | 用途 |
|  ----  | ----  |
| public(D)	| 由hexo generate生成的静态页面，hexo clean清除。Hexo 引入了差分机制，如果 public 目录存在，那么 hexo g 只会重新生成改动的文件。若要强制重新生成，使用hexo g -f |
| .deploy_git(D) | hexo-deployer-git通过在.deploy_git中生成站点并强制将其推送到_config.yml中配置的仓库来工作。 如果.deploy_git不存在，则将初始化存储库（git init）。 否则，将使用当前的仓库（及其提交历史）。 |

如此看来,public目录中保存的是hexo g本地生成的静态站点页面。.deploy_git实际上就是代码仓，跟踪的就是`<username>.github.io`代码仓。当执行hexo d时，会将public目录中的内容形成一次新的提交记录，提交到`<username>.github.io`代码仓中。
对比下.deploy_git与`<username>.github.io`下的历史记录，两者完全匹配：
{% asset_img deploy_git_log.png %}
{% asset_img github_history.png %}

`.deploy_git`是`<uername>.github.io`的本地仓库，其master分支追踪`<uername>.github.io`的本地分支。
在blog根目录中，在hexo init初始化时便存在.gitignore文件：
{% codeblock .gitignore %}
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/
{% endcodeblock %}
可见hexo有意要将blog也作为git repo，在此处已经为我们列好了忽略项，忽略相关环境配置文件。这样，hexo也有意保存两份git repo:
1. 在使用hexo d -g部署的时候，将<strong>.deploy_git</strong>中的静态文件推送到<uername>.github.io的master分支
2. 将**根目录下的源文件**提交到<uername>.github.io另一分支，提交会忽略静态文件和环境配置文件。我们将源文件提交到代码仓的src分支中。

在博客根目录下，创建git repo：
```bash
$ git init .
$ git add . -A && git commit -m "init blog"
$ git remote add origin git@github.com:system-thoughts/system-thoughts.github.io.git
$ git push origin master:src
```
当前，博文源文件已经可以同步，为了进一步方便博客自动化生成、部署，可以使用travis CI进行持续集成。

## Travis CI持续集成hexo blog
本地部署和travis CI部署博客的工作流程区分如图所示：
{% asset_img travis_ci.png %}
Travis CI会自动部署hexo依赖的环境，一旦更新源文件，便会触发Travis CI构建、部署blog。具体步骤如下：
1. 打开官方网站[https://travis-ci.com/](https://travis-ci.com/)，使用github账号进行登录，然后 Travis CI会列出你 Github上面所有的仓库，以及你所属于的组织。点击安装Travis CI github APP，随后选择Travis CI有权限访问账号下的所有代码仓。
{% asset_img travis_org.png %}
{% asset_img repository_access.png %}
2. 若要Travis CI能够构建项目，需要在代码仓根目录中添加.travis.yml配置文件。当代码仓中有新的提交时，Travis CI会根据yaml文件的配置进行构建、发布。
```bash
language: node_js
node_js:
- '12'
install:
- npm install
script:
- npm run clean
- npm run build
deploy:
  provider: pages
  skip_cleanup: true
  github_token: "$GITHUB_TOKEN"
  local_dir: public
  on:
    branch: master
  target_branch: gh-pages
env:
  global:
    secure: KbvbdGLy5/C3mB5dr+yCF5llN9cAS9Sy498mnz9iMf3ONgQ+oOKYVHQLdqX6eYcQv4KUCrdUQHmUCmADr5vFNEblioRqzjZ4Ftcrg+tR2JaaApxYvPN5nm8TngZIHfEw0KbF83jFFUACMIsPM+sxmFxVX06K7D25l9N27K844uWKlxIqH3g/JygXphuBPSgqSjQ0j+m7AZOADpN16SCHfOqm6b6QPjnbqkEIeqyg7FChbmlGEFuYb4C7kL1E80m0hD8j8NOxohnmHWHTDcoP8xYehPHRNQ2VvrwZSAKVUDFTmmaP2vDYetO6p9MnQafRdgVBwtx+IO0nU7eQMB4eislRWajwXidon+yd0sDiLQyQXhjcg7qqckdatJc+LrOufx/gdSnmOg0fNZB0cVM8F7Sn1rafj9Vv5js9L3/OFjnH0Sc3Aa31bvli+BthXBHH0mhnoFhFupMRDk301af904Nyx/UejzDs3zN+aFV7XedkGEnsFA8TImz7Y2NOamxUSp15cOAiKcuK8fYnEBJh93m98dVCvk9haiuD8FiwIbqNJ+yDYiGxOeriLRxVRrmYKbn3GHKW+RT7t18dljR6ffYf/38jlRbtUnruMChbQ3/gZRsuq30J4hOEDYKS82JU0q6VZKsokDWo1RjU4NlrLDiTzzUUvkNLxGbckQM8MCQ=
```
3. Travis CI将构建生成的静态页面部署到github page，需要推送代码到`<username>.github.io`代码仓的权限。Github提供了Personal Access Token赋予Travis CI权限。前往 Github 帐号 Settings 页面，在左侧选择Developer settings → Personal Access Token，然后在右侧面板点击 “Generate new token” 来新建一个 Token。需要注意的是，创建完的 Token 只有第一次可见，之后再访问就无法看见（只能看见他的名称），因此要保存好这个值。
之前生成 Personal Access Token时，只选择了以下权限：
{% asset_img token_permission.png %}
加密密钥：
```bash
$ yum -y install ruby ruby-devel rubygems rpm-build
$ gem install travis
$ travis login --pro --github-token '<Personal Access Token>'
$ travis encrypt --pro 'GITHUB_TOKEN=<Personal Access Token>' --add
```
这里要登录[https://api.travis-ci.com](https://api.travis-ci.com/) 而非使用`travis login  --github-token '<Personal Access Token>'`登录[https://api.travis-ci.org/](https://api.travis-ci.org/)。因为我们的工程在[https://api.travis-ci.com](https://api.travis-ci.com/) 上构建。后续使用travis encrypt加密的GITHUB_TOKEN环境变量应该在travis-ci.com中。travis encrpt中自行贴入`Personal Access Token`的明文。之后.travis.yml的内容会自动更新。
密钥添加成功之后，每次触发travis-ci构建都能观察到该环境变量：
{% asset_img token_env.png %}
4. 对源文件进行修改之后，提交即可触发Travis CI构建，若要忽略某次提交，即这次提交不要触发Travis CI构建发布hexo blog，travis若要忽略一次提交，即这次提交不会触发Travis-CI构建，可以在commit msg中加入`[ci skip]`或者`[skip-ci]`关键字：
```bash
git commit -m 'documentation update [ci skip]'
```
# Reference
[1] [https://neveryu.github.io/2019/02/05/travis-ci/](https://neveryu.github.io/2019/02/05/travis-ci/)
[2] [https://medium.com/starbugs/travis-ci-簡單事情就交給電腦去做之ci-cd-初體驗-讓-github-pages-自動更新-7647be30eb1c](https://medium.com/starbugs/travis-ci-%E7%B0%A1%E5%96%AE%E4%BA%8B%E6%83%85%E5%B0%B1%E4%BA%A4%E7%B5%A6%E9%9B%BB%E8%85%A6%E5%8E%BB%E5%81%9A%E4%B9%8Bci-cd-%E5%88%9D%E9%AB%94%E9%A9%97-%E8%AE%93-github-pages-%E8%87%AA%E5%8B%95%E6%9B%B4%E6%96%B0-7647be30eb1c)
[3] [https://segmentfault.com/a/1190000019067492](https://segmentfault.com/a/1190000019067492)
[4] [http://magicse7en.github.io/2016/03/27/travis-ci-auto-deploy-hexo-github/](http://magicse7en.github.io/2016/03/27/travis-ci-auto-deploy-hexo-github/)
[5] [https://anran758.github.io/blog/2020/06/08/github-travis-build/](https://anran758.github.io/blog/2020/06/08/github-travis-build/)
[6] https://hexo.io/docs/setup.html