---
title: "为开源文档做贡献 Contribut The Document"
date: 2021-05-07T16:30:32+08:00
draft: false
---

# 开始为开源文档做贡献

​	如果您想开始为ToadOCR文档做贡献，此页面及其相关主题可以帮助您开始使用。您无需成为开发人员或技术人员即可对ToadOCR文档和用户体验产生重大影响！编辑此页面上所有主题所需的只是一个GitHub帐户和一个Web浏览器。

## 有关文档的基础知识

​	ToadOCR文档是用Markdown编写的，并使用Hugo进行了处理和部署。来源在GitHub上，网址为https://github.com/suvvm/ToadOCRDocument/。大多数文档来源都存储在`/content/`目录下。

​	您可以从GitHub网站上提交问题，编辑内容以及查看其他方面的更改。您还可以使用GitHub的嵌入式历史记录和搜索工具。

### 文档布局

该文档遵循该帖子中描述的布局 [documentation](https://www.divio.com/blog/documentation/)。

它分为多个部分。每个部分都是`content/` 存储库的目录。

### 多种语言

​	ToadOCR文档当前还没有进行多语言文档处理，但hugo提供了完备的多语言处理方式，文档内容在/ content /中可以以多种语言的方式提供。通过添加由以下内容确定的两个字母的代码，可以用任何语言翻译每个页面：[ISO 639-1标准](https://en.wikipedia.org/wiki/List_of_ISO_639-1_codes)。没有后缀的文件默认为英语。

例如，页面的法语文档被命名为 `page.fr.md`

## 改善文件

### 修复现有内容

​	您可以通过修复文档中的错误或错字来改善文档。要改善现有内容，请在创建*fork*之后提交*拉取请求（PR）*。这两个词是[特定于GitHub](https://help.github.com/categories/collaborating-with-issues-and-pull-requests/)。出于本主题的目的，您不需要了解所有有关它们的信息，因为您可以使用Web浏览器来完成所有操作。

### 创建新内容。

​	存储库的源代码保存在 `develop`分支。因此，该分支必须是新分支的基础，PR也应指向该分支。

​	要创建新内容，请在与文档主题相对应的目录中创建一个新页面（请参见该段 [文档布局](https://gorgonia.org/getting-started/contributing-doc/#layout-of-the-documentation)）

​	如果您在`hugo`本地，则可以使用以下方法创建一个新页面：

```shell
hugo new content/about/mypage.md
```

否则，请创建一个标题如下的新页面：

```yaml
---
title: "The title of the page"
date: 2021-05-06T14:59:03+01:00
draft: false
---

your content
```

然后按照以下说明提交拉取请求。

### 提交拉取请求

请按照以下步骤提交请求，以改进ToadOCR文档。

- 在出现问题的页面上，单击右上角的“编辑此页面”图标。出现一个新的GitHub页面，其中包含一些帮助文本。

- 如果您从未在github上创建过ToadOCR文档存储库的分支，您可以这样做。在您的GitHub用户名下目标创建派生。分支通常具有一个网址，例如`https://github.com/<username>/website`，除非您已经有一个名称冲突的存储库。

  提示您创建派生的原因是您无权直接将分支推送到ToadOCR文档存储库。

- 出现GitHub Markdown编辑器，并加载了源Markdown文件。进行更改。在编辑器下方，填写**Propose file change**表单。第一个字段是提交消息的摘要，长度不得超过50个字符。第二个字段是可选的，但如果有条件，可以包含更多详细的描述信息。单击**Propose file change**。所做的更改将保存为提交到您分支中的新分支中，该分支会自动命名为`patch-1`。

- 下一个屏幕总结了所做的更改，方法是将新分支（**head fork** 与 **compare**选择框）与 **base fork** and **base**分支的当前状态进行比较（`develop` 在 `suvvm/ToadOCRDocument`）。您可以更改任何显示的选择框，但现在不要这样做。查看屏幕底部的分支差异查看器，如果一切看起来正确，请点击 **Create pull request**.

- 出现 **Open a pull request** 页面。拉取请求的分支与提交摘要相同，但是可以根据需要进行更改。正文由您的扩展提交消息（如果存在）和一些模板文本填充。阅读模板文本并填写所需的详细信息，然后删除多余的模板文本。如果添加到说明中`fixes #<000000>` 或者 `closes #<000000>`， 在哪里 `#<000000>`是相关问题的编号，当PR合并时，GitHub将自动关闭该问题。将**Allow edits from maintainers** 复选框取消。单击  **Create pull request**.

  恭喜你！您的提交的请求位于 [Pull Request](https://github.com/suvvm/ToadOCRDocument/pulls)。

- 等待审查。如果审阅者要求您进行更改，则可以转到 **Files changed** 页面，然后在拉请求中已更改的任何文件上单击铅笔图标。当您保存更改的文件时，将在由拉取请求监视的分支中创建一个新的提交。如果您正在等待审核者审核更改，请每隔7天主动与审核者（suvvm）联系一次。
- 如果您的更改被接受，那么审阅者会合并您的请求请求，并且几分钟后，更改会在ToadOCRDocument网站上发布。

这只是提交请求的一种方式。如果您已经是Git和GitHub的高级用户，则可以使用本地GUI或命令行Git客户端，而不是使用GitHub UI。