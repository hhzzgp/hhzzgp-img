图床设置教程：

使用GitHub 图床 + Cloudflare 加速：通过 CDN 优化 GitHub 图床外链速度，解决图片被墙问题。  

---

## **一、准备工作**
1. **GitHub 账号**：注册好 GitHub，并创建一个仓库用于存放图片，比如叫 `my-images`。  
2. **Cloudflare 账号**：注册好 Cloudflare，用于加速你的 GitHub Pages 域名。  
3. **自定义域名（可选）**：为了使用 Cloudflare 加速，你最好有一个域名（可以用免费的 `freenom` 或已有域名）。  

---

## **二、GitHub 图床搭建**
1. **创建仓库**  
   - 仓库名随意，比如 `my-images`  
   - 初始化时可以选择 `README.md`  
2. **上传图片**  
   - 可以直接在 GitHub 页面上传，也可以用 Git 命令批量上传  

3. **开启 GitHub Pages**  
   - 设置 → Pages → Source → 选择 `main` 分支，根目录 `/`  
   - 保存后会生成一个访问 URL，例如：  
     ```
     https://yourusername.github.io/my-images/
     ```
   - 你上传的图片访问地址就是：
     ```
     https://yourusername.github.io/my-images/example.png
     ```

---

## **三、用自定义域名 + Cloudflare 加速**
1. **添加自定义域名到 GitHub Pages**  
   - 在仓库根目录建一个 `CNAME` 文件，内容写你的域名，例如：
     ```
     img.yourdomain.com
     ```
   - 在 GitHub Pages 设置里也填写相同域名  

2. **Cloudflare DNS 设置**  
   - 登录 Cloudflare → 添加你的域名  
   - DNS 设置：
     - 类型：CNAME  
     - 名称：`img`  
     - 目标：`yourusername.github.io`  
     - 云朵状态：开启（橙色云）  

3. **开启 Cloudflare CDN 加速 & HTTPS**  
   - Crypto → SSL/TLS → Full 或 Flexible  
   - Caching → 默认即可  
   - Page Rules → 可配置缓存策略，例如：
     ```
     URL: img.yourdomain.com/*
     Cache Level: Cache Everything
     Edge Cache TTL: a month
     ```

---

## **四、测试图片加速效果**
1. 上传一张图片：
   ```
   example.png
   ```
2. 外链访问：
   ```
   https://img.yourdomain.com/example.png
   ```
3. 测试国内访问速度和防封锁效果  

---

## **五、额外优化**
- 图片较多时，建议用 **Git LFS** 管理大文件  
- 可以配合 **Cloudflare Workers** 做智能缓存  
- 注意 GitHub 每月流量限制，如果访问量大可以考虑额外 CDN  

---

✅ **总结**  
1. GitHub 作为图床存储  
2. GitHub Pages 提供静态访问  
3. 自定义域名 + Cloudflare CDN 加速和防墙  

这样搭建完成后，你的图片就能高速、稳定访问，几乎不会被墙。

---


GitHub Pages
