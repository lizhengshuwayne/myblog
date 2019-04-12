---
title: lianfan-publisher
date: 2019-04-11 21:30:14
tags: little tools
---

## about

心血来潮，用js写了一个脚本，用于发布公司产品，不用手动，~~然而并没有什么卵用~~很爽。

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

公司的项目由web上对应的打包工具，输出zip格式的压缩包的下载地址。

对于文件名我做了特殊处理，输出的文件名带有时间戳， 即productName_YYYYMMDDHHss.zip。

同时生成一份内容为当前最新包的名称的txt文件。

之后只需要先下载txt文件，识别内容，取得最新包的filename，就可以下载对应的包了。

```bat
set filename=mobile_%date:~0,4%%date:~5,2%%date:~8,2%%time:~0,2%%time:~3,2%.zip
7z a %filename% ./dist/*
echo %filename% > ./dist/latest_filename.txt
```
然后连接ssh服务器，上传包于对应的tmp文件夹，执行shell命令， 解压包到对应发布目录并覆盖同名文件，到此发布完成。

```shell
unzip -o /var/lianfan/static/mobile_YYYYMMDDHHss.zip -d /var/lianfan/static/mobile/
```

### analysis

如果是第一次运行，需要初始化一些数据，登录打包网站的username:password, 连接对应ssh服务器的host、 username和password，生成.cache文件作为缓存, 方便下次使用。

下次运行脚本可以选择是否重新初始化。

```js
// init()
const answers = await inquirer.prompt([
    {
        type: "input",
        name: "host",
        message: "input your remote ssh host: ",
    },
    {
        type: "input",
        name: "username",
        message: "input your remote ssh username: ",
    },
    {
        type: "input",
        name: "password",
        message: "input your remote ssh password: ",
    },
    {
        type: "input",
        name: "psd",
        message: "input your ci username and password (username:password): ",
    },
]);
const answersStr = JSON.stringify(answers);
fs.writeFileSync(".cache", answersStr);
```

```js
// init ssh's host, username, password and ci's psd
if (fs.existsSync(".cache")) {
    const {isChange} = await inquirer.prompt([
        {
            type: "confirm",
            name: "isChange",
            message: "The ssh and ci's cache is exist. Change it ?",
            default: false,
        },
    ]);
    if (isChange) {
        await init();
    }
} else {
    await init();
}
```

选择你需要发布的产品类型。

```js
// choose which product is your wish
const {product} = await inquirer.prompt([
    {
        type: "list",
        name: "product",
        choices: ["dashboard", "mobile", "pc", "pc_intern"],
        default: "dashboard",
    },
]);
```

通过下载解析latest_filename.txt文件，取得对应的zip包，放在.back目录下，方便版本追溯。

```js
// download()
const content = fs.readFileSync(".cache", "utf8");
const {psd} = JSON.parse(content.trim());
await download(
    `http://ci.lianfan.net/job/numas/job/${productMap[product]}/ws/${pre}${filename}`,
    `./.back/${product}`,
    {
        headers: {
            Authorization: `Basic ${Buffer.from(psd).toString("base64")}`, // 类curl协议 Authorization:base64(username:password)
        },
    },
);
```

```js
// 下载记录最新zip包文件名的txt
await download("latest_filename.txt", product, "dist/");

// 获取最新zip包名称
const content = fs.readFileSync(`./.back/${product}/latest_filename.txt`, "utf8");
const filename = content.trim();
console.log("get filename success!");

// 下载对应zip包
await download(filename, product);
```

连接远程服务器

```js
// 下载记录最新zip包文件名的txt
const cache = fs.readFileSync(".cache", "utf8");
const {host, username, password} = JSON.parse(cache.trim());
await ssh.connect({
    host,
    username,
    password,
});
```

上传zip包并执行命令

```js
await ssh.putFile(`./.back/${product}/${filename}`, `/var/lianfan/static/${filename}`);

// 执行shell命令
await ssh.execCommand(`unzip -o /var/lianfan/static/${filename} -d /var/lianfan/static/${product}/`, {cwd: "/"});
```

## others

折腾了很久，学会了怎么用轮子拼装起来，做一个成熟的手推车。

犹豫就会败北，岂可修。