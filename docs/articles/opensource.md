# 开源项目

### Anto

一款基于当前市场主流`翻译`服务创建的`字幕翻译桌面应用`（仅支持`Windows`），使用`golang+lxn/walk`开发。速度挺不错，就是翻译质量取决于选择使用的翻译服务。当前主要功能有：

1. 支持选择翻译引擎，以及配置引擎对应的参数；

2. 支持不同翻译模式：增量翻译和全量翻译，增量翻译即忽略已翻译字幕，可有效提速；

3. 支持自定义选择导出轨道，适应不同场景需求；

4. 支持自定义主轨道来源（主字幕），即源字幕（来源语种）和翻译字幕（目标语种）的顺序，默认是源字幕在主轨，翻译字幕在副轨；

5. 支持单文件和目录选择模式，如果需要翻译多个目录，建议复制到同一个目录，方便检索处理；

6. 支持多任务模式；

当前支持的翻译引擎：

1. [阿里云机器翻译服务](https://mt.console.aliyun.com/)，每月免费额度100w字符；

2. [百度翻译](https://fanyi-api.baidu.com/doc/11)，每月免费额度：5w字符（标准版）、100w字符（高级版）、200w字符（尊享版）；

3. [彩云小译](https://docs.caiyunapp.com/blog/2018/09/03/lingocloud-api/)，每月免费额度100w字符；

4. [小牛翻译](https://niutrans.com/trans?type=text)，好像没有每月免费？；

5. [华为云翻译](https://support.huaweicloud.com/api-nlp/nlp_03_0024.html)，每月免费额度100w字符；

6. [有道智云](https://ai.youdao.com/)，每月免费额度？？？好像有点坑；

7. [腾讯云翻译](https://cloud.tencent.com/document/product/551/7372)，每月免费额度500w字符；

8. [LingVA翻译](https://lingva.ml/)，免费，我不知道是不是我的原因，这个站点单次请求翻译字符数量做了下调，罪过罪过；

9. [火山引擎](https://www.volcengine.com/docs/4640/62099)，每月免费额度200w字符；

**项目地址**：https://github.com/speauty/anto

**下载地址**：https://pan.quark.cn/s/24f36ee55269

**备用下载**：https://github.com/speauty/anto/releases