# Subresource Integrity (SRI) for Validating Web Resources Hosted on Third Party Services (CDNs)
使用子资源完整性（SRI）验证托管在三方平台（CDNs）的网站资源

On the 23rd of June 2016 the World Wide Web Consortium (W3C) announced the Subresource Integrity (SRI) as a W3C recommendation.

2016年6月23日，万维网联盟（W3C）宣布将子资源完整性（SRI）作为 W3C 推荐标准。

On the same day we also announced some new web security checks for the Subresource Integrity in our automated web application security scanner.

同一天，我们也公布了使用我们的自动网站安全扫描平台进行子资源完整性的一些安全检查项目。(这个是推荐后面图示中他们自己平台的，如有兴趣，请看原文查看相关链接)

## What is Subresource Integrity?
什么是子资源完整性？

Subresource Integrity (SRI) is a method that allows web application developers to ensure that resources hosted on third party services such as Content Delivery Networks (CDN) has been delivered without any unexpected modifications.

子资源完整性（SRI）是一种允许网站应用程序开发人员确保托管在第三方服务，如内容交付网络（CDN）上的资源，在传递过程中未被修改的方法。

The W3C recommended the Subresource Integrity (SRI) as a best-practise whenever resources are loaded from a third-party source.

W3C 建议将子资源完整性（SRI）加载第三方资源的最佳实践。

## How Does the Subresource Integrity (SRI) Work?
子资源完整性（SRI）如何工作？

SRI does this by using hash comparisons; it compares the hash value of the resources hosted on the web server and those hosted on the third party server or service.

SRI 使用 hash 来保证子资源完整性。他将会对比保存在网站服务器上的资源以及第三方服务或服务器上托管资源的 hash 值。

## Why Use the Subresource Integrity (SRI) ?
如何使用子资源完整性（SRI）？

To improve the performance of their websites many organizations hosts different resources on different servers. For example resources such as scripts, CSS stylesheets and images are typically hosted on a content delivery network (CDN).

为了提高网站的性能，许多公司或组织选择在不同的服务器上托管不同的资源。诸如脚本、CSS 样式表和图像之类的资源通常托管在内容传递网络（CDN）上。

However by doing so you put explicit trust in your CDN or third party service. This means that should the CDN service get hacked, or the DNS is hijacked then your web application is hacked as well. At this stage the attacker can modify the content of a script file that is hosted on your CDN which will lead to a cross-site scripting vulnerability on your website.

但如果这样做，您将百分百信任您的 CDN 或第三方服务商。这意味着如果 CDN 服务商遭到黑客攻击或者 DNS 被劫持，那么您的网站应用程序也会被黑客入侵。在此阶段，攻击者可以修改 CDN 上托管的脚本文件的内容，这将导致您网站出现跨站点脚本漏洞。

Therefore by implementing SRI you ensure that the web application is referring to a file that is actually legitimate, and should the file change your browser will not load it and the attack will fail.

因此通过实施 SRI，您可以确保网站应用程序引用的文件是合法的，如果文件发生更改，您的浏览器将无法加载它以及对于网站的攻击将会失败。

You can check if your browser supports the Subresource Integrity from the [Can I Use](https://caniuse.com/#feat=subresource-integrity) website.

您可以从我可提供的网站查看您的浏览器是否支持子资源完整性。

## Why Isn't HTTPS Enough?
为什么只有 HTTPS 是不够的？

HTTPS is used to ensure that the connection between the user's browser and the web server is encrypted. Therefore in case the content on the third party server or service is tampered with malicious code, it will still be delivered to the user, irrelevant if the connection is encrypted or not.

HTTPS 用于确保用户浏览器和万丈服务器之间的连接加密。 但是如果第三方服务器或服务上的内容被恶意代码篡改，它仍将被传递给用户，这与连接是否加密无关。

## Netsparker Subresource Integrity (SRI) Security Checks
Netsparker 的子资源完整性（SRI）安全检查

When you scan your websites with the Netsparker web vulnerability scanner it will check if SubResource Integrity (SRI) is implemented.

当您使用 Netsparker Web 漏洞扫描程序扫描您的网站时，它将检查是否已实施子资源完整性（SRI）。

![subresource_integrity_not_implemented](https://raw.githubusercontent.com/1018ji/OCTP/master/201901/Subresource%20Integrity%20(SRI)%20for%20Validating%20Web%20Resources%20Hosted%20on%20Third%20Party%20Services%20(CDNs)/subresource_integrity_not_implemented.png)

And if SRI is implemented, the security scanner will also check for invalid hashes. Therefore should Netsparker report a mismatch in the hashes, as in the below screenshot, it means that an external resource has been compromised.

如果实施了 SRI，安全扫描程序还将检查哈希值是否有效。如果 Netsparker 报告哈希值不匹配，如下面的屏幕截图所示，那么表示外部资源已被泄露。

![subresource_integrity_sri_invalid_hash](https://github.com/1018ji/OCTP/blob/master/201901/Subresource%20Integrity%20(SRI)%20for%20Validating%20Web%20Resources%20Hosted%20on%20Third%20Party%20Services%20(CDNs)/subresource_integrity_sri_invalid_hash.png?raw=true)

## Example: implementing Subresource Integrity (SRI)
实例：实施子资源完整性（SRI）

The format of the integrity HTML attribute is:

完整性 HTML 属性的格式为：

```
integrity="[hash algorithm]-[base64 encoded cryptographic hash value]
```

The hash algorithms that can be used are; sha256, sha384 or sha512. Therefore to enable the Subresource integrity checks simply add the integrity HTML attribute to the script tag as shown in the below example:

可以使用的哈希算法为 sha256、sha384 或 sha512。 因此要启用子资源完整性检查，只需将完整性 HTML 属性添加到脚本标签，如下例所示：

```
<script src="https://code.jquery.com/jquery-2.1.4.min.js" integrity="sha384-R4/ztc4ZlRqWjqIuvf6RX5yb/v90qNGx6fS48N0tRxiGkqveZETq72KgDVJCp2TC" crossorigin="anonymous"></script>
```

For more detailed information on SRI refer to the [W3C SRI documentation](https://www.w3.org/TR/SRI/).

有关 SRI 的更多详细信息，请参阅 [W3C SRI 文档](https://www.w3.org/TR/SRI/)。

## 原文

[Subresource Integrity (SRI) for Validating Web Resources Hosted on Third Party Services (CDNs)](https://www.netsparker.com/blog/web-security/subresource-integrity-sri-security/)

****
**THE END [ 2019-01-25 ]**