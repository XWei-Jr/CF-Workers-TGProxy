详细操作步骤
步骤 1：设置 Worker 回源域名（Fallback Origin）
按照【方案一】的步骤，先给你的 TGProxy Worker 绑定一个自定义域，例如 worker-tg.domain-a.com。
访问 https://worker-tg.domain-a.com/health，确认返回 {"ok":true}。
步骤 2：开通 Cloudflare for SaaS 并设置回源
在 Cloudflare 控制台打开主域名 domain-a.com。
在左侧菜单点击 SSL/TLS -> Custom Hostnames (自定义主机名)。
首次进入点击开通 Cloudflare for SaaS。
在页面下方的 Fallback Origin (回源来源) 输入框中，填入第一步绑定的域名：worker-tg.domain-a.com，点击 Add Fallback Origin。
等待 Fallback Origin 状态变为绿色的 Active。
步骤 3：添加自定义主机名（优选域名）
在同一页面点击 Add Custom Hostname。
Custom Hostname：输入客户端实际访问的域名，例如 tg.myproxy.com。
TLS 设置：
Minimum TLS Version: TLS 1.2
Certificate authority: 选择 Let's Encrypt 或 Google Trust Services
Certificate validation method: 选择 TXT Validation
点击添加。
步骤 4：在第三方 DNS 完成所有权与证书验证
添加完成后，点击展开该主机名，会看到两条需要添加的 TXT 记录：
Hostname Ownership Validation（主机名所有权）：
类型：TXT
主机记录：_cf-custom-hostname.tg
记录值：复制 Cloudflare 提供的校验字符
Certificate Validation（SSL 证书申请）：
类型：TXT
主机记录：_acme-challenge.tg
记录值：复制 Cloudflare 提供的校验字符
前往你管理 myproxy.com 的 DNS 解析平台（如 DNSPod），添加这两条 TXT 记录。等待 2~5 分钟后回到 Cloudflare 刷新，直到 Certificate status 和 Hostname status 全部变成 Active。
步骤 5：配置优选 IP 解析
当证书状态为 Active 后，即可在第三方 DNS 将该域名解析到测出的优质 Anycast IP：
本地使用测速工具（例如 CloudflareSpeedTest (https://github.com/XIU2/CloudflareSpeedTest)）测出当前宽带网络下延迟低、丢包为 0 的 Cloudflare IP 段。
在 DNS 控制台中，为 tg.myproxy.com 添加 A 记录，指向测出的优质 IP：
可以设置分线路解析：移动线路指向香港/日本优选 IP，电信/联通指向美西圣何塞优选 IP。
（也可以直接 CNAME 指向维护良好的公共优选 CNAME 地址）。
关键步骤：重新计算 Bridge URL 并配置 Telegram
在 CF-Workers-TGProxy 中，访问代理页面的鉴权参数 ?bridge=<token> 依赖了访问时的完整域名。如果更换了域名，之前的 bridge 参数会直接失效导致 404。
1. 本地生成新的 Bridge 链接
确保本地已安装 Node.js，打开终端执行以下命令（替换其中的 SECRET 和 DOMAIN 为你自己的配置）：
bash
node -e '
const crypto = require("crypto");
// 填写你在 Worker 环境变量配置的 SECRET
const secret = "你的SECRET_32位或34位hex";
// 填写你最终给客户端使用的域名（方案一的域名或方案二的优选域名）
const domain = "tg.myproxy.com";
const hexKey = secret.length === 34 && secret.startsWith("dd") ? secret : secret.slice(-32);
const key = Buffer.from(hexKey, "hex");
const hmac = crypto.createHmac("sha256", key);
hmac.update("tdesktop-web-proxy-bridge-v1\n" + domain.trim().toLowerCase());
const bridgeToken = hmac.digest("base64").replaceAll("+", "-").replaceAll("/", "_").replace(/=+$/, "");
console.log("\n================ 专属代理地址 ================");
console.log(`https://${domain}/?bridge=${bridgeToken}`);
console.log("==============================================\n");
'
2. 在 Telegram 客户端配置
Telegram Desktop：
启动参数测试：在终端或快捷方式属性中添加启动参数：
text
Telegram.exe --web-proxy="https://tg.myproxy.com/?bridge=<计算出来的token>"
或者进入 Telegram 设置：Settings -> Advanced -> Connection type -> 启用 Web Proxy / tproxy 并粘贴上述完整 URL。
Android / Webview：
在支持应用内 Web Proxy 协议的客户端中，填入上述生成的完整链接即可。
常见问题与排查清单
访问 URL 显示空白或 404：
检查 bridge 参数是否是用当前访问的精确域名（小写，不带端口）生成的。
检查 SECRET 是否与 Worker 环境变量中的配置严格一致。
连接出现 1034 错误 (Edge IP Restricted)：
该错误表示域名直接解析到了 Cloudflare IP，但 Cloudflare 边缘没有该域名的路由信息。请确认 Cloudflare for SaaS 中的 Custom Hostname 是否已是 Active 状态。
SSL 证书错误：
检查 SaaS 控制台中的证书验证状态，必须等待两条 TXT 记录生效并通过 DCV 证书签发后再切优选 IP 解析。
