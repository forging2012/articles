# 别再浪费您的带宽

随着业务的增长，带宽的支出已经不容忽视，老大下达了领导的一个小目标：两周之内必须让带宽减少10%，帮公司节省一个(?)元。兵马未动，粮草先行，首先了解公司带宽费用的支出主要是两种：CDN与应用服务器的带宽，而站点数据基本就是两种类型：文本、图片。

数据缓存的处理已经在上一年做过调整，该优化的层面已经优化过了，那么就往数据压缩方向去调整了。

## 选择更好的CDN

### 文本压缩

对文本压缩处理，可能大家觉得这是标配的功能，没有什么选择的必要，但是我们要对各家对比之后，发现一个大家忽视的要点：压缩率。对于CDN中的文本文件，我们希望压缩率越高越好（解压速度很快，性能影响不大），对比了几家CDN的压缩率，差距居然达到16%（最差的是我们自已搭建的伪CDN），最好的是`unpkg`（但是国外的不好用）。在一翻价格与性能的对比之后，我们使用新的CDN测试站点，在静态文本资源的加载上大概减少8%左右的流量。

统计现有客户端对`brotli`的支持比率达到20%，我们联系过CDN服务商是否能提供智能的`brotli`与`gzip`压缩选择方案，但是对方说技术暂时未能支持，因此只能作罢。

### 图像压缩

图片主要都是`jpeg`为主，在原来的逻辑处理上已经将图片都以quality:90的方式做了预处理，一开始天真的想法是直接以quality:80的方式重新处理一次图片，带宽就立马能降下来。此方案一经提出，老大就否绝了，说这太掉面子，不能接受。一翻调研之后，最终选择使用`guetzli`压缩算法对现有的图片进行处理，理论上减少30%数据量的处理，老大一听觉得倍有面子（哥们没告诉他压缩的速度让他怀疑人生），于是先试点其它一个站点的图片看看压缩后的数据量能减少到多少。

经过一晚的图像压缩（一张不大不小的图片基本需要1分钟的处理时间），第二天向老大报告喜讯，数据比原来的减少了22%，当然处理时间平均也汇报了（给他来点冷水）。由于图片都是上传预处理，老大批准了方案的实施，但是要求后续改进为先以原有处理方式上传，后续`guetzli`处理完成后，再更新替换（报复来了报复来了，(๑•́ωก̀๑)）。

`webp`的图片数据量更少而且压缩速度比`guetzli`快多了，但是由于不是所有的客户端都支持，因此我们后续的改进方案也只是对`png`与`jpeg`的图片，都另外再生成一个`webp`版本，客户端根据是否支持`webp`选择适合的格式。因为此种修改需要客户端调整，因此并没有纳入此次调整中。（大部分的CDN都有提供图像处理，有些会提供webp之类的支持，可以根据自己的需求选择是自行处理还是由CDN的服务解决）

浏览器判断是否支持`webp`:

```js
// 判断是否支持webp格式
let checkWebpPromise = null;
async function isSupportWebp() {
  if (checkWebpPromise) {
    return checkWebpPromise;
  }
  const images = {
    basic: 'data:image/webp;base64,UklGRjIAAABXRUJQVlA4ICYAAACyAgCdASoCAAEALmk0mk0iIiIiIgBoSygABc6zbAAA/v56QAAAAA==',
    lossless: 'data:image/webp;base64,UklGRh4AAABXRUJQVlA4TBEAAAAvAQAAAAfQ//73v/+BiOh/AAA=',
  };
  const check = data => new Promise((resolve, reject) => {
    const img = new Image();
    img.onload = resolve;
    img.onerror = reject;
    img.src = data;
  });
  checkWebpPromise = Promise.all(_.map(images, check)).then(() => true)
    .catch(() => false);
  return checkWebpPromise;
}
```

## 应用服务的带宽优化

应用服务的都是接口服务，并不像静态文件可以做长时间的缓存，因此客户访问的频率较高，数据的压缩更显得重要了。

接口数据有做`gzip`压缩，但是compress level是1，这是怎么回事怎么回事？大致了解了一下，当时配置1是因为考虑到压缩数据是CPU密集型的处理，设置太高会占用太多的CPU资源，影响性能。这说得好像是那么一回事，先去看看最近一周的CPU统计，哇，我的大爷，这明明就是CPU一直在排排坐吃果果，闲得发霉。

老大大手一挥，把这个配置改为9，现在是CPU资源太多，不要浪费了CPU资源。小弟一听，不行啊，直接跳跃这么大，真挂了不就是我背锅，立即低眉折腰的说，改是肯定要改为9的，要节约带宽，不能浪费公司的钱，不过当时设置1肯定是有考虑的，要不我们循序渐进，先改为5试试。

最终先试验改成5，运行一个星期之后，CPU还是排排坐吃果果，大家也没啥好担忧了，直接切换为9，CPU使用率略有上升，系统稳定运行，带宽基本节约了20%以上，老大倍有面子。

优化任务：调整缓存服务器，对于可缓存的HTTP响应，生成`gzip`与`brotli`两组缓存数据，根据响应头`Accept-Encoding`来选择返回的数据格式。不可缓存的请求则使用`gzip`压缩（速度更快）。

## 小结

以前的优化一直都主要是关注于缓存的处理，这次的调整主要是基于压缩的处理，由于无法保证客户端都支持最新的压缩方式，因此为了更好的性能数据上有所冗余。在只调整基础服务，不调整应用服务，最终带宽的使用减少了10%以上，已完美达到目标。

如果需要尝试一下各种压缩处理后能减少多少数据，可以在`http://tiny.aslant.site/`上试验一下。