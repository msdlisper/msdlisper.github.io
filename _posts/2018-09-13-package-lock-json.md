---
layout: post
title:  "无辜的小明和package-lock.json"
date:   2018-09-13 09:19:00
categories: learn 
tags:  npm
---

* content
{:toc}

关于package-lock.json的总结以及避坑指南




## 背景

#### 无辜的小明的项目
小明负责一个新的项目, 项目大概是这样:

package.json:

`"react-scroll": "^1.5.2",`

package-lock.json :

```
"react-scroll": {
     "version": "1.6.7",
     "resolved": "https://registry.npmjs.org/react-scroll/-/react-scroll-1.6.7.tgz",
     ...
   },
```

然后小明就这样愉快的码代码....直到有一天, 新同事小王入职, npm install后, 发现代码崩溃, 怎么也运行不起来...小王心中默念MMP x 100次后, 找到小明: "你代码跑不起来呀? "小明把项目npm start后发现, 跑的起来了, 回答: '是你跑的姿势不对'...小王回去心中继续MMP....直到小王发现自己电脑的package-lock.json:
```
"react-scroll": {
     "version": "1.7.2",
     "resolved": "https://registry.npmjs.org/react-scroll/-/react-scroll-1.7.2.tgz",
     ...
   },
```
1.7.2版本的react-scroll与1.6.7出现了api不兼容的情况, 导致了代码崩溃.

小王开始怀疑人生, 思考是不是可以做点什么了...

#### 继承无辜的小明的项目
小明一如既往的愉快码代码....直到有一天, pm说, 我们项目上线吧, 于是小明在本地测妥了后就happly提上线单了, 于是, 线上就崩了, 小明百思不得其姐, 为啥本地打包处理没有问题, 线上就出问题了???? 被pm折磨后, 小明选择了回滚..


## npm@5的lockfile

回过头来看, 到底是个什么样的问题? 

