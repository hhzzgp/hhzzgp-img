图床设置教程：https://hanscn.com/post/tuc/

Hans Blog
两种免费图床搭建教程：GitHub + Cloudflare 加速 & Cloudflare R2（附 PicGo 配置详解）
知识分享

    全部文章 文章分类 标签

宝库资源

    软件下载 素材下载

苹果系统

    MacOS镜像 MacOS软件

LibreTV
关于
隐私政策
6
两种免费图床搭建教程：GitHub + Cloudflare 加速 & Cloudflare R2（附 PicGo 配置详解）
发表于2024-09-23更新于2026-04-16
深圳
CloudflareGitHubPicGoTypora免费图床Workers部署
两种免费图床搭建教程：GitHub + Cloudflare 加速 & Cloudflare R2（附 PicGo 配置详解）
Hans汉斯
2024-09-23
2026-04-16
  免费存储图片太难了，上传速度慢，甚至可能会遇到流量限制，尤其是部分图床服务在国内经常被墙，导致图片无法正常加载。

为了解决这一问题，本文将介绍两种高效、免费的图床搭建方法：

    GitHub 图床 + Cloudflare 加速：通过 CDN 优化 GitHub 图床外链速度，解决图片被墙问题。
    Cloudflare R2 对象存储图床：提供更高效的大容量图片存储和全球加速服务。

无论你是个人博主、写作爱好者，还是开发者，希望这篇文章能帮助你解决图片被墙的问题。
GitHub 图床 + Cloudflare 加速 部署
1. 问题背景：GitHub 图床的局限性

    GitHub 图床免费且稳定，但其图片外链 raw.githubusercontent.com 经常在国内访问缓慢，甚至被墙，影响图片加载速度。
    解决方案：利用 Cloudflare CDN 对 GitHub 图床的外链进行加速，将图片通过 Cloudflare 的域名访问，从而绕开网络限制。

2. 创建 GitHub 图床仓库

    登录 GitHub，

    创建一个新的存储仓库

01

    输入存储库名称,现在建立私库(不公开)（例如 hans-img）

    image-20241123164709435

