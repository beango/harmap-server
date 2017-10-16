---
title: NPM错误集锦
date: 2017-09-19 9:32:35
category: others
tags: [npm]
---

## Error: EPERM: operation not permitted on npm 5.4 on windows

``` bash
D:\workspace\HY\f7v2>npm install
npm ERR! path D:\workspace\HY\f7v2\node_modules\fsevents\node_modules\ansi-regex\package.json
npm ERR! code EPERM
npm ERR! errno -4048
npm ERR! syscall unlink
npm ERR! Error: EPERM: operation not permitted, unlink 'D:\workspace\HY\f7v2\node_modules\fsevents\node_modules\ansi-regex\package.json'
npm ERR!  { Error: EPERM: operation not permitted, unlink 'D:\workspace\HY\f7v2\node_modules\fsevents\node_modules\ansi-regex\package.json'
npm ERR!   stack: 'Error: EPERM: operation not permitted, unlink \'D:\\workspace\\HY\\f7v2\\node_modules\\fsevents\\node_modules\\ansi-regex\\package.json\'',
npm ERR!   errno: -4048,
npm ERR!   code: 'EPERM',
npm ERR!   syscall: 'unlink',
npm ERR!   path: 'D:\\workspace\\HY\\f7v2\\node_modules\\fsevents\\node_modules\\ansi-regex\\package.json' }
npm ERR!
npm ERR! Please try running this command again as root/Administrator.

npm ERR! A complete log of this run can be found in:
npm ERR!     C:\Users\Think\AppData\Roaming\npm-cache\_logs\2017-09-19T09_55_46_594Z-debug.log
```

错误原因：npm版本过高。

``` bash
npm -v prints: 5.4.0
node -v prints: 8.4.0
```

解决方案：降到5.3版本

``` bash
npm install npm@5.3 -g
```
