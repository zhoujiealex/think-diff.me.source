# think-diff.me.source


[![Build Status](https://travis-ci.com/zhoujiealex/think-diff.me.source.svg?token=9BJg6qgmazAszMMZZ7d6&branch=master)](https://travis-ci.com/zhoujiealex/think-diff.me.source)


think-diff.me personal site source code markdown file, theme, etc

I put source hexo blog codes here.

And using TravisCI to deploy github pages. please see http://www.think-diff.me/


# Note
For theme, it's a submodule.
Please use `git submodule update --init --recursive` to sync.

If fail, it may caused by network issue. Retry with:

``` bash
git ls-files --stage | grep 160000
git rm themes/next
git submodule  add git@github.com:zhoujiealex/hexo-theme-next.git  themes/next
```
