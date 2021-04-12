---
title: hexo next 主题通过 travis ci 执行 hexo generate 报错
date: 2021-04-12 21:12:13
tags:
- travis
categories:
- troubleshooting
---

## 问题记录

前段时间给 github page 换了一个 hexo 主题，换成了 [next](https://theme-next.iissnan.com/) 主题，但是在更换主题之后，发现 travis 上的 CI 无法跑过了，查看日志内容如下:

```shell
$ node --version
v6.9.4
$ npm --version
3.10.10
$ nvm --version
0.37.2
before_install.1
0.01s$ export TZ='Asia/Shanghai'
before_install.2
6.90s$ npm install -g hexo
before_install.3
2.81s$ npm install -g hexo-cli
install
1.54s$ npm install
before_script.1
0.01s$ git config --global user.name "Win-Man"
before_script.2
0.00s$ git config --global user.email 825895587@qq.com
before_script.3
0.00s$ sed -i'' "s~git@github.com:Win-Man/win-man.github.io.git~https://${CI_TOKEN}@github.com/Win-Man/win-man.github.io.git~" _config.yml
0.67s$ hexo clean
The command "hexo clean" exited with 0.
1.17s$ hexo generate
INFO  Start processing
FATAL Something's wrong. Maybe you can find the solution here: https://hexo.io/docs/troubleshooting.html
TypeError: Object.values is not a function
    at points.views.forEach.type (/home/travis/build/Win-Man/win-man.github.io/themes/next/scripts/events/lib/injects.js:82:46)
    at Array.forEach (native)
    at module.exports.hexo (/home/travis/build/Win-Man/win-man.github.io/themes/next/scripts/events/lib/injects.js:67:16)
    at Hexo.hexo.on (/home/travis/build/Win-Man/win-man.github.io/themes/next/scripts/events/index.js:9:27)
    at emitNone (events.js:86:13)
    at Hexo.emit (events.js:185:7)
    at Hexo._generate (/home/travis/build/Win-Man/win-man.github.io/node_modules/hexo/lib/hexo/index.js:399:8)
    at loadDatabase.then.then (/home/travis/build/Win-Man/win-man.github.io/node_modules/hexo/lib/hexo/index.js:249:22)
    at tryCatcher (/home/travis/build/Win-Man/win-man.github.io/node_modules/bluebird/js/release/util.js:16:23)
    at Promise._settlePromiseFromHandler (/home/travis/build/Win-Man/win-man.github.io/node_modules/bluebird/js/release/promise.js:547:31)
    at Promise._settlePromise (/home/travis/build/Win-Man/win-man.github.io/node_modules/bluebird/js/release/promise.js:604:18)
    at Promise._settlePromise0 (/home/travis/build/Win-Man/win-man.github.io/node_modules/bluebird/js/release/promise.js:649:10)
    at Promise._settlePromises (/home/travis/build/Win-Man/win-man.github.io/node_modules/bluebird/js/release/promise.js:729:18)
    at Promise._fulfill (/home/travis/build/Win-Man/win-man.github.io/node_modules/bluebird/js/release/promise.js:673:18)
    at PromiseArray._resolve (/home/travis/build/Win-Man/win-man.github.io/node_modules/bluebird/js/release/promise_array.js:127:19)
    at PromiseArray._promiseFulfilled (/home/travis/build/Win-Man/win-man.github.io/node_modules/bluebird/js/release/promise_array.js:145:14)
    at Promise._settlePromise (/home/travis/build/Win-Man/win-man.github.io/node_modules/bluebird/js/release/promise.js:609:26)
The command "hexo generate" exited with 2.
cache.2
store build cache
```

但是在我本地电脑执行 `hexo g`,`hexo s`,`hexo d` 都没啥问题，怀疑到了 node js 版本的问题，因为这个 repo 中的 `.travis.yml` 配置文件还是好几年前的了，于是尝试修改 node js 版本与本地版本一致。修改版本并提交之后还是有报错，但是报错信息变了。

```shell
$ node --version
v13.11.0
$ npm --version
6.13.7
$ nvm --version
0.37.2
before_install.1
0.01s$ export TZ='Asia/Shanghai'
before_install.2
5.40s$ npm install -g hexo
3.16s$ npm install -g hexo-cli
npm WARN optional SKIPPING OPTIONAL DEPENDENCY: fsevents@~2.3.1 (node_modules/hexo-cli/node_modules/chokidar/node_modules/fsevents):
npm WARN notsup SKIPPING OPTIONAL DEPENDENCY: Unsupported platform for fsevents@2.3.2: wanted {"os":"darwin","arch":"any"} (current: {"os":"linux","arch":"x64"})
npm ERR! code EEXIST
npm ERR! syscall symlink
npm ERR! path ../lib/node_modules/hexo-cli/bin/hexo
npm ERR! dest /home/travis/.nvm/versions/node/v13.11.0/bin/hexo
npm ERR! errno -17
npm ERR! EEXIST: file already exists, symlink '../lib/node_modules/hexo-cli/bin/hexo' -> '/home/travis/.nvm/versions/node/v13.11.0/bin/hexo'
npm ERR! File exists: /home/travis/.nvm/versions/node/v13.11.0/bin/hexo
npm ERR! Remove the existing file and try again, or run npm
npm ERR! with --force to overwrite files recklessly.
npm ERR! A complete log of this run can be found in:
npm ERR!     /home/travis/.npm/_logs/2021-04-12T06_49_56_362Z-debug.log
The command "npm install -g hexo-cli" failed and exited with 239 during .
```

根据错误信息查找到一个链接
https://segmentfault.com/a/1190000018759308

## 解决方案

参考链接中的结果方案在 `npm install` 中加上 `-f` 选项。
附上完整的 `.travis.yml` 内容:

```
language: node_js
node_js:
- '13.11.0'
branches:
  only:
  - dev
cache:
  directories:
  - node_modules
before_install:
- export TZ='Asia/Shanghai'
- npm install -g hexo
- npm install -g hexo-cli -f
before_script:
- git config --global user.name "Win-Man"
- git config --global user.email 825895587@qq.com
- sed -i'' "s~git@github.com:Win-Man/win-man.github.io.git~https://${CI_TOKEN}@github.com/Win-Man/win-man.github.io.git~" _config.yml
install:
- npm install -f
script:
- hexo clean
- hexo generate
after_success:
- hexo deploy
```