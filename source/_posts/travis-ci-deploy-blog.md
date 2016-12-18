---
title: 用Travis CI自动部署Hexo博客
permalink_url_parent: 2016
permalink_url: 05/28/travis-ci-deploy-blog
comments: true
tags:
  - hexo
  - 持续集成
  - gulp
  - Node.js
categories: hexo
date: 2016-05-28 15:34:49
keywords:
    - CI
    - hexo
    - 持续集成
    - gulp
    - 博客优化
    - 压缩
---

![Travis Build Flow](/images/travis-ci-deploy-blog/build-flow.png)


通常更新一篇Hexo博客，基本流程是：

1. 本地新建post页面

        hexo n travis-ci-deploy-blog
        INFO  Created: e:\WORK\GitHub\think-diff.me.source\source\_posts\travis-ci-deploy-blog.md
2. 在文本编辑器里用markdown语法编辑新建页面
3. 本地生成public文件：`hexo g`
4. 启动本地测试web server：`hexo s -p 4000 --debug`
5. 浏览器打开`http://localhost:4000/`, 浏览生成文章
6. 如果满意，即可部署到Github存放page仓库里： `hexo d`

本文主要介绍如何利用TravisCI自动完成第3-6步，同时利用gulp脚本进行html/css/js/images的压缩。

<!-- more -->

# What is Travis CI?

{% cq %} ![Travis CI](/images/common/travis.png) {% endcq %}

CI(Continuous Integration)翻译为持续集成。Travis CI是一个提供持续集成功能的平台，在Github上，可以添加Travis CI，当有code push时候，会推送通知到Travis，根据设置的脚本运行指定任务。

目前有两个站点:

1.[Travis.org](https://travis-ci.org/) 对于所有public项目完全免费

2.[Travics.com](https://travis-ci.com/) 只针对private项目，提供更多一些额外功能，如cache，并行build个数

两个站点只能看到各自的项目，不能通用。

# Why we need Travis CI?
有人可能会有疑问: 在本地写完博客，直接一个命令`hexo d`，不就搞定了么， 为啥要费力搞CI？

的确, 想用TravisCI来自动部署Hexo博客程序，需要不少设置（瞎折腾），为了给大伙信心，列举一些优点：

## 优点1：直接在线编辑文件，立即生效
假设你已经发表了一篇文章，过了几天你在朋友机器上浏览发现有几个明显的错别字，对于有强迫症的，这是不能容忍的。 但你手头又没有完整的hexo+nodejs+git的开发环境，重新下载git，node，hexo配置会花费不少时间，特别不划算。

如果按照这篇完整折腾完，你可以直接用浏览器访问github个人项目仓库，直接编辑那篇post的原md文件，前后2分钟改完。 稍等片刻，你的博客就自动更新了。

## 优点2：自动部署，同时部署到多个地方
在gitcafe是被收购之前，很多同学（包括我）都是托管在上面的，国内访问速度比Github快很多。
配合DNS根据IP位置可以自动选择导到gitcafe, 还是github，甚至你还可以部署到七牛云的静态网站。
利用Travis CI可同时更新多个仓库。

## 优点3：部署快捷方便
手动deploy需要推送public整个folder到github上，当后期网站文章、图片较多时候，对于天朝的网络，有时候连接github 就是不顺畅，经常要傻等不少上传时间。
有了CI，你可以只提交post文件里单独的md文件即可，很快很爽，谁用谁知道。

## 优点4：bigger than bigger<a id="build-icon"></a>
![build icon](/images/travis-ci-deploy-blog/passing.svg)

你的项目Readme里面可以显示CI build图标，很酷有没有？
另外通过设置，可以在当build失败时自动发邮件提醒你。
上面的图标，如果登陆后你在Github项目里，直接点击图标，会跳转到你当前项目build的log界面，很方便。

当然有了CI，你可以做很多事情，如自动运行单元测试，成功后再deploy等等。很多项目里的持续集成基本也是这个道理。

# How to use Travis CI to deploy hexo blog?
这篇教程是用的private项目演示的，多年前领取了Github的Education plan，给了我五个private项目。不管public或者private，步骤都是通用的。
我的博客项目设置的Travis脚本`.travis.yml`会做如下事情：

1. 从github clone整个项目
2. 在本地安装nodejs，hexo以及相关node modules
3. 运行gulp脚本 build文章，包括压缩css、html、js、image
4. 进入到public folder，初始化一个git仓库，然后推送到github page
我用的是user page，也可以是gh-pages，具体区别可参考[这里](https://help.github.com/articles/user-organization-and-project-pages/)

## 原理
对于public项目，Travis CI可以自由clone，也就是说有read权限，但是没有write权限，如果什么都不设置，是push不了code的。
所以最关键的一步是找到一个安全的方法，让TravisCI拿着Github的token去push code。

![Travis加密图解](/images/travis-ci-deploy-blog/travis-encrypt-keys.png)

用非对称加密key，简单说就是用[Travis public key加密](https://docs.travis-ci.com/user/encryption-keys/)，然后Travis用自己的private key去解密，然后使用。
至于加密的内容，你可以加密多种东西，以[Travis环境变量](https://docs.travis-ci.com/user/environment-variables/)的格式引用，下文详述。

参考了如下几篇文章：

1.[用 Travis CI 自動部署網站到 GitHub](https://zespia.tw/blog/2015/01/21/continuous-deployment-to-github-with-travis/ ) 作者是hexo的发明者[@tommy351](https://github.com/tommy351)，他的做法是用travis client加密ssh key，然后travis解密拿到key去操作仓库代码。缺点：因为是给的访问github个人账户的ssh私钥，所以权限是full的，可以操作任何部分。

2.[When Hexo Meets GitHub Pages and Travis CI plus Raspberry Pi](http://changyuheng.me/2015/when-hexo-static-site-meets-github-pages-and-travis-ci/) 这篇文章讲了一个tricky的方法，github每个仓库可以单独设置访问ssh public key，然后把private key放在Travis的环境变量里面，但是因为key中间有特殊字符，不能直接使用，所以经过base64编码存放在Travis环境变量。然后再travis.yml设置base64 -d解码出来这个环境变量

3.[使用 Travis CI 自动更新 GitHub Pages](http://notes.iissnan.com/2016/publishing-github-pages-with-travis-ci/#comments) 这也是本文主要参考的一篇文章，上面图也是借用的这里的。原理：github提供了access token的方法来访问仓库，其实是OAuth2里面的token，每个token可以设置不同的[访问scope](https://developer.github.com/v3/oauth/#scopes)来控制访问范围。然后加密这个token存放到Travis环境变量里面，后面步骤类似。

## Step by Step

1. 准备Travis CI账号, 传送门：[public项目](https://travis-ci.org/), [private项目](https://travis-ci.com/), 在登陆成功后，可以看到自己的Github项目，然后把开关打开，会自动hook到Github。
2. 准备Github Personal Access Token。在Github的[setting页面](https://github.com/settings/profile)，左侧面板选择[Personal access tokens](https://github.com/settings/tokens), 右上角点击`Generate new token`。生成token时候需要确定访问scope，这里我们选择第一个`repo`即可，参见[完整scope介绍](https://developer.github.com/v3/oauth/#scopes)。**重要**：生成的token只有第一次可见，一定要保存下来备用。
![access token](/images/travis-ci-deploy-blog/access-token.png)
3. 准备[Travis命令行工具](https://github.com/travis-ci/travis.rb)，需要依赖ruby环境。对于Windows环境，可以使用这里的[安装包](http://rubyinstaller.org/downloads/)，安装完成后可用`ruby -v`检查。
安装命令行工具，参考[这里官方文档](https://github.com/travis-ci/travis.rb#installation)：

        $ gem install travis -v 1.8.2 --no-rdoc --no-ri

        Now make sure everything is working:
        $ travis version
        1.8.2

4. 加密第2步生成的token，**用Github的**用户名/密码登录这个命令行工具。

        $ travis login
        We need your GitHub login to identify you.
        This information will not be sent to Travis CI, only to GitHub.
        The password will not be displayed.

        Try running with --github-token or --auto if you don't want to enter your password anyway.

        Username: rkh
        Password: *******************

        Successfully logged in!
登陆成功后，开始加密，参考[这里](https://github.com/travis-ci/travis.rb#encrypt)：

        travis encrypt -r <github name>/<github repo> GH_Token=XXX

        #sample：
        travis encrypt -r zhoujiealex/think-diff.me.source GH_TOKEN=8440d51db2c46xxx
把输出的`secure:"xxxx"`保存，其中`GH_Token`是设置的Travis的环境变量名字，git push时候会用到。

5. 配置`.travis.yml`（如果没有，新建），参考[javascript的guide](https://docs.travis-ci.com/user/languages/javascript-with-nodejs)。最关键的是其中`env`的部分:

        env:
         global:
           - GIT_PAGE_REF: github.com/zhoujiealex/zhoujiealex.github.io.git
           - secure: "<第4步生成的加密key>"
这里需要**注意**的是：仓库地址，这个地址其实就是你github上存放静态博客最终文件的仓库地址，末尾加上`.git`。

6. 让Travis CI自动向上面设置的仓库地址提交code。这里我用了gulp脚本，来自动调用`hexo generate`，然后进行压缩，最后deploy。

7. [集成build icon](https://docs.travis-ci.com/user/status-images/)，在Travis CI控制台里，点击那个icon，选择markdown的样式，然后放到项目Readme里即可。

# 完整配置文件

完整`.travis.yml`，参考[这个gist](https://gist.github.com/zhoujiealex/787870ec64e774a1ac3d5e17e7e0b23c)(翻不了墙的[看这里](/files/travis-ci-deploy-blog/travis_yml.txt))，里面设置了邮件通知，启用node module cache等。主要是环境变量`GH_Token` 和 `GIT_PAGE_REF` 使用方式。

完整`gulpfile.js`，参考[这个gist](https://gist.github.com/zhoujiealex/4d926889b02b85d4d8d73f036ef728eb)(翻不了墙的[看这里](/files/travis-ci-deploy-blog/gulpfile_js.txt))。
唯一的魔法，就是其中调用Hexo API去generate public文件的部分，参考[Hexo API](https://hexo.io/api/), 语法是hexo使用的[bluebird](https://github.com/petkaantonov/bluebird)的Promise写法，感兴趣具体可自行谷歌。

我使用的[package.json](https://gist.github.com/zhoujiealex/3259ca61efdb12bbdf83aea99d9ff705)(翻不了墙的[看这里](/files/travis-ci-deploy-blog/package_json.txt))，如果想使用我的`gulpfile.js`需要用`npm install`安装里面的module。

配置完成后，以后当你本地编辑完md文件后，只需要运行`git push orgin master`，推送代码到Github就会触发自动Travis CI 自动生成build，部署新的文章。

最后上一张，点击上面[build那个icon](#build-icon)时候，看到的后台Travis CI的build log，如果出错就需要查看这里。

![travis ci build log](/images/travis-ci-deploy-blog/build-success.png)


# 后记
其实我很早就搞了这个域名，一直没有维护，当时2014年第一次看别人介绍Hexo部署个人博客，感觉很酷，就折腾了下， 托管在GitCafe上。前段时间，一直收到DNSPOD的报警，说网站访问不了了，过了好几天还没恢复，以往都是10分钟以内就能 恢复的。抽空看了下，才知道[GitCafe被Coding收购](http://www.zhihu.com/question/40942874)了。 没办法我就把站点搬回Github了，又瞎折腾了一堆主题和插件。

真的很佩服那些持续写博客记录，分享的人，自己也从中学到了很多知识。坚持写作不容易，费时费力。

最后希望这篇文章能帮助一些小伙伴少走些弯路。
