---
title: hexo主题开发简要指南
tags: 
	- 前端 
	- hexo
---
next主题虽然好看，但是实在是烂大街了，而且我更喜欢像word的那种可以折叠的目录。正好这段时间比较闲，所以我决定自己写个hexo主题，提不提交到hexo官方倒是其次，关键用自己写的主题心里舒服。

闲话不说，这篇文章是hexo主题开发的简要指南，并记录了一些我在开发过程中遇到的坑。

## hexo安装和使用

想要给hexo开发主题，首先得hexo玩得溜。hexo的安装和使用，[官方网站](https://hexo.io/zh-cn/docs/)上有相应文档，可以照着一步一步来。这里只提几点：

<!-- more -->

- 安装好的hexo，本质上是一堆node.js写的代码。这堆代码提供了hexo-cli，以便我们可以使用`hexo g`，`hexo clean`、`hexo d`之类的指令，来将markdown文件转化成网页并部署到某些网站上。另外，在新版本的hexo中，server模块被独立出来了，需要单独安装，才能使用`hexo s`命令。
- 既然我们要开发自己的主题，那么起码得知道怎么修改主题。打开网站文件夹根目录的`_config.yml`文件，找到`theme`那一行，改成`theme: xxx`，xxx即为我们自己主题的名字。然后把我们自己的主题文件夹丢到网站文件夹根目录的themes文件夹中。

## hexo的各个页面解释

hexo中有几种类型的页面：

- 主页，index。这是我们访问网站时看到的第一个页面。
- 文章，post。我们平时写的文章就是post，它在`source/_posts`目录中。用`hexo new post <name>`可以创建一个文章。当然，你直接去对应目录创建markdown文件也行。
- 草稿，draft。当文章还没有被发布时，它就是草稿状态，它在`source/_draft`目录中。用`hexo new draft <name>`可以创建一个草稿，然后用`hexo publish`命令可以将一个草稿发布为文章。
- 页面，page。这个概念官方没解释，而名字又比较唬人。page可以理解为除了index、post和draft之外的其它网页。典型的例子是archive、category和tag这三个网页。另外，我们可以新建自己的page。比如我不仅想要hexo能拿来写博客，还能拿来做相册。那我可以创建一个新的page，比如叫做photo。为此，我们使用`hexo new page photo`。这样，hexo会在source下创建一个名为photo的目录，即`source/photo`。打开这个目录，我们会看到一个`index.md`。在这个markdown中编辑一些文字，使用`hexo s`跑起来服务器以后，我们可以在`localhost:4000/photo/`中看到刚才编辑的文字（在`_config.yml`中设置了`pretty_urls`的话，可能网址会有所不同）。另外，如果我想实现`localhost:4000/photo/test`这样的子目录的效果的话，可以在`source/photo/`中创建`test`目录，并在该目录下新建一个`index.md`文件。当然，如果想让photo目录下的展示效果不同于普通文章的话，需要给它设置`layout`属性，并自己在主题中为它写对应的布局。
- archive，它属于page的一种，主要功能是按时间给文章归档。
- category和tag，它们都是一种page。按照[hexo的说法](https://hexo.io/zh-cn/docs/front-matter)，category是有层级之分的，而tag没有层级之分。实际上，它们的功能都是按照标签给文章归档。

## 主题的目录结构

[官方建议](https://hexo.io/zh-cn/docs/themes)一个hexo的主题的目录结构推荐如下：

> .
> ├── _config.yml
> ├── languages
> ├── layout
> ├── scripts
> └── source

作为一个不完全指南，我们只关注layout和source这两个目录。

- layout目录存放主题的模板，模板可以使用swig、ejs之类的。以下以ejs为例。对于index、post、page、archive、category和tag，分别设置他们各自同名的模板（即index.ejs、post.ejs之类的）。然后，最好还要有个layout.ejs。hexo在渲染所有页面时，都会默认使用layout.ejs。在layout.ejs中，使用`<%- body %>`会引入各个页面对应的同名模板。按照官网的例子，对于首页index而言，如果有以下文件：

  index.ejs

  ```ejs
  index
  ```

  layout.ejs

  ```ejs
  <!DOCTYPE html>
  <html>
    <body><%- body %></body>
  </html>
  ```

  会渲染出以下html：

  ```html
  <!DOCTYPE html>
  <html>
    <body>index</body>
  </html>
  ```

  当然，博客的作者也可以为不同文章指定不同的模板，只要他使用的主题中有该模板就行。比如上文提到的photo。

- 官方建议在layout里面再创建一个partial子目录，并把一些常用到的组件（比如侧边栏，header和footer）放到partial里面，然后用`<%- partial(layout, [locals], [options]) %>`这样的形式来引入这些组件。这个函数详见https://hexo.io/zh-cn/docs/helpers#partial

- source目录中建议创建css和js两个目录，然后将所有要用到的 css和js文件放到对应目录中。在模板中，使用`<%- css(path, ...) %>`和`<%- js(path, ...) %>`来引入这些文件。具体详见https://hexo.io/zh-cn/docs/helpers#css

## 细节和踩坑

主题的具体开发过程就不解释了，只要会html、css、js，另外再学点ejs或者swig之类的，就能上手开动了。这里主要记录一些具体的，比较坑的地方。

1. 官方文档太坑了，讲的很粗浅，不过还是得看，链接在这里https://hexo.io/zh-cn/docs/variables。
2. 归档的url为`yourwebsite/archives/`，然后，它可以进入更细分的url，比如展示2020年1月的所有文章，url为`yourwebsite/archives/2020/01`。如果在archive的首页想要展示所有的文章，就要注意了。如果我们使用`page.posts`，只能拿到部分文章，因为hexo的配置文件中限制了一页最多展示多少篇文章，默认是10篇。为此，我们需要用`site.posts`来获取所有的文章。同样，`yourwebsite/archives/2020/01`中，使用`page.posts`也不会获取到全部的2020年1月的文章，这时候我们唯一能做的是插入一个分页链接`<%- paginator(options) %>`，因为hexo会自动帮我们分页。详见https://hexo.io/zh-cn/docs/helpers#paginator
3. category和tag没有自己的index页面。也就是说，`yourwebsite/categories`和`yourwebsite/tags`这两个url是访问不了的。但是，你可以访问它更细分的某个url。比如说，你创建了一个叫做`foo`的tag，那么你可以访问`yourwebsite/tags/foo/`这个url。这一点和archive不一样，估计是官方认为archive的index也是category和tag的index。在开发中，我就是想要一个`yourwebsite/tags`，用这个页面展示左右的tag和其对应的文章。解决办法是使用page。
   - 首先，`hexo new page tags`创建一个tags的page。这样，网站文件中会出现`source/page/`这么一个目录。在这个目录下有一个`index.md`。
   - 打开`index.md`，在里面添上`layout: tag_index`这么一行（注意，不要添到正文里面去了，应该添到[Front-matter](https://hexo.io/zh-cn/docs/front-matter)中）。
   - 接下来，在主题文件目录的`layout`子目录中创建一个`tag_index.ejs`。
   - 这样，当我们访问`yourwebsite/tags`时，就会访问到`tag_index.ejs`生成的网页。
4. category和tag还有另一个坑。在官网中，`site.categories`、`site.tags`、`page.categories`、`page.tags`都没有给出它们具体的对象的结构。而且，用js打印出来也不行，因为它们是循环引用的，无法转成json。那么，问题来了，我想在`yourwebsite/tags`中，获取到某个tag对应的所有文章，该怎么弄？我的解决方法是这样的。以tag为例，
   - 首先，我们现在只用了`tag_index.ejs`这个模板，并没有用上`tag.ejs`这个真正该用的模板。在`tag.ejs`这个模板中，引用`page.posts`能获取到一部分属于某个tag的文章列表，剩下的文章列表在`page.next`_link所指向的url对应的下一个页面中。如果`page.next_link`为空，则没有下一个页面了。
   - 为此，我们可以把`tag.ejs`做成传json数据的页面。在`tag.ejs`中，遍历`page.posts`这个变量，并把这些post的信息和下一个页面的链接信息转成json，记录到当前页面中。
   - 在`tag_index`中，我们用`<%- list_tags([options]) %>`来获取到所有的tag列表。
   - 在`tag_index`中，我们还得写点js代码。tag的列表中的每个`a`元素的href，对应着一个以`tag.ejs`生成的页面。用ajax把这些页面抓取过来，提取出里面的json并解析，就能知道每个tag对应的所有文章了。另外，有些tag的文章太多，hexo把它给分页了，这就需要解析出json里面的next_link信息，继续读取下一个页面了。
5. hexo也提供了生成文章目录的函数，不要傻傻地自己去解析文章来生成目录。生成目录的函数是toc，详见https://hexo.io/zh-cn/docs/helpers#toc。
6. 善用已有的轮子，比如fancybox和highlight。我都是自己手动给代码高亮写完css了，才知道有个东西叫做highlightjs。

## 测试

在开发主题的过程中，很多时候想看看自己有没有什么没考虑周全的地方，或者看看文章展示效果好不好。这时候除了使用自己已有的文章来测试外，可以用hexo官方给的[单元测试](https://github.com/hexojs/hexo-theme-unit-test)。顺便一提，如果你想要发布自己写的主题，这个单元测试是必须的。我的主题目前还过不了这个测试233333.