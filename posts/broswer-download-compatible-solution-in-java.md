---
title: '兼容各浏览器的文件下载时中文名称乱码的解决方案'
date: 2020-04-20 21:48:19
tags: [f0rb说,Java]
published: true
hideInList: false
feature: 
isTop: false
---

这段代码处理了文件下载时不同浏览器解析中文文件名所出现的乱码问题和firefox的空格截断问题，在IE, chrome, opera, safari, firefox下均测试通过。
```
public class DownloadServlet extends HttpServlet {  
    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {  
        // codes.. 
        String name = "中文名 带空格 的测试文件.txt";  
        String userAgent = request.getHeader("User-Agent");  
        byte[] bytes = userAgent.contains("MSIE") ? name.getBytes() : name.getBytes("UTF-8"); // name.getBytes("UTF-8")处理safari的乱码问题
        name = new String(bytes, "ISO-8859-1"); // 各浏览器基本都支持ISO编码  
        response.setHeader("Content-disposition", String.format("attachment; filename=\"%s\"", name)); // 文件名外的双引号处理firefox的空格截断问题
        // codes.. 
    }  
}  
```

原文链接：[https://f0rb.iteye.com/blog/1308579](https://f0rb.iteye.com/blog/1308579)



