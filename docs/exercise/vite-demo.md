# vite 初探

记录使用 vite + vue3 + typescript 的过程

1. 使用 vite 的模板创建

```
yarn create vite-app <project-name>

```

2. 如果想通过 vue add 的方式安装插件，还需要安装 @vue/cli-service，否则会报错

3. element-plus
   element-ui 针对 vue3 的包是 element-plus

```
vue add element-plus

```

4. typescript

```
vue add typescript

```

5. vue-router，通过 vue-cli 安装会报错，还是 npm 吧

```
npm i vue-router@4
```

遇到几个问题：

1. router/index.ts 里，如果引入 VueRouter

```
import VueRouter from 'vue-router'
```

会报错，即使加入 optimizeDeps 配置依旧不对
![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ad564127-6caf-4743-a37a-d07e1f20df31/Untitled.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ad564127-6caf-4743-a37a-d07e1f20df31/Untitled.png)

暂时未找到原因，通过按需引入可用

```
import {createRouter, createWebHashHistory} from'vue-router'
```
