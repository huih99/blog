---
title:  vue2环境下成功应用vite的实践总结
date: 2021-05-10
tags:
  - vue
  - vite
categories:
  - 笔记
---

# vue2环境下成功使用vite的实践总结

自从vite发布以来，社区赞誉无数，现在vite也更新到了2.0大版本，我也一直心水vite的极速编译的特性，特别是由于现在项目体积过大，已经严重拖慢了编译和热更新的速度，急需寻找一个解决方案。但无奈官方现在只有支持vue3的插件，而我的项目还是使用的vue2版本，于是我就一直在尝试如何将vite成功应用于vue2项目中，这是这段时间以来实践中总结出的一些问题解决方案，供大家参考。

说明： 按我设想的理想方案是：webpack作为打包工具，vite作为开发工具，所以我会考虑到在vite和webpack环境下尽量的维护一份共同配置。

##  如何正确的引入vue单文件

在@vue/cli搭建的项目环境中，webpack配置中增加了对`.vue`文件扩展名的解析，以使我们导入时可以省略`.vue`后缀，但在vite文档中不建议忽略自定义导入类型的扩展名（例如：.vue），因为它会影响 IDE 和类型支持，所以需要将以前所有没有写明`.vue`后缀的模块引入都给补上。由于项目庞大，手动去每个文件中更改肯定是不现实的事情，所以我写了一个脚本，在node中执行，自动补全`.vue`后缀名

```js
// rewriteImportPath.js

const path = require("path");
const fs = require("fs");
const chalk = require("chalk");

const overridePaths = [];
const r = /(?<!\/\/\s+|\*\s+)(?:im|ex)port (.*?) from "((?:\.{0,2}\/|[^.\s\r\n\t\\\/]+\/)*[^\s\r\n\t\\\/;]+)"|import\((?:\s*\/\*.*?\*\/\s*)?"((?:\.{0,2}\/|[^.\s\r\n\t\\\/]+\/)*[^\s\r\n\t\\\/;]+)"\)/g;

const baseDir = path.join(process.cwd(), "src");
const defaultExtensions = [".mjs", ".js", ".ts", ".jsx", ".tsx", ".json"];

function rewritePath(baseDir, rootPath) {
  try {
    const resolveAlias = require("./resolveAlias");
    const directory = fs.readdirSync(baseDir);
    directory.forEach(file => {
      const filePath = path.join(baseDir, file);
      const stat = fs.statSync(filePath);
      if (stat.isDirectory()) {
        rewritePath(filePath);
      } else if (/\.vue$|\.jsx?$/.test(file)) {
        const fileContent = fs.readFileSync(filePath, "utf-8");
        const newFileContent = fileContent.replace(r, ($$, $1, $2, $3) => {
          let importPath;
          const isDynamicImport = !!$1;
          const modulePath = isDynamicImport ? $2 : $3;
          if (modulePath.startsWith(".")) {
            importPath = path.join(baseDir, modulePath);
          } else {
            if (path.isAbsolute(modulePath)) {
              importPath = path.join(rootPath, modulePath);
            } else {
              const dirLevels = modulePath.split("/");
              const pathAlias = resolveAlias[dirLevels.shift()];
              if (pathAlias) {
                importPath = path.join(pathAlias, ...dirLevels);
              }
            }
          }
          if (!importPath) {
            return $$;
          }
          let suffix;
          const files = fs.readdirSync(path.dirname(importPath));
          const parsedImpPath = path.parse(importPath);
          files.some(file => {
            const parsedFilePath = path.parse(file);
            if (
              defaultExtensions.includes(parsedFilePath.ext) &&
              parsedFilePath.name === parsedImpPath.name
            ) {
              // 修复模块导入的后缀错误
              if (
                parsedImpPath.ext === ".js" &&
                parsedFilePath.ext === ".jsx"
              ) {
                suffix = parsedFilePath.ext;
              }
              return true;
            }
            if (
              !parsedImpPath.ext &&
              parsedFilePath.ext === ".vue" &&
              parsedFilePath.name === parsedImpPath.name
            ) {
              suffix = ".vue";
              return true;
            }
          });
          if (suffix) {
            const {dir,name} = path.parse(modulePath)
            const overridePath = $$.replace(
              modulePath,
              `${dir}/${name}${suffix}`
            );
            overridePaths.push("pathRewrite：" + $$ + "  >>>  " + overridePath);
            return overridePath;
          }
          return $$;
        });
        if (fileContent !== newFileContent) {
          fs.writeFileSync(filePath, newFileContent, "utf-8");
        }
      }
    });
  } catch (e) {
    console.error(`error in ${baseDir}：\n`, e);
  }
}
rewritePath(baseDir, process.cwd());
overridePaths.forEach(v => console.log(v));
console.log(
  "\n✔️ ",
  chalk.green("共成功改写" + overridePaths.length + "条引用路径")
);
```

## 如何让vite能够解析vue2代码
vite与vue2搭配使用，主要使用到的插件为[vite-plugin-vue2](https://github.com/underfin/vite-plugin-vue2)，这也是vite文档中推荐适配vue2开发的插件

## 如何启用jsx语法

在vite中使用`jsx`还是稍微有点麻烦的，一是使用到`jsx`语法的js文件都必须改成使用`jsx`后缀名，二是在vue的`sfc`组件中还得加上`jsx`标识

1. 首先需要在`vite-plugin-vue2`中启用`jsx`

    vite.config.js
    ```js
    import { defineConfig } from "vite";
    import { createVuePlugin } from "vite-plugin-vue2";
    export default defineConfig({
        plugins: [createVuePlugin({jsx: true})]
    })
    ```

2. `sfc`组件中加上`jsx`标识

    需要在**script block**中加上`lang=jsx`的标识
    ```vue
    <script lang="jsx">
        export default {
            render(){
                return <div>JSX Render</div>
            }
        }
    </script>
    ```

    **另外，由于现在开发vue2项目中基本都是在IDE中使用vetur插件启用vue特性支持，然而当在vue文件中的script标签中加上lang="jsx"后，无法再使用vetur进行代码格式化，详见此[issue](https://github.com/vuejs/vetur/issues/2781)**

3. 需要将所有使用到`jsx`语法的js文件后缀名改为`.jsx`

当`vite-plugin-vue2`发现有导入的资源是vue类型并且有`lang=jsx`的标识的时候，就会启用jsx转译，其核心依然是通过`babel`使用`@vue/babel-preset-jsx`进行转译,这里有一个坑点需要注意：

**当使用babel转译的时候，babel会默认搜寻当前项目目录中的babel配置文件，例如`babelrc`或者`babel.config.js`,如果当前项目存在着有babel的配置文件，则会在编译`jsx`语法代码的时候被启用,那么则需要确认配置文件中是否已经包含过@vue/babel-preset-jsx,不能重复添加同一个preset，否则编译会产生错误**

因为我项目中打算是开发环境使用`vite`，而生产环境则依然使用webpack进行打包，所以babel配置文件也是不能删除的，解决方案如下：

1. 如果是使用的babel.config.js文件，通过配置中`env`决定只在生产模式下才启用配置

    ```js
    module.exports = {
      env: {
        production: {
          presets: [
            [
                '@vue/app',
                {
                    'useBuiltIns': 'entry'
                }
            ]
          ]
        }
      }
    };
    ```

2. 使用`.babelrc`文件替换`babel.config.js`，因为`vite-plugin-vue2`中使用`babel`时指定了`babelrc:false`,也就是忽略`.babelrc`中的配置，避免使用到无关的`babel`配置。

    vite-plugin-vue2中的jsxTransform.js
    ```js
    function transformVueJsx(code, id, jsxOptions) {
        const plugins = [];
        if (/\.tsx$/.test(id)) {
            plugins.push([
                require.resolve('@babel/plugin-transform-typescript'),
                { isTSX: true, allowExtensions: true },
            ]);
        }
        const result = core_1.transform(code, {
            presets: [[require.resolve('@vue/babel-preset-jsx'), jsxOptions]],
            sourceFileName: id,
            sourceMaps: true,
            plugins,
            babelrc: false,
        });
        return {
            code: result.code,
            map: result.map,
        };
    }
    ```
我给`vite-plugin-vue2`的作者发起了一个[pr](https://github.com/underfin/vite-plugin-vue2/pull/78),在`babel.transform`中加入`configFile:false`选项以解决此问题，但是还未被成功采纳。

## 问题实录

### 一、不要在transition-group子元素上使用index作为key

解决方案： 不要在`v-for`循环中使用`index`作为key,应替换为唯一值


![image-20210401094139124.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/96e956a8e0444acfa5d83fd9874079d8~tplv-k3u1fbpfcp-watermark.image)

### 二、scss无法使用:export导出变量

目前`vite`不支持`:export`这种语法：

```sass
$primary: #1890ff
:export {
    primary: $primary
}
```

查找过相应的[issue](https://github.com/vitejs/vite/issues/1279)，尤大说的只会支持sass官方所支持的语法，建议使用`css-modules`替代。而:export实际并不是sass语法，webpack环境下支持`:export`这种写法实际是由**css-loader**提供的能力

解决方案：启用**css-modules**功能即可，开启**css-modules**功能也很简单，直接将scss文件后缀名改为 **module.scss**即可

### 三、 @ant-design/icons-vue导入报错

在项目中，使用到了`ant-design-vue`中的几个组件，使用的按需导入方式（通过[vite-plugin-imp](https://github.com/onebay/vite-plugin-imp)插件），其内部的图标组件就依赖了`@ant-design/icons-vue`，报错如下：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9b87a253d9d345a897f48c8f86cd5d2a~tplv-k3u1fbpfcp-watermark.image)

主要原因为：
node_modules\ant-design-vue\es\icon\index.js
```js
...
import * as allIcons from '@ant-design/icons/lib/dist';
...
```
从`@ant-design/icons/lib/dist`中导入的`allIcons`多了一个`default`的导出项,这是由于vite的**commonjs转es**模块所导致的，所以遍历`allIcons`的时候遍历到`default`的时候报错

解决方案：

不再使用按需导入的方式，而是通过全量引入，直接引用打包编译过后的代码，需要配置路径别名，当遇到`imoprt {xxx} from 'antd-design-vue'`的时候，指向到 `import {xxx} from 'ant-design-vue/dist/antd.min.js`


vite.config.js

```js
export defineConfig({
    resolve:{
        alias: [
            {find: /ant-design-vue$/,replacement: 'ant-design-vue/dist/antd.min'}
        ]
    }
})
```

**注意，当这样改了过后就不要再使用按需导入的插件了，只有引入编译后的代码才能解决这个错误，只能全量引入，在开发环境下，全量导入也是能接受的**

另外，全量引入后，记得手动引入`ant-design-vue`的样式文件

### 四、vue模版编译出现多余的空格字符

此问题可能导致页面内容错位，因为Dom结构中会出现多余的空白字符，导致文本内容间出现分隔、行内标签间出现空格。而`vue-cli`中默认是对模版编译选项中配置了对空格的处理选项，而`vite-plugin-vue2`是没有默认配置的，就导致默认情况下与`webpack`版`vue-cli`环境下的配置有所差异。详见此[issue](https://github.com/underfin/vite-plugin-vue2/pull/62).

解决方案：

vite.config.js

```js
import { createVuePlugin } from "vite-plugin-vue2";
export defineConfig({
      plugins: [
        createVuePlugin({
            jsx: true,
            vueTemplateOptions: {
                compilerOptions: {
                  whitespace: "condense"
                }
            }
        })
     ]
})
```

### 五、require.context在vite环境下无法使用

`require.context`是webpack独有的语法，用于创建一个require上下文，通常用于批量导入模块。然而在`vite`中根本不支持`require`，所有的模块导入都必须使用`import`，所以`vite`中也提供了一个用于批量导入的[glob导入](https://cn.vitejs.dev/guide/features.html#glob-import)功能

**import.meta.glob**

用于动态的导入多个模块（懒加载），且每个模块在构建时被分离为单独的chunk。例：
```js
const modules = import.meta.glob('./dir/*.js')

// 以上语句会被转译为如下的样子：

// vite 生成的代码
const modules = {
  './dir/foo.js': () => import('./dir/foo.js'),
  './dir/bar.js': () => import('./dir/bar.js')
}
```

**import.meta.globEager**

用于静态的导入多个模块，例：

```js
const modules = import.meta.glob('./dir/*.js')

// 以上语句会被转译为如下的样子：

// vite 生成的代码
import * as __glob__0_0 from './dir/foo.js'
import * as __glob__0_1 from './dir/bar.js'
const modules = {
  './dir/foo.js': __glob__0_0,
  './dir/bar.js': __glob__0_1
}
```

无论是批量动态导入还是批量静态导入，返回的都是一个对象，其中的`key`是该模块的所处路径，value是一个动态加载模块的函数或者直接就是一个导入的模块。而`require.context`返回的则是一个函数，所以在将`require.context`替换为`glob导入`时还需要注意获取获取导入的模块的方式改变。

由于与`require.context`的差异，所以在webpack环境中与在vite环境中无法使用通用的处理逻辑，最近我发现了一个`babel`插件[babel-plugin-transform-vite-meta-glob](https://www.npmjs.com/package/babel-plugin-transform-vite-meta-glob)，可以直接在webpack环境中使用`glob导入`方式，此插件会将其转译为与vite中相同的处理方式，这样的话可保持vite环境与webpack环境下批量导入模块的代码一致，例：

```js
const modules = import.meta.glob('./path/to/files/**/*')

const eagerModules = import.meta.globEager('./path/to/files/**/*')

// 以上代码会转译为如下这样

const modules = {
  './path/to/files/file1.js': () => import('./path/to/files/file1.js'),
  './path/to/files/file2.js': () => import(('./path/to/files/file2.js'),
  './path/to/files/file3.js': () => import(('./path/to/files/file3.js')
}

const eagerModules = {
  './path/to/files/file1.js': require('./path/to/files/file1.js'),
  './path/to/files/file2.js': require('./path/to/files/file2.js'),
  './path/to/files/file3.js': require('./path/to/files/file3.js')
}
```

### 六、Svg Icon如何使用

在webpack中使用svg icon,一般都是采用[svg-sprite-loader](https://github.com/JetBrains/svg-sprite-loader)去做处理，方便直接在代码使用。而vite下也有一个类似的插件[vite-plugin-svg-icons](https://github.com/anncwb/vite-plugin-svg-icons/blob/main/README.zh_CN.md),配置与`svg-sprite-loader`类似

```js
import { defineConfig } from "vite";
import { createVuePlugin } from "vite-plugin-vue2";
import viteSvgIcons from "vite-plugin-svg-icons";

export default defineConfig({
  plugins: [
    createVuePlugin({
      jsx: true,
      vueTemplateOptions: {
        compilerOptions: {
          whitespace: "condense"
        }
      }
    }),
    viteSvgIcons({
      iconDirs: [resolve(__dirname, "src/icon/svg")],
      symbolId: "icon-[name]"
    })
  ]
})
```

然后在项目入口的js文件中，添加一个模块引入：
```js
// main.js
...
import "vite-plugin-svg-icons/register";
...
```

大功告成，接下来即可像在webpack环境中使用**svg icon**一样尽情享用了。

### 七、别名配置

vite中同样支持与webpack类似的别名系统
```js
import {defineConfig} from 'vite';

export default defineConfig {
  resolve: {
    extensions: [".mjs", ".js", ".ts", ".jsx", ".tsx", ".json"],
    alias: [
      ...Object.keys(aliasConfig).map(key => ({
        find: key,
        replacement: aliasConfig[key]
      })),
      { find: "timers", replacement: "timers-browserify" },
      { find: /ant-design-vue$/, replacement: "ant-design-vue/dist/antd.min" }
    ]
  }
}
```
这里我是将别名配置抽出为一份公共的配置文件，以备webpack和vite使用一份相同的别名配置：

```
// aliasConfig

const {resolve} = require('path');
module.exports = {
  "@": resolve(__dirname, "src"),
  static: resolve(__dirname, "public/static"),
  YZT: resolve(__dirname, "src/views/YZT"),
  FZSC: resolve(__dirname, "src/views/FZSC"),
  JCYJ: resolve(__dirname, "src/views/JCYJ"),
  GHSS: resolve(__dirname, "src/views/GHSS"),
  FZBZ: resolve(__dirname, "src/views/FZBZ"),
  ZBMX: resolve(__dirname, "src/views/ZBMX"),
  YWXT: resolve(__dirname, "src/views/YWXT"),
  MXXT: resolve(__dirname, "src/views/MXXT"),
  JDGL: resolve(__dirname, "src/views/JDGL"),
  JDTB: resolve(__dirname, "src/views/JDTB")
};
```

## 剩余待解决问题

1. vue中script标签中增加`lang="jsx"`标识后，更改代码无法实现热更新,关联[issue](https://github.com/vitejs/vite/issues/3008)
2. vue中script标签增加`lang="jsx"`标识后，无法使用vetur格式化，关联[issue](https://github.com/vuejs/vetur/issues/2781)
3. 热更新状态丢失，关于这个问题，我搜了很多issue，但都没找到有类似问题的记录，并且我使用demo做测试时也并不能复现问题，但是实际应用在项目，就确实会出现这个问题，还没有找到头绪
4. 某些依赖包由于内部使用了`require`导入的语法，导致vite不能成功将其转化为es模块，在运行时会报错。

## 总结

现在我的项目已经能成功使用vite启动起来了，但实际感受下来，在项目启动速度上并没有质的改变，因为项目过大，初次启动时通过网络请求加载模块的数量也非常非常多，导致速度并没有明显提升。但是在热更新方面，确实如尤大所说：速度快到惊人。

现在基于vite成功应用在生产环境中的案例也还比较少，遇到些问题都还得自己探索，但不可否认，vite提供了一种与webpack完全不同的方式，并且在编译和热更新速度上有较大的优势，期望能尽快完善vite相关的体系，能更加便捷的应用在项目中。

大家平时可以多关注下这个项目：[awesome-vite](https://github.com/vitejs/awesome-vite)，里面有很多推荐的插件集合和各种不同技术栈使用vite的示例项目，遇到问题时可以提供一些解决思路。