3. GitHub 生成Personal access token(个人令牌)

    点击右上角用户头像，选择 Settings（设置）
    在页面左侧菜单栏最后，选择 Developer settings（开发人员设置）
    然后点击 Personal access tokens（个人[访问令牌）,选择 Token(classic)
    点击 Generate new token 开始创建一个新的令牌，注意一定要选择 classic 方式

image-20241123170701295

    输入个人令牌名称(自定义),Expiration选择No expiratio(永久)然后repo全部打勾,其他保持默认

image-20241123171129480

    保存生成的tokens

      注:要保存个人令牌,页面关闭后就无法查看!

image-20241123171417366

    上传图片到仓库，可以通过 GitHub 的 Web 界面直接拖拽上传。
    获取图片外链：上传后点击图片，右键复制 raw.githubusercontent.com 的 URL，即为图片的外链地址。

4. Cloudflare 加速 GitHub 图床

    登录 Cloudflare

    左侧菜单栏找到Workers和Pages

    创建一个新的Workers,输入项目名称,然后部署

    打开Workers项目,找到右上方 编辑代码

    清空编辑器原来代码,复制粘贴下面代码

    // Website you intended to retrieve for users.
    const upstream = "raw.githubusercontent.com";

    // Custom pathname for the upstream website.
    // (1) 填写代理的路径，格式为 /<用户>/<仓库名>/<分支>
    const upstream_path = "/hansvlss/PicGo-img/master";

    // github personal access token.
    // (2) 填写github令牌
    const github_token = "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";

    // Website you intended to retrieve for users using mobile devices.
    const upstream_mobile = upstream;

    // Countries and regions where you wish to suspend your service.
    const blocked_region = [];

    // IP addresses which you wish to block from using your service.
    const blocked_ip_address = ["0.0.0.0", "127.0.0.1"];

    // Whether to use HTTPS protocol for upstream address.
    const https = true;

    // Whether to disable cache.
    const disable_cache = false;

    // Replace texts.
    const replace_dict = {
      $upstream: "$custom_domain",
    };

    addEventListener("fetch", (event) => {
      event.respondWith(fetchAndApply(event.request));
    });

    async function fetchAndApply(request) {
      const region = request.headers.get("cf-ipcountry")?.toUpperCase();
      const ip_address = request.headers.get("cf-connecting-ip");
      const user_agent = request.headers.get("user-agent");

      let response = null;
      let url = new URL(request.url);
      let url_hostname = url.hostname;

      if (https == true) {
        url.protocol = "https:";
      } else {
        url.protocol = "http:";
      }

      if (await device_status(user_agent)) {
        var upstream_domain = upstream;
      } else {
        var upstream_domain = upstream_mobile;
      }

      url.host = upstream_domain;
      if (url.pathname == "/") {
        url.pathname = upstream_path;
      } else {
        url.pathname = upstream_path + url.pathname;
      }

      if (blocked_region.includes(region)) {
        response = new Response(
          "Access denied: WorkersProxy is not available in your region yet.",
          {
            status: 403,
          }
        );
      } else if (blocked_ip_address.includes(ip_address)) {
        response = new Response(
          "Access denied: Your IP address is blocked by WorkersProxy.",
          {
            status: 403,
          }
        );
      } else {
        let method = request.method;
        let request_headers = request.headers;
        let new_request_headers = new Headers(request_headers);

        new_request_headers.set("Host", upstream_domain);
        new_request_headers.set("Referer", url.protocol + "//" + url_hostname);
        new_request_headers.set("Authorization", "token " + github_token);

        let original_response = await fetch(url.href, {
          method: method,
          headers: new_request_headers,
          body: request.body,
        });

        connection_upgrade = new_request_headers.get("Upgrade");
        if (connection_upgrade && connection_upgrade.toLowerCase() == "websocket") {
          return original_response;
        }

        let original_response_clone = original_response.clone();
        let original_text = null;
        let response_headers = original_response.headers;
        let new_response_headers = new Headers(response_headers);
        let status = original_response.status;

        if (disable_cache) {
          new_response_headers.set("Cache-Control", "no-store");
        } else {
          new_response_headers.set("Cache-Control", "max-age=43200000");
        }

        new_response_headers.set("access-control-allow-origin", "*");
        new_response_headers.set("access-control-allow-credentials", true);
        new_response_headers.delete("content-security-policy");
        new_response_headers.delete("content-security-policy-report-only");
        new_response_headers.delete("clear-site-data");

        if (new_response_headers.get("x-pjax-url")) {
          new_response_headers.set(
            "x-pjax-url",
            response_headers
              .get("x-pjax-url")
              .replace("//" + upstream_domain, "//" + url_hostname)
          );
        }

        const content_type = new_response_headers.get("content-type");
        if (
          content_type != null &&
          content_type.includes("text/html") &&
          content_type.includes("UTF-8")
        ) {
          original_text = await replace_response_text(
            original_response_clone,
            upstream_domain,
            url_hostname
          );
        } else {
          original_text = original_response_clone.body;
        }

        response = new Response(original_text, {
          status,
          headers: new_response_headers,
        });
      }
      return response;
    }

    async function replace_response_text(response, upstream_domain, host_name) {
      let text = await response.text();

      var i, j;
      for (i in replace_dict) {
        j = replace_dict[i];
        if (i == "$upstream") {
          i = upstream_domain;
        } else if (i == "$custom_domain") {
          i = host_name;
        }

        if (j == "$upstream") {
          j = upstream_domain;
        } else if (j == "$custom_domain") {
          j = host_name;
        }

        let re = new RegExp(i, "g");
        text = text.replace(re, j);
      }
      return text;
    }

    async function device_status(user_agent_info) {
      var agents = [
        "Android",
        "iPhone",
        "SymbianOS",
        "Windows Phone",
        "iPad",
        "iPod",
      ];
      var flag = true;
      for (var v = 0; v < agents.length; v++) {
        if (user_agent_info.indexOf(agents[v]) > 0) {
          flag = false;
          break;
        }
      }
      return flag;
    }

    修改第6行、第10行,替换你自己的信息,然后点击右上角部署

      注:const upstream_path填写GitHub仓库路径，格式为 /<用户>/<仓库名>/<分支>;const github_token填许GitHub创建的个人令牌;

    image-20241123174028171

    添加你自己的域名到 Cloudflare

    点击设置 找到域和路由 点击添加 输入二级域名

    注:输入托管到Cloudflare的二级域名(域名前面添加一个名称就变成二级域名);

    image-20241123175639086

5. PicGo 配置

    打开PicGo,点击图床设置,找到GitHub

    图床设置GitHub

        图床配置名:自定义,

        仓库名:GitHub账号名/仓库名称

        设定Token:填写GitHub个人令牌;

        自定义域:https://+Workers绑定的二级域名;

        注:可以参考我的模版,替换你的信息,点击确定,然后设为默认图床;

    image-20241123201201930

    点击上传区,图片上传确认图床名称,然后通过拖拽图片/选择图片/剪贴板图片上传即可

    image-20241123192359420

    图床部署完成GitHub存储库验证

      注:由于PicGo存储路径没有设置,默认上传到根目录,可设置目录可以更好分类管理;

    image-20241123192809388

6. 优缺点总结
优点 	缺点
免费且易于配置 	私有仓库的容量限制为 1GB
稳定性较高 	适合中小型图床使用场景

注:每个私有仓库的容量限制为1GB,超过再建新的存储库就可以了

Cloudflare R2 图床部署
1. 什么是 Cloudflare R2？

Cloudflare R2 是一种对象存储服务，支持高效的图片存储和全球加速。与 GitHub 图床相比，它的优势是不限请求次数，适合需要大规模存储的场景。 image-20241123214208703

注:免费额度能满足个人博客的使用需求,即使超出之后,费用也是非常便宜,相比其他图床厂商,R2搭建图床,再不用担心跑路,稳定、可靠,还有白嫖额度,完美的选择.

2. 付款账户绑定

    登录 Cloudflare

    左侧菜单栏找到R2对象存储,需要绑定付款账户支持信用卡或PayPal账户

    image-20241123215300469

    注:因为R2有额度限制,超出需要付费,所以需要绑定付款账户,绑定VISa信用卡,如果没有信用卡可绑Paypal账号,PayPal账户需要绑定一张银行卡即可,操作很简单,相当于无门槛;

3. 创建 R2 存储桶

    进入 R2 控制台，创建一个新的存储桶（如 hans-img02）

    image-20241123224058352

4. 绑定自定义域名

    点击设置,公开访问自定义域名

    连接域,输入是绑定的二级域名(如img02.hansvlss.us.kg

    image-20241123225108486

5. 设置允许公开访问

    点击设置,找到R2.dev 子域,点击允许访问

    image-20241123225646466

    注:这一步很重要,如果不设置,上传图片后,是不能直接在公网访问,我们是为博客使用,使用要打开;

6. 创建R2 API令牌

    进入 R2 控制台，右边找到管理R2 API令牌

    image-20241123230332120

    右边创建API令牌,填写名称以及选择对象读与写,其他保存默认

    image-20241123230636373

    保存API令牌凭证(访问密钥ID、机密访问密钥)

image-20241123230954156

注:要保存API令牌凭证,页面关闭后就无法查看!
7. PicGo 配置

    打开PicGo,点击插件设置,搜索s3`并安装

    image-20241123231617942

    打开PicGo,点击图床设置,找到Amazon S3

    图床设置Amazon S3

        图床配置名:自定义,

        应用密钥ID:访问密钥 ID

        应用密钥:密访问密钥;

        自定义节点:S3 API链接(删掉链接最后 /桶名称)

        自定义域名:https://绑定的二级域名;

        image-20241123233211801

        注:可以参考我的模版,替换你的信息,点击确定,然后设为默认图床;

    上传图片到 R2 存储桶

        通过PicGO上传图片到存储桶

        在 R2 控制台中，手动上传图片到存储桶

        获取图片外链地址，通过公共端点访问上传的图片

        image-20241123233852952

8. 优缺点总结
优点 	缺点
支持大容量存储 	配置复杂，需要绑定付款账户
性能优越，全球加速 	超过免费额度可能产生费用
PicGo 安装

    下载地址：PicGo 官网。
    安装完成后，进入设置界面，选择“图床配置”。

Typora搭配PicGo 使用

    Typora设置：

        打开Typora文档 菜单栏文件

        找到偏好设置

        点击图像,找到上传服务设定

        上传服务:选择PicGO(app), PicGO路径:Typora软件安装路径(默认C:\Program Files\Typora\Typora.exe)

        点击验证图片上传选项 提示验证成功即可

        image-20241124135306754

    设置完成后，进入Typora编辑文档
        选择插入到文档中的图片
        鼠标右键:上传图片
        完成单图片快速上传

    image-20241124135805485

    注:如果整个文档所以图片快速上传方法,找到菜单栏---格式---图像---上传所有本地图片;

总结与推荐
方法 	GitHub 图床 + Cloudflare 加速 	Cloudflare R2 图床
适用场景 	小型图床、个人写作 	大型存储、全球访问
成本 	完全免费 	免费额度内免费，超出有费用
技术要求 	简单易学 	需要掌握 API 和存储配置

    如果是轻量级图片存储需求，优先选择 GitHub 图床 + Cloudflare 加速。
    如果需要大容量存储和高性能访问，选择 Cloudflare R2 图床。

通过搭配 PicGo，两种图床都能实现便捷的自动化上传，适合个人博客和 Markdown 写作者。
你是否遇到过博客图片被墙的问题？更喜欢哪种图床方案？
如果觉得教程有帮助，别忘了分享给你的朋友！
我是新人博主，需要您的支持，请帮忙给我点赞、关注、收藏，非常感谢！！！
头像头像
Hans汉斯
发现未知，学习有趣内容
两种免费图床搭建教程：GitHub + Cloudflare 加速 & Cloudflare R2（附 PicGo 配置详解）
本博客所有文章除特别声明外，均采用 CC BY-NC-SA 4.0 许可协议。转载请注明来自 Hans Blog！
Cloudflare4
GitHub3
PicGo1
Typora1
免费图床1
Workers部署1
cover of previous post
上一篇
免费域名注册us.kg：从域名注册到Cloudflare托管全流程教程 | 永久免费
cover of next post
下一篇
实测2024最稳定免费VPN:永久免费自建节点！免费解锁Netflix、ChatGPT！通过Cloudflare Worker、Pages部署免费的VLESS节点！薅羊毛党推荐!!!
喜欢这篇文章的人也看了
cover
2025-05-29
白嫖Cloudns免费域名注册教程,托管到Cloudflare ,替代 us.kg 获取永久免费域名,“小地球仪VPN”解决IP滥用问题
cover
2025-03-29
2025域名注册新玩法|一年只要 5 元，.XYZ 域名更划算！｜续费10年不超50元！新手域名注册全攻略|Cloudflare托管教程
cover
2024-09-23
免费域名注册us.kg：从域名注册到Cloudflare托管全流程教程 | 永久免费
cover
2025-07-11
🚀MoonTV 完整部署教程｜免费搭建影视聚合平台！支持 Cloudflare Pages + 自动更新 + 多资源接口
cover
2025-06-29
5分钟搭建免费的个人影视网站！零基础部署LibreTV教程（完整版 | Cloudflare Pages）全网资源 一键搜索 完全免费 无广告
评论
匿名评论隐私政策
TwikooWaline
✅ 你无需删除空行，直接评论以获取最佳展示效果
欢迎来到闲趣分享实验室！这里是我解决生活中各种小难题与需求的创意空间。无论是技术教程还是实用工具，愿你在这里找到灵感与帮助，共同探索开源的无限可能！
Hans汉斯
发现未知，学习有趣内容
文章目录

    GitHub 图床 + Cloudflare 加速 部署
    Cloudflare R2 图床部署
    PicGo 安装
    Typora搭配PicGo 使用
    总结与推荐

最近发布
PVE 软路由教程：OpenWrt 全功能部署指南，实现 2.5G 网口与无线网卡双直通（MT7612U 无线网卡）
PVE 软路由教程：OpenWrt 全功能部署指南，实现 2.5G 网口与无线网卡双直通（MT7612U 无线网卡）
比特浏览器-好用的指纹浏览器多开账号
比特浏览器-好用的指纹浏览器多开账号
解决 Adobe 软件“未经授权/非正版”弹窗保姆级教程
解决 Adobe 软件“未经授权/非正版”弹窗保姆级教程
Win11安装软件被拦截？一招关闭“智能应用控制”，告别报错！
Win11安装软件被拦截？一招关闭“智能应用控制”，告别报错！
NAS 旁路由部署全攻略：Docker 安装 Clash/Mihomo | TikTok 跨境住宅 IP 分流实操
NAS 旁路由部署全攻略：Docker 安装 Clash/Mihomo | TikTok 跨境住宅 IP 分流实操
©2020 - 2026 By Hans汉斯
主题
文章
27
标签
123
分类
0
功能
显示模式
网页
博客博客
项目
安知鱼图床安知鱼图床
知识分享

    全部文章
    文章分类
    标签

宝库资源

    软件下载
    素材下载

苹果系统

    MacOS镜像
    MacOS软件

LibreTV
关于
隐私政策
标签
1101错误解决1
AI2
AX6000刷机1
Adobe弹窗修复1
BPB节点4
BPB面板2
CF IP 优选2
ChatGPT2
ChatGPT Search1
Chrome1
Cloudflare Pages1
Cloudflare免费节点2
Firefox1
GPT2
Gemini 3 Pro1
Google Gemini1
OpenWrt3
PS 20251
Pages部署2
Perplexity1
Photoshop教程1
TikTok3
VLESS免费节点2
VPN节点订阅2
免费VPN搭建2
免费域名3
免费梯子2
免费翻墙2
指纹浏览器1
无限流量VPN1
比特浏览器1
注册美区苹果账户1
海外住宅IP2
港区苹果ID1
科学上网5
美区apple ID1
美国apple ID1
苹果账号1
跨境电商1
静态住宅IP4
