title: HttpURLConnection-关于HttpRequestMethodNotSupportedException异常
author: Xiang Chuang
tags:
  - HttpURLConnection
  - HttpRequestMethodNotSupportedException
  - post/get请求
categories:
  - 爱学爱问
date: 2019-05-16 21:37:00
---
背景：提供场景化业务监控能力时，需要通过捕获到的异常流水gid直接跳转到apm进行链路信息查询。

方案：调用apm提供的gid翻译为traceId方法，该能力是基于spring mvc提供，并且是https协议。godeye中利用HttpURLConnection封装post及get方法进行调用：
```
URL realUrl = new URL(url);
            HttpURLConnection conn = (HttpURLConnection )realUrl.openConnection();
            conn.setRequestProperty("accept", "*/*");
            conn.setRequestProperty("connection", "Keep-Alive");
            conn.setRequestProperty("user-agent", "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1;SV1)");
            conn.setRequestMethod("POST");
            conn.setRequestProperty("Accept-Charset", "UTF-8");
            conn.setRequestProperty("Content-Type", "application/json;charset=UTF-8");
            conn.setRequestProperty("Content-Length", String.valueOf(param.length()));

            conn.setDoOutput(true);
            conn.setDoInput(true);
            out = new OutputStreamWriter(conn.getOutputStream(),DEFAULT_UNICODE);
            out.write(param);
            out.flush();
            read = new BufferedReader(new InputStreamReader(conn.getInputStream(),DEFAULT_UNICODE));
            String line;
            while ((line = read.readLine()) != null) {
                result += line;
            }
            ```
问题：完成功能后测试时发现出现结果如下：{"timestamp":"2019-05-16 21:36:11","status":200,"error":"OK","exception":"org.springframework.web.HttpRequestMethodNotSupportedException","message":"Request method 'POST' not supported","path":"/gidQuery"}

百思不得其解，度娘了相关错误，指出了很多可能引起该问题的原因。

后发现问题出现在：
```
1、conn.setRequestMethod("POST");及
2、conn.setDoOutput(true);
            conn.setDoInput(true);
            out = new 3、OutputStreamWriter(conn.getOutputStream(),DEFAULT_UNICODE);
            out.write(param);
            out.flush();
```
删除2、3这几行代码后，同时1改为conn.setRequestMethod("GET")，执行正常。
