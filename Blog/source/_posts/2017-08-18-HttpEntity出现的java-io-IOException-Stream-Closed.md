---
title: 'HttpEntity出现的java.io.IOException:Stream Closed'
date: 2017-08-18 10:24:18
tags: fix bug
categories: 日常点滴
---

&emsp;&emsp;最近在开发调试过程中出现了一个java.io.IOException:Stream Closed的错误，而这个错误隐藏的比较好，因为它不涉及到主要的流程，只是在通知邮件的时候出现了这么一个异常，所以直到测试人员介入的时候才发现了这一个隐藏的IOException。
<!--more-->

{% asset_img stream_closed.JPG stream_closed %}

下面就来看看这个异常是怎么产生的：
```
	for (int i = 0; i < batch; i++) {
                List<HashMap<String, String>> sub = performances.subList(100 * i, Math.min(100 * (i + 1), performances.size()));
                HashMap<String, Object> json = new HashMap<>();
                json.put("userId", user);
                json.put("sign", sign);
                json.put("type", "year");
                json.put("list", sub);
                CloseableHttpResponse response = null;
                try {
                    response = HttpClientUtil.postJson(url, JSON.toJSONString(json), 30 * 1000, 30 * 1000, 30 * 1000);
                    if (response != null && response.getStatusLine().getStatusCode() == HttpStatus.SC_OK) {                   
                        JSONObject jsonObject = JSON.parseObject(EntityUtils.toString(response.getEntity(), Charset.forName("UTF-8")));
                        String code = jsonObject.getString("code");
                        List<Integer> subIds = ids.subList(100 * i, Math.min(100 * (i + 1), performances.size()));
                        if (!"200".equals(code)) {
                            sendFailMail(subIds, "",返回code非200",response != null ? EntityUtils.toString(response.getEntity(), Charset.forName("UTF-8")) : "空");
                        }
                    } else {
                        List<Integer> subIds = ids.subList(100 * i, Math.min(100 * (i + 1), performances.size()));
                        sendFailMail(subIds, "",response != null ? EntityUtils.toString(response.getEntity(), Charset.forName("UTF-8")) : "空");
                    }
                } catch (Exception e) {
                    log.error("", e);
                    List<Integer> subIds = ids.subList(100 * i, Math.min(100 * (i + 1), performances.size()));
                    sendFailMail(subIds, "", response != null ? EntityUtils.toString(response.getEntity(), Charset.forName("UTF-8")) : "空");
                    throw e;
                } finally {
                    if (response != null) {
                        response.close();
                    }
                }
	}


```

当通过打断点进行定位的时候发现这个异常出现在这个循环体中，再进一步定位发现问题出现在：
```
 if (!"200".equals(code)) 
 {
	sendFailMail(subIds, "",返回code非200",response != null ? EntityUtils.toString(response.getEntity(), Charset.forName("UTF-8")) : "空");
 }
```

于是进入到EntityUtils.toString(response.getEntity(), Charset.forName("UTF-8"))这个方法中去看EntityUtils方法的实现,这是httpcore-4.4.4.jar的下org.apache.http.util.EntityUtils.class方法
```
public static String toString(HttpEntity entity, Charset defaultCharset) throws IOException, ParseException {
        Args.notNull(entity, "Entity");
        InputStream instream = entity.getContent();
        if(instream == null) {
            return null;
        } else {
            try {
                Args.check(entity.getContentLength() <= 2147483647L, "HTTP entity too large to be buffered in memory");
                int i = (int)entity.getContentLength();
                if(i < 0) {
                    i = 4096;
                }

                Charset charset = null;

                try {
                    ContentType contentType = ContentType.get(entity);
                    if(contentType != null) {
                        charset = contentType.getCharset();
                    }
                } catch (UnsupportedCharsetException var13) {
                    if(defaultCharset == null) {
                        throw new UnsupportedEncodingException(var13.getMessage());
                    }
                }

                if(charset == null) {
                    charset = defaultCharset;
                }

                if(charset == null) {
                    charset = HTTP.DEF_CONTENT_CHARSET;
                }

                Reader reader = new InputStreamReader(instream, charset);
                CharArrayBuffer buffer = new CharArrayBuffer(i);
                char[] tmp = new char[1024];

                int l;
                while((l = reader.read(tmp)) != -1) {
                    buffer.append(tmp, 0, l);
                }

                String var9 = buffer.toString();
                return var9;
            } finally {
                instream.close();
            }
        }
    }
```

