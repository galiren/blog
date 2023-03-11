---
title: 使用 Cloudflare Workers 配合 jsDelivr 做图床
date: 2023-03-11 20:52:34
tags:
---

在最开始我的图片是存储在 [SM.MS](https://sm.ms) 上的免费空间里。这带来比较直接的问题是我无法使用自定义的域名去访问它，而是要用 SM.MS 生成的比较丑的链接。

然后就注意到 [jsDelivr](https://www.jsdelivr.com/) 有在为 GitHub 上的资源提供 CDN 服务，具体而言，对于一张存储在 `https://github.com/galiren/blog_assets` 的 `main` 分支下的图片 `gnome-desktop.png`，可以通过形如 `https://cdn.jsdelivr.net/gh/galiren/blog_assets@main/gnome-desktop.png` 的形式来访问它。正好 GitHub 我也是很常用的，所以想着应该可以把图片迁移到 GitHub 上存着。

之后就是解决自定义域名的问题了，jsDelivr 给的链接就是有迹可循的了，一开始采用的是 [`Cloudflare Page Rules`](https://www.cloudflare.com/features-page-rules/) 的转发功能，不过问题是转发后原地址 （形如 `image.mydomain.com/test.jpg`）就会变成 jsDelivr 的地址，虽然在 Markdown 里可以通过自己的域名去访问......但还是不满意。

随后在某群组里遇到万能群友说 [Cloudflare Workers](https://workers.cloudflare.com/) 可以解决问题。其实就是将一小段代码交给 Cloudflare，然后让 Cloudflare 根据所给代码来做操作。

Cloudflare Dashboard 可以看到 Worker，然后抄一下 [文档](https://developers.cloudflare.com/workers/examples/images-workers/) 里的代码修改一下：

```javascript
export default {
	async fetch(request) {
		const { pathname } = new URL(request.url);
		return fetch(`hhttps://cdn.jsdelivr.net/gh/galiren/blog_assets@main/${pathname}`);
	}
};
```
很简单，然后部署。部署好之后 Worker 是有一个域名，向这个域名请求符合 Workers 的要求就会运作这一小段代码返回 jsdelivr 的资源。Worker 也支持自定义域名，设置也很方便。

总的来说体验很不错，Cloudflare 实在是很好用，个人建站我做的功能都涵盖进来了。
