# think-diff.me.source


[![Build Status](https://travis-ci.com/zhoujiealex/think-diff.me.source.svg?token=9BJg6qgmazAszMMZZ7d6&branch=master)](https://travis-ci.com/zhoujiealex/think-diff.me.source)


think-diff.me personal site source code markdown file, theme, etc

I put source hexo blog codes here.

And using TravisCI to deploy github pages. please see http://www.think-diff.me/

# Installation
Use git clone:

> git clone git@github.com:zhoujiealex/think-diff.me.source.git

Then sync theme submodule `next`:

> git submodule update --init --recursive

# Note
For **themes/next**, it's a submodule.
If fail, it may caused by network issue. Check [here](https://chrisjean.com/git-submodules-adding-using-removing-and-updating/) and [here](http://stackoverflow.com/questions/4185365/no-submodule-mapping-found-in-gitmodule-for-a-path-thats-not-a-submodule).

Retry with:
``` bash
git ls-files --stage | grep 160000
git rm themes/next
git submodule  add git@github.com:zhoujiealex/hexo-theme-next.git  themes/next
```

# Build
Install node modules:
> cnpm install

> cnpm install gulp -g

Sometimes can't find gulp, then run `cnpm install gulp` locally.

build static files:
> gulp build

start local webserver to test:
> hexo server -p 65432 --debug

# Test
Open [http://localhost:65432/](http://localhost:65432/) to check the blog.

# Push to github
add remote repo
``` bash
git remote -v
[optinal] git remote add origin git@github.com:zhoujiealex/think-diff.me.source.git
git push
```

# TODO
1. add `gulp server` - start server and open page using default browser

# References
1. [Github think diff source](https://github.com/zhoujiealex/think-diff.me.source)
2. [TravisCI com](https://travis-ci.com/zhoujiealex/think-diff.me.source)
3. [TravisCI org](https://travis-org.com/zhoujiealex)
4. [Github blog page](https://github.com/zhoujiealex/zhoujiealex.github.io)
5. [Markdown syntax](https://daringfireball.net/projects/markdown/syntax)
6. [my .travis.yml](https://github.com/zhoujiealex/think-diff.me.source/blob/master/.travis.yml)