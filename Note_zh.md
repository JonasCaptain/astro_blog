> 学习笔记基于Astro官网学习，所以大部分记录都是复刻官网的内容

# 前提准备
 - Node.js >= v16.12.0
 - 建议使用带有astro官方扩展的vscode
 - 终端(Windows下使用powershell，或者cmd，Linux/mac自带的shell)


# 入门
通过命令行创建一个Astro项目，不需要事先创建新目录！向导将自动为你创建项目文件夹。

选择入门模板

你将看到可供你挑选的简短入门模板列表：

- Just the basics：对于想要探索 Astro 的人而言，这是个很好的入门模板。
- Blog、Documentation、 Portfolio：根据具体用例提供参考的模板。
- Completely empty：仅供入门使用的最小模板。

使用箭头键（向上和向下）导航到你要安装的模板，然后按回车（Enter）提交。

然后安装依赖，`npm install`

运行示例项目，`npm run dev`

部署到网络，`npm run build`，将在项目根路径下创建一个dist目录，将打包后文件放在里面

# 项目结构

## 目录和文件

- src/*: 项目代码(组件、页面、样式等)
- public/*: 非代码、未处理的资源(字体、图标等)
- package.json: 项目列表
- astro.config.mjs: Astro配置文件(可选)
- tsconfig.json: TypeScript配置文件(可选)

### src/* 文件夹

src下面存放的内容包括：
- 页面
- 布局
- Astro组件
- 前端组件(React等)
- 样式(CSS、Sass)
- Markdown

Astro处理、压缩和捆绑src/下文件并传递到Server，与静态的public/目录不同，src/下的astro文件有Astro建立并处理，且不会被发送到浏览器，而是被渲染成静态HTML。其他文件(如CSS)与其他CSS压缩之后一并发送。

### src/components

components是HTML页面中可重复使用的代码单元。它可以是Astro组件或是像React或Vue这样的前端组件。通常将项目中所有组件都分组放在这个文件夹中。

### src/layouts

layout是特殊的组件，它将一些内容包裹在一个更大的页面布局中。通常用在Astro页面和Markdown页面中以定义页面的布局。
和src/components一样，这个目录也是约定俗成。

### src/pages

page是一种用于创建新的页面的特殊组件。一个页面可以是一个Astro组件，也可以是一个Markdown文件，它代表网站的一些内容页面。

> src/pages 是Astro项目中必须要有的目录。没有它，你的网站将没有任何页面或路径。

### src/styles

在src/styles目录下存放CSS或Sass文件仍只是一个习惯。只要你的样式在src/目录下的某个地方，并且正确导入，Astro就能处理并压缩它们。

### public

public/目录用于文件和资源，它不会在Astro构建过程中处理。这些文件将不加修改直接复制到构建文件夹。
因此，public一般用于存放【图片】和【字体】等普通资源或robot.txt和manifest.webmanifest等特殊文件。

### package.json

Javascript包管理器用它来管理依赖关系。
在package.json中可以指定两种依赖关系：dependencies和devDependencies。大多数情况下它们效果一样，Astro在构建时需要所有依赖，而你的包管理器则会同时安装这两种依赖。Astro建议把所有依赖放在dependencies中，只有发现特殊需要后，在使用devDependencies。

### astro.config.mjs

每个入门模板都有它，它存储着Astro项目的配置。你可以在这里指定要使用的集成、构建选项、服务器选项以及其他内容。


### tsconfig.json

此文件在每个启动器模板中都会生成，它包括你的Astro项目的TypeScript配置项。如果没有tsconfig.json文件，编辑器将不完全支持某些功能。