可以看出确实是在这个方法中会抛出IOException，而且问题应该就出现在finally方法块的instream.close()上，既然改不了httpcore-4.4.4.jar下的文件，那么肯定是原先的代码中调用的方式不对。

回到源代码中发现这个方法在
```
 JSONObject jsonObject = JSON.parseObject(EntityUtils.toString(response.getEntity(), Charset.forName("UTF-8")));
```
调用过一次，而在
```
 if (!"200".equals(code)) 
 {
	sendFailMail(subIds, "",返回code非200",response != null ? EntityUtils.toString(response.getEntity(), Charset.forName("UTF-8")) : "空");
 }
```
又调用了一次。

接着去网上搜索了一下HttpEntity的这个方法

[HttpEntity Apache HttpCore 4.4.6 API]( https://hc.apache.org/httpcomponents-core-ga/httpcore/apidocs/org/apache/http/HttpEntity.html)


{% asset_img HttpEntity的getContent.JPG HttpEntity的getContent %}

在getContent()方法下有这么一句话
**
Returns a content stream of the entity. Repeatable entities are expected to create a new instance of InputStream for each invocation of this method and therefore can be consumed multiple times. Entities that are not repeatable are expected to return the same InputStream instance and therefore may not be consumed more than once.
**

翻译过来的含义是getContent()方法会返回内容流的一个实体。一个entity可以重复也就意味着通过调用这个方法创建一个新的读入流实例，它的content可以被多次读取，但如果一个entity不可重复，那么它通过调用这个方法只会返回同一个读入流实例，也不能被读取多次。

在这里response.getEntity()方法获得的entity是不可重复的，因此在调用了两次EntityUtils.toString()方法后会出现StreamClosed的错误。

改写后的代码如下,即可避免java.io.IOException:Stream Closed这个异常：

```
	for (int i = 0; i < batch; i++) {
                List<HashMap<String, String>> sub = performances.subList(100 * i, Math.min(100 * (i + 1), performances.size()));
                HashMap<String, Object> json = new HashMap<>();
                json.put("userId", user);
                json.put("sign", sign);
                json.put("type", "year");
                json.put("list", sub);
                CloseableHttpResponse response = null;
                try {
                    response = HttpClientUtil.postJson(url, JSON.toJSONString(json), 30 * 1000, 30 * 1000, 30 * 1000);
                    if (response != null && response.getStatusLine().getStatusCode() == HttpStatus.SC_OK) { 
						String temp = EntityUtils.toString(response.getEntity(), Charset.forName("UTF-8"));
						JSONObject jsonObject = JSON.parseObject(temp);
                        String code = jsonObject.getString("code");
                        List<Integer> subIds = ids.subList(100 * i, Math.min(100 * (i + 1), performances.size()));
                        if (!"200".equals(code)) {
                            sendFailMail(subIds, "",返回code非200",response != null ? temp : "空");
                        }
                    } else {
                        List<Integer> subIds = ids.subList(100 * i, Math.min(100 * (i + 1), performances.size()));
                        sendFailMail(subIds, "",response != null ? EntityUtils.toString(response.getEntity(), Charset.forName("UTF-8")) : "空");
                    }
                } catch (Exception e) {
                    log.error("", e);
                    List<Integer> subIds = ids.subList(100 * i, Math.min(100 * (i + 1), performances.size()));
                    sendFailMail(subIds, "", response != null ? EntityUtils.toString(response.getEntity(), Charset.forName("UTF-8")) : "空");
                    throw e;
                } finally {
                    if (response != null) {
                        response.close();
                    }
                }
	}


```