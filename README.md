### 看这里

[![Greenkeeper badge](https://badges.greenkeeper.io/Kelier/reptileForTest.svg)](https://greenkeeper.io/)

这里是一个node的基础爬虫程序，方便像和我一样的新手入门

> 这里采用两个npm包

```
npm install superagent //页面数据加载
npm install cheerio //页面数据解析
```

- superagent是nodejs里一个非常方便的客户端请求代码模块，superagent是一个轻量级的，渐进式的ajax API，可读性好，学习曲线低，内部依赖nodejs原生的请求API,适用于nodejs环境下。
- Cheerio 包括了 jQuery 核心的子集。Cheerio 从jQuery库中去除了所有 DOM不一致性和浏览器尴尬的部分，揭示了它真正优雅的API。Cheerio 工作在一个非常简单，一致的DOM模型之上。解析，操作，呈送都变得难以置信的高效。基础的端到端的基准测试显示Cheerio 大约比JSDOM快八倍(8x)。Cheerio 封装了兼容的htmlparser。Cheerio 几乎能够解析任何的 HTML 和 XML document。

#### 简书

我们先从首页抓取20条数据

理清一个思路：首先建一个app.js,做这么些事：
1. 引入依赖
2. 定义一个地址
3. 发起请求
4. 解析数据
5. 分析数据
6. 生成数据

##### part.1
```
const superagent=require('superagent');
const cheerio=require(('cheerio'));
```

##### part.2
```
const reptileUrl = "http://www.jianshu.com/";
```

##### part.3
```
superagent.get(reptileUrl).end(function (err, res) {
    // 抛错拦截
     if(err){
         throw Error(err);
     }
    // 等待 code
});
```

##### part.4
```
superagent.get(reptileUrl).end(function (err, res) {
    // 抛错拦截
     if(err){
         return throw Error(err);
     }
   /**
   * res.text 包含未解析前的响应内容
   * 我们通过cheerio的load方法解析整个文档，就是html页面所有内容，可以通过console.log($.html());在控制台查看
   */
   let $ = cheerio.load(res.text);
});
```

以上配置只是刚开始，没有太多难点，下面才是重头戏

打开官网，咱们发现：这20条数据存在页面一个类叫.note-list的ul里面，每条数据就是一个li，ul父级有一个id叫list-container，学过html的都知道id是唯一，保证不出错，我选择id往下查找。

难点是我们需要组装一个数据结构：

```
{
     id：  每条文章id
    slug：每条文章访问的id （加密的id）
    title： 标题
    abstract： 描述
    thumbnails： 缩略图 （如果文章有图，就会抓第一张，如果没有图就没有这个字段）
   collection_tag：文集分类标签
   reads_count： 阅读计数
   comments_count： 评论计数
   likes_count：喜欢计数
   author： {    作者信息
      id：没有找到
      slug： 每个用户访问的id （加密的id）
      avatar：会员头像
      nickname：会员昵称（注册填的那个）
      sharedTime：发布日期
   }
}
```

实际的页面dom
```
<li id="note-12732916" data-note-id="12732916" class="have-img">
    <a class="wrap-img" href="/p/b0ea2ac2d5c4" target="_blank">
      <img src="//upload-images.jianshu.io/upload_images/1996705-7e00331b8f3dbc5d.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/375/h/300" alt="300" />
    </a>
  <div class="content">
    <div class="author">
      <a class="avatar" target="_blank" href="/u/652fbdd1e7b3">
        <img src="//upload.jianshu.io/users/upload_avatars/1996705/738ba2908445?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96" alt="96" />
</a>      <div class="name">
        <a class="blue-link" target="_blank" href="/u/652fbdd1e7b3">xxx</a>
        <span class="time" data-shared-at="2017-05-24T08:05:12+08:00"></span>
      </div>
    </div>
    <a class="title" target="_blank" href="/p/b0ea2ac2d5c4">xxxxxxx</a>
    <p class="abstract">
     xxxxxxxxx...
    </p>
    <div class="meta">
        <a class="collection-tag" target="_blank" href="/c/8c92f845cd4d">xxxx</a>
      <a target="_blank" href="/p/b0ea2ac2d5c4">
        <i class="iconfont ic-list-read"></i> 414
</a>        <a target="_blank" href="/p/b0ea2ac2d5c4#comments">
          <i class="iconfont ic-list-comments"></i> 2
</a>      <span><i class="iconfont ic-list-like"></i> 16</span>
        <span><i class="iconfont ic-list-money"></i> 1</span>
    </div>
  </div>
</li>
```

我们拿两者进行比对。定义好每个元素获得值的对应方法，就是下面这个样子：
```
let data = [];
// 下面就是和jQuery一样获取元素，遍历，组装我们需要数据，添加到数组里面
$('#list-container .note-list li').each(function(i, elem) {
    let _this = $(elem);
    data.push({
       id: _this.attr('data-note-id'),
       slug: _this.find('.title').attr('href').replace(/\/p\//, ""),
       author: {
           slug: _this.find('.avatar').attr('href').replace(/\/u\//, ""),
           avatar: _this.find('.avatar img').attr('src'),
           nickname: replaceText(_this.find('.blue-link').text()),
           sharedTime: _this.find('.time').attr('data-shared-at')
       },
       title: replaceText(_this.find('.title').text()),
       abstract: replaceText(_this.find('.abstract').text()),
       thumbnails: _this.find('.wrap-img img').attr('src'),
       collection_tag: replaceText(_this.find('.collection-tag').text()),
       reads_count: replaceText(_this.find('.ic-list-read').parent().text()) * 1,
       comments_count: replaceText(_this.find('.ic-list-comments').parent().text()) * 1,
       likes_count: replaceText(_this.find('.ic-list-like').parent().text()) * 1
   });
});
```

里面的text()方法，可能原样带有空格或者换行符，需要加个方法过滤掉
```
function replaceText(text){
    return text.replace(/\n/g, "").replace(/\s/g, "");
}
```

然后我们这里将它写入一个文件中，引入内置的fs包

写文件的方法是:
```
fs.writeFile(filename,data,[options],callback);
/**
 * filename, 必选参数，文件名
 * data, 写入的数据，可以字符或一个Buffer对象
 * [options],flag 默认‘2’,mode(权限) 默认‘0o666’,encoding 默认‘utf8’
 * callback  回调函数，回调函数只包含错误信息参数(err)，在写入失败时返回。
 */
```

我们就照着模板来就好了
```
// 写入数据, 文件不存在会自动创建
fs.writeFile(__dirname + '/data/article.json', JSON.stringify({
    status: 0,
    data: data
}), function (err) {
    if (err) throw err;
    console.log('写入完成');
});
```

这样就大功告成了，自己node app.js看下效果吧

参考更多，请转向：[segmentfault](https://segmentfault.com/a/1190000009542336)