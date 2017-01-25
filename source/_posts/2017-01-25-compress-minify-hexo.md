---
title: 优化Hexo博客 - 压缩 HTML、CSS、JS、IMG 等
permalink_url_parent: articles
permalink_url: compress-minify-hexo
comments: true
tags:
  - Hexo
  - gulp
  - Node.js
categories: Blog
date: 2017-01-25 10:06:18
updated_time: 2017-01-25 10:06:18
keywords:
    - Hexo
    - 持续集成
    - gulp
    - 博客优化
    - 压缩js
    - minify css
    - 压缩图片
---

![compress](/images/compress-minify-hexo/compress.jpg)
通过压缩可以明显的减少静态资源的体积，加快响应速度。

半年前我在这篇文章 [用Travis CI自动部署Hexo博客](/2016/05/28/travis-ci-deploy-blog/#完整配置文件) 中，简单提到了配合Travis CI，在真正部署之前，
自动压缩图片、混淆CSS、JS等资源。有好多朋友还是来信问如何配置，这里再详细介绍下我的做法。

我使用了 gulp 来定义要执行的压缩任务，而执行真正压缩的是如下几个模块，见名知意：

- clean-css
- htmlclean
- htmlmin
- imagemin
- uglify-js

每个模块，都有对应的 gulp wrapper 模块。

<!-- more -->

# 使用方法

为了节省折腾时间（适合懒人），把使用方法放在最前面。

需要两个文件：

1. [package.json](/files/travis-ci-deploy-blog/package_json.txt)
2. [gulpfile.js](/files/travis-ci-deploy-blog/gulpfile_js.txt)

下载后，去掉后缀 txt，放到 Hexo 博客的顶层目录，例如我的：

1. karlzhou_com_source/package.json
2. karlzhou_com_source/gulpfile.js

执行：

``` bash
# 1. 安装依赖
npm install

# 2. 执行压缩
gulp build

# 3. 发布
hexo deploy
```

效果：

压缩前： public文件夹大小

| 文件  |压缩前 | 压缩后 | 压缩比率 |
| ----  |------| ------ | ------ |
| public | 4487KB | 3686 KB| 82.15% |
| public/css | 59KB |44KB | 74.58% |
|public/images| 870KB| 741KB|85.17%|
|public/js | 136KB|84KB| 61.76%|

输出示例：

``` bash
F:\KARL\Personal\Blog\karlzhou_com_source>gulp build
[10:34:48] Using gulpfile F:\KARL\Personal\Blog\karlzhou_com_source\gulpfile.js
[10:34:48] Starting 'build'...
[10:34:48] Starting 'clean'...
[10:34:48] Finished 'clean' after 125 ms
[10:34:48] Starting 'generate'...
INFO  Start processing

INFO  Files loaded in 800 ms
INFO  Generated: baidusitemap.xml
...
INFO  Generated: articles/compress-minify-hexo/index.html
...
INFO  151 files generated in 3.71 s
[10:34:54] Finished 'generate' after 5.3 s
[10:34:54] Starting 'compress'...
[10:34:54] Starting 'minify-html'...
[10:34:54] Starting 'minify-css'...
[10:34:54] Starting 'minify-js'...
[10:34:54] Starting 'minify-img-aggressive'...
...
[10:35:23] gulp-imagemin: Minified 39 images (saved 132 kB - 14.8%)
[10:35:23] Finished 'minify-img-aggressive' after 29 s
[10:35:23] Finished 'compress' after 29 s
[10:35:23] Finished 'build' after 35 s
```

# package.json 说明

介绍下我自己使用的Hexo模块

``` bash
{
  "name": "hexo-karlzhou",
  "version": "0.0.1",
  "private": true,
  "hexo": {
    "version": "3.2.2"
  },
  "dependencies": {
    "hexo": "^3.2.2",
    "hexo-autonofollow": "^1.0.1",      # 自动给外链添加 nofollow， SEO 友好
    "hexo-baidu-url-submit": "0.0.5",   # 主动推送文章链接到baidu，这里是仅仅用来生存url文件，详情见下文。
    "hexo-deployer-git": "^0.2.0",
    "hexo-generator-archive": "^0.1.4",
    "hexo-generator-baidu-sitemap": "^0.1.2", # baidu格式的站点图
    "hexo-generator-category": "^0.1.3",
    "hexo-generator-feed": "^1.2.0",
    "hexo-generator-index": "^0.2.0",
    "hexo-generator-search": "^1.0.3",
    "hexo-generator-sitemap": "^1.1.2",
    "hexo-generator-tag": "^0.2.0",
    "hexo-renderer-ejs": "^0.2.0",
    "hexo-renderer-marked": "^0.2.11",
    "hexo-renderer-stylus": "^0.3.1",
    "hexo-server": "^0.2.0"
  },
  "devDependencies": {
    "del": "^2.2.2",
    "gulp": "^3.9.1",
    "gulp-clean-css": "^2.3.2",     # 压缩css
    "gulp-htmlclean": "^2.7.8",     # 处理html
    "gulp-htmlmin": "^3.0.0",       # 压缩html
    "gulp-imagemin": "^3.1.1",      # 压缩图片
    "gulp-uglify": "^2.0.1",        # 处理js
    "run-sequence": "^1.2.2"        # 控制任务执行顺序
  }
} 
```

# gulp 任务

参考如下注释。
``` js
var gulp = require('gulp');
var minifycss = require('gulp-clean-css');
var uglify = require('gulp-uglify');
var htmlmin = require('gulp-htmlmin');
var htmlclean = require('gulp-htmlclean');
var imagemin = require('gulp-imagemin');
var del = require('del');
var runSequence = require('run-sequence');
var Hexo = require('hexo');

// 清除public文件夹
gulp.task('clean', function() {
    return del(['public/**/*']);
});

// 利用Hexo API 来生成博客内容， 效果和在命令行运行： hexo g 一样
// generate html with 'hexo generate'
var hexo = new Hexo(process.cwd(), {});
gulp.task('generate', function(cb) {
    hexo.init().then(function() {
        return hexo.call('generate', {
            watch: false
        });
    }).then(function() {
        return hexo.exit();
    }).then(function() {
        return cb()
    }).catch(function(err) {
        console.log(err);
        hexo.exit(err);
        return cb(err);
    })
})

// 压缩public目录下的所有css
gulp.task('minify-css', function() {
    return gulp.src('./public/**/*.css')
        .pipe(minifycss({
            compatibility: 'ie8'
        }))
        .pipe(gulp.dest('./public'));
});

// 压缩public目录下的所有html
gulp.task('minify-html', function() {
    return gulp.src('./public/**/*.html')
        .pipe(htmlclean())
        .pipe(htmlmin({
            removeComments: true,
            minifyJS: true,
            minifyCSS: true,
            minifyURLs: true,
        }))
        .pipe(gulp.dest('./public'))
});

// 压缩public目录下的所有js
gulp.task('minify-js', function() {
    return gulp.src('./public/**/*.js')
        .pipe(uglify())
        .pipe(gulp.dest('./public'));
});

// 压缩public目录下的所有img： 这个采用默认配置
gulp.task('minify-img', function() {
    return gulp.src('./public/images/**/*.*')
        .pipe(imagemin())
        .pipe(gulp.dest('./public/images'))
})

// 同上，压缩图片，这里采用了： 最大化压缩效果。
gulp.task('minify-img-aggressive', function() {
    return gulp.src('./public/images/**/*.*')
        .pipe(imagemin(
        [imagemin.gifsicle({'optimizationLevel': 3}), 
        imagemin.jpegtran({'progressive': true}), 
        imagemin.optipng({'optimizationLevel': 7}), 
        imagemin.svgo()],
        {'verbose': true}))
        .pipe(gulp.dest('./public/images'))
})

// 用run-sequence并发执行，同时处理html，css，js，img
gulp.task('compress', function(cb) {
    runSequence(['minify-html', 'minify-css', 'minify-js', 'minify-img-aggressive'], cb);
});

// 执行顺序： 清除public目录 -> 产生原始博客内容 -> 执行压缩混淆
gulp.task('build', function(cb) {
    runSequence('clean', 'generate', 'compress', cb)
});

gulp.task('default', ['build'])

```

# 百度主动推送

这个模块 `hexo-baidu-url-submit`，会在 public 下生成博客的url链接。

具体使用方法，参考 [https://github.com/huiwang/hexo-baidu-url-submit](https://github.com/huiwang/hexo-baidu-url-submit)。

需要在 顶层的 `_config.yml`， 配置下：

``` bash
# Baidu submit
baidu_url_submit:
  count: 10                     ## 比如3，代表提交最新的三个链接
  host: www.karlzhou.com        ## 在百度站长平台中注册的域名
  token: NO_NEED                ## 请注意这是您的秘钥， 请不要发布在公众仓库里! 我用了travis 所以用curl发送即可
  path: baidu_urls.txt

```

最终url文件位于： `./public/baidu_urls.txt`

这里是我用 Travis CI 来自动推送生成的 txt 文件， 并没有用这个模块自带的生成方式。

``` bash
# .travis.yml

after_success:
- cd public
- git init
- git config user.name "Karl Zhou"
- git config user.email "i@karlzhou.com"
- git add .
- git commit -m "Update docs"
- git push --force --quiet "https://${GH_TOKEN}@${GIT_PAGE_REF}" master:master
- curl -H 'Content-Type:text/plain' --data-binary @baidu_urls.txt "http://data.zz.baidu.com/urls?site=www.karlzhou.com&token=${baidu_submit_token}"
```

其中的环境变量 `baidu_submit_token`，是保存在 Travis CI里面的，安全！

baidu 推送 token 获取参考: [百度站长链接提交：主动推送（实时）](http://zhanzhang.baidu.com/linksubmit/index)。curl推送示例：
``` bash
将要提交的链接按照每行一条的格式写入一个文本文件中，命名此文件为urls.txt，然后进入该文件所在目录，执行如下命令：
curl -H 'Content-Type:text/plain' --data-binary @urls.txt "http://data.zz.baidu.com/urls?site=www.karlzhou.com&token=Aen1csVmYZo40Xi1"
```

# 参考链接

1. [Hexo API Execute Commands](https://hexo.io/api/#Execute-Commands)
2. [hexo-theme-zurb-foundation gulp.js 示例](https://github.com/chrisjlee/hexo-theme-zurb-foundation/blob/master/gulpfile.js)
3. [Gulp Delete files and folders](https://github.com/gulpjs/gulp/blob/master/docs/recipes/delete-files-folder.md)