由于我们前端的项目也加入了ci(continuous integration)的过程, 需要线上依赖包要和实际开发的包版本一致的问题, npm为了解决这个问题, 引入了lockfile, 常用的就是我们现在用的package-lock.json, 但我们package.json的依赖又是[semver-rang](https://semver.npmjs.com/)的版本, 类似(`"react-scroll": "^1.5.2"`), 在某些情况下导致了我们在reproducting时, 线上安装的依赖并没有按照我们package-lock.json里指定的版本和源来安装.

所以, 某些情况包括哪些情况?

- npm的版本
- package.json的依赖是否用了semver(有没有指定一个固定的版本)
- package的依赖有新版本发布了

上面小明的情况, 我是实际遇到过, 由于上线时间紧迫, 换用yarn来作为包管理器, 现在才有空总结下npm的问题, 但已经忘了是具体情况是啥了, 所以我做了个实验:

### 实验

实验用的包: 我在内网的registry里发布了testa, testb包, 然后packagetest依赖这两个包, 用于模拟新版的依赖发布, 并且把packagetest上传的gitlab上

packagetest的package.json 

```
"dependencies": {
  "@ddpay/testb": "^1.0.0"
}
```

testb的package.json

```
"dependencies": {
  "@ddpay/testa": "^1.0.0"
}
```

testa@1.0.0, testb@1.0.1, packagetest@1.0.0, 他们的关系如下:

```
    packagetest@1.0.0 /Users/didi/work/experiment/packageTest
    └─┬ @ddpay/testb@1.0.1
      └── @ddpay/testa@1.0.0
```

实验方式, 也就是验证的问题:

- 手动改写packagetest的package.json里的testb: 1.0.0, 在reinstall下, 会不会导致package-lock.json更新到testb@1.0.0(Y/N)
- package.json里 使用semver-rang的包有更新时, 会不会根据lock文件的内容下载
    + testb发布跟新, 在新clone的packagetest里`npm i`, package-lock.json会不会更新(Y/N)
    + testa发布更新, 在新clone的packagetest里`npm i`, package-lock.json会不会更新(Y/N)
- 使用不同版本的npm来验证 是否改变package-lock.json

包管理器版本 | 修改了package.json | 依赖有新版本
---|---|---
npm@5.0.0 | N | N
npm@5.1.0 | Y | N
npm@5.3.0 | N | N
npm@5.4.2 | N | N
npm@5.5.1 | Y | N
yarn@1.6.0 | Y | N

![黑人](https://ws1.sinaimg.cn/large/6af89bc8gw1f8t4h2t0l1g206404kqcy.gif)

不是说好的bug呢? 为啥依赖有新新版本的情况, 全部都锁住了! 就像生活无情的给我了一巴掌一样

![反杀](http://ww1.sinaimg.cn/mw690/ba0e41a3ly1fv7qq74gedj20t40qsk27.jpg)

反正我这次没复现出来, 但并不代表这个问题不存在, 不然github上那么多issus都是在交流感情吗?

### 网上的讨论

[issue17979](https://github.com/npm/npm/issues/17979)

npm@5.3.0没有reinstall时, 无视lock文件
>Example:

>package.json:
"Package-A": "^v1.0.0"

>package-lock.json:
"Package-A": { version: 1.0.0 }

>When I have no node_modules folder and I attempt to do a fresh npm install, previously in npm version 5.0.3 this would install version 1.0.0 (as this is what the lock file states). However, now on npm version 5.3.0, a fresh install will cause any version from the range ^v1.0.0 to be installed, completely ignoring the lock file.
> 

各个大佬(npm的开发者)的回答, 都一口咬定不会有这个问题, 看的出有激烈的讨论:

>The following should modify the lockfile: npm update, npm i packageName, npm i but only if you manually edited package.json. That is, the only time package-lock is updated on npm i is if the lockfile couldn't possibly be generated by the package.json.

>  I support this. And I'm going to delete my previous comment because it's noise.

似乎还改过口...

[issue21122](https://github.com/npm/npm/issues/21122#issuecomment-401634412)

同样的问题

>If one has to specify exact versions in package.json as some seem to suggest on reddit/SO/etc, then what is the point of package-lock.json?

> Try yarn or pnpm, npm unfortunately does not treat the lockfile as the single source of truth.

这哥子直接建议用yarn..

类似的问题在issue里还有很多



[stackoverflow的讨论](https://stackoverflow.com/questions/45022048/why-does-npm-install-rewrite-package-lock-json)

> That means that package.json can trump package-lock.json whenever a newer version is found for a dependency in package.json.
> To be clear: package-lock.json alone no longer locks the root level dependencies!
> 

想这样的讨论还有很多, 虽然我现在没有复现这个问题, 但不代表没有问题, 因为我以前被坑过, 同事也被坑过

## lockfile避坑指南

在实际项目里, 以下几点是我总结出的最佳实践

- 一定要提交lockfile, 不然ci环境下就真的和你开发环境依赖的版本不一样了
- 保证同一开发项目团队使用的node/npm的版本一致, 并且也与ci环境版本保持一致
    + [踩过坑的同学](https://juejin.im/post/5b6908ba6fb9a04f9c43e7b8)
    + npm@5以下都没有package-lock.json...
    + 不同版本表现不一致
    + 即使表现一致,ci出来的依赖能与node的环境不兼容(gulp与node10就是个例子)

如果要用npm, 有两个选择:

- 升级到npm@5.7.0, 这个版本提供[npm ci](https://docs.npmjs.com/cli/ci)命令, 他与npm i 的区别; 
    + 如果package.json和package-lock.json不能对应, 会报错, 但不改变package-lock.json
    + npm ci 会先删除node_modules
    + 任何情况下不会更改package.json或package-lock.json
- 锁定你的package.json依赖的版本号(不用^,~这样的符合), 预发因为semver版本导致package-lock.json失效
    + 以前遇到这个问题, 是这样做的, 检验是可行的
    + 保险起见, 万一npm某个版本没有锁住第一层的依赖的版本呢?

可以考虑yarn, 如果你的ci的环境没有npm@5.7.0, 保证你在ci环境完全还原开发的是指定的版本. 就像`npm ci`. 如果你的package-lock.json里面 有一个包在另一个源里, npm i 一定是安装不了, 比如package-lock.json是这种:

```
"xxxxa": {
  "version": "0.1.7",
  "resolved": "https://a.com.xxxxa.tgz",
  "integrity": "sha1-gt8AtgV0JGO9SKUIFQZznzrPx2w=",
  "requires": {
    "x": "0.1.27"
  }
},
"xxxxb": {
  "version": "0.1.7",
  "resolved": "https://b.com.xxxxa.tgz,
  "integrity": "sha2-gt8AtgV0JGO9SKUIFQZznzrPx2w=",
  "requires": {
    "y": "0.1.27"
  }
},
```

然后npm i, npm 默认的registry是a.com 那一定安装不了xxxxb , yarn没有这个问题

但注意, 使用yarn时, 必须保证,你使用的registry能在ci环境访问, 因为*yarn install 会完全根据你的yarn.lock里的源和version来安装*

## 我们团队的实践

- 统一本地的node版本, 与ci保持一致
- 统一使用registry
    + npm config set registry http://registry.npm.xiaojukeji.com/
- 一定提交lockfile(package-lock.json/yarn.lock)
- 锁定package.json里的包
    + 新代码安装是使用 `-E` 参数, 会写死版本号
        * npm i xx -E
    + 老代码去掉 `^ ~ < > *`等符号, 在npm@5.x可以依次执行
        * rm -rf node_modules && rm package-lock.json
        * npm outdated 查看可以更新的包
        * npm update
        * 手动去掉package.json里的所有^
        * npm i 
        * 验证项目
        * git commit -am 'lock package'
- 如有特殊需要, 可以使用yarn



