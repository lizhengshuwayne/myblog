---
title: lianfan-publisher
date: 2019-04-11 21:30:14
tags: little tools
---

## about

自动化部署钉钉《连帆排班》前端工程的脚本。

项目地址：[https://github.com/lizhengshuwayne/lianfan-publisher](https://github.com/lizhengshuwayne/lianfan-publisher)

## details

### dependencies

*   [node-ssh](https://github.com/steelbrain/node-ssh)

    用promise封装的 ssh client and server js库。

*   [inquirer](https://github.com/SBoudrias/Inquirer.js)

    非常强大的交互式命令行js库，实现终端交互。

*   [download](https://github.com/kevva/download)

    用于下载的js库。
    
### demands

手动部署前端工程很不方便，而且容易出现误操作。

需要实现可自动化部署的脚本来简化工作内容。

### analysis

第一次运行，需要初始化一些验证信息：打包服务器的username:password，部署服务器的host、 username和password。

生成一份.cache文件保存验证信息, 方便下次使用。

可重复输入命令行以多次覆盖验证信息。

```sh
$ yarn init-cache
```

从打包服务器下载最新版本的包，并上传到部署服务器备份。

过程中，获取了当前上传时间，将最新的包重名为带有时间搓的文件名，如mobile_201901010000.zip。

```sh
$ yarn update-package
```

选择需要的版本进行发布。

列出备份的不同版本包的文件名，并按照时间倒序排序，作为待选项让发布者自行选择。

为了便于发布新版本或者还原到任意老版本，因此让发布者自行选择发布版本。

```sh
$ yarn publish-package
```

“选择对应版本发布前，需要确认是否发布选中的版本，如果不是，则重新选择版本。”

此处可用递归实现。如下：

```js
// 定义一个getPackageNameAsync函数，首先选中版本，确认该版本则返回文件名，否则再次执行getPackageName操作。
const getPackageNameAsync = async (packageNames) => {
    const {packageName} = await inquirer.prompt([
        {
            type: "list",
            name: "packageName",
            message: "Which package will be published?",
            default: packageNames[0],
            choices: packageNames,
        },
    ]);
    const {isSure} = await inquirer.prompt([
        {
            type: "confirm",
            name: "isSure",
            message: `Are you sure to publish ${packageName}?`,
            default: false,
        },
    ]);

    if (isSure) {
        return packageName;
    } else {
        return await getPackageNameAsync(packageNames);
    }
};
```

## others

待实际场景验证可靠性及实用性。