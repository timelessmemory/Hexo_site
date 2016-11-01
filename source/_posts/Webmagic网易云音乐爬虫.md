---
title: Webmagic网易云音乐爬虫
date: 2016-11-01 10:28:12
tags: [webmagic, java, 爬虫]
category: 网络爬虫
---

发现了一个java爬虫框架，熟悉了一下写了一个爬取网易云音乐专辑歌曲信息的爬虫，其中歌曲的歌词和评论是动态获取的，我只能单独自己写代码进行请求。

<!--more-->

代码如下
	import java.io.IOException;  
	import java.io.UnsupportedEncodingException;  
	import java.math.BigInteger;  
	import java.security.SecureRandom;  
	import java.text.SimpleDateFormat;  
	import java.util.ArrayList;  
	import java.util.Date;  
	import java.util.List;  
	import javax.crypto.Cipher;  
	import javax.crypto.spec.IvParameterSpec;  
	import javax.crypto.spec.SecretKeySpec;  
	import javax.xml.bind.DatatypeConverter;  
	import org.apache.commons.codec.binary.Base64;  
	import org.apache.http.HttpEntity;  
	import org.apache.http.NameValuePair;  
	import org.apache.http.client.entity.UrlEncodedFormEntity;  
	import org.apache.http.client.methods.CloseableHttpResponse;  
	import org.apache.http.client.methods.HttpPost;  
	import org.apache.http.impl.client.CloseableHttpClient;  
	import org.apache.http.impl.client.HttpClients;  
	import org.apache.http.message.BasicNameValuePair;  
	import org.apache.http.util.EntityUtils;  
	import com.alibaba.fastjson.JSON;  
	import com.alibaba.fastjson.JSONPath;  
	import us.codecraft.webmagic.Page;  
	import us.codecraft.webmagic.Site;  
	import us.codecraft.webmagic.Spider;  
	import us.codecraft.webmagic.pipeline.ConsolePipeline;  
	import us.codecraft.webmagic.pipeline.FilePipeline;  
	import us.codecraft.webmagic.processor.PageProcessor;  

	public class NetEaseMusicPageProcessor implements PageProcessor {  
      
	    //正则表达式\\. \\转义java中的\  \.转义正则中的.  
	      
	    //主域名  
	    public static final String BASE_URL = "http://music.163.com/";  
	  
	    //匹配专辑URL  
	    public static final String ALBUM_URL = "http://music\\.163\\.com/album\\?id=\\d+";  
	      
	    //匹配歌曲URL  
	    public static final String MUSIC_URL = "http://music\\.163\\.com/song\\?id=\\d+";  
	      
	    //初始地址, JAY_CHOU 周杰伦的床边故事专辑, 可以改为其他歌手专辑地址  
	    public static final String START_URL = "http://music.163.com/album?id=34720827";  
	      
	    //加密使用到的文本  
	    public static final String TEXT = "{\"username\": \"\", \"rememberLogin\": \"true\", \"password\": \"\"}";  
	      
	    //爬取结果保存文件路径  
	    public static final String RESULT_PATH = "/home/user/workspace/WebMagicCrawler/result/";  
	      
	    private Site site = Site.me()  
	                            .setDomain("http://music.163.com")  
	                            .setSleepTime(1000)  
	                            .setRetryTimes(30)  
	                            .setCharset("utf-8")  
	                            .setTimeOut(30000)  
	                            .setUserAgent("Mozilla/5.0 (Macintosh; Intel Mac OS X 10_7_2) AppleWebKit/537.31 (KHTML, like Gecko) Chrome/26.0.1410.65 Safari/537.31");  
	  
	    @Override  
	    public Site getSite() {  
	        return site;  
	    }  
	      
	    @Override  
	    public void process(Page page) {  
	        //根据URL判断页面类型  
	        if (page.getUrl().regex(ALBUM_URL).match()) {  
	            //爬取歌曲URl加入队列  
	            page.addTargetRequests(page.getHtml().xpath("//div[@id=\"song-list-pre-cache\"]").links().regex(MUSIC_URL).all());  
	              
	            //爬取专辑URL加入队列  
	            page.addTargetRequests(page.getHtml().xpath("//div[@class=\"cver u-cover u-cover-3\"]").links().regex(ALBUM_URL).all());  
	        } else {  
	            String url = page.getUrl().toString();  
	            page.putField("title", page.getHtml().xpath("//em[@class='f-ff2']/text()"));  
	            page.putField("author", page.getHtml().xpath("//p[@class='des s-fc4']/span/a/text()"));  
	            page.putField("album", page.getHtml().xpath("//p[@class='des s-fc4']/a/text()"));  
	            page.putField("URL", url);  
	              
	            //单独对AJAX请求获取评论数, 使用JSON解析返回结果  
	            page.putField("commentCount", JSONPath.eval(JSON.parse(crawlAjaxUrl(url.substring(url.indexOf("id=") + 3))), "$.total"));  
	        }  
	    }  
	      
	    public static void main(String[] args) {  
	        Spider.create(new NetEaseMusicPageProcessor())  
	        //初始URL  
	        .addUrl(START_URL)  
	        .addPipeline(new ConsolePipeline())  
	        //结果输出到文件  
	        .addPipeline(new FilePipeline(RESULT_PATH))  
	        .run();  
	    }  
	      
	    //对AJAX数据进行单独请求  
	    public String crawlAjaxUrl(String songId) {   
	        CloseableHttpClient httpclient = HttpClients.createDefault();  
	        CloseableHttpResponse response =null;  
	          
	        try {  
	            //参数加密  
	            String secKey = new BigInteger(100, new SecureRandom()).toString(32).substring(0, 16);  
	            String encText = aesEncrypt(aesEncrypt(TEXT, "0CoJUm6Qyw8W8jud"), secKey);  
	            String encSecKey = rsaEncrypt(secKey);  
	              
	            HttpPost httpPost = new HttpPost("http://music.163.com/weapi/v1/resource/comments/R_SO_4_" + songId + "/?csrf_token=");  
	            httpPost.addHeader("Referer", BASE_URL);  
	              
	            List<NameValuePair> ls = new ArrayList<NameValuePair>();  
	            ls.add(new BasicNameValuePair("params", encText));  
	            ls.add(new BasicNameValuePair("encSecKey", encSecKey));  
	              
	            UrlEncodedFormEntity paramEntity = new UrlEncodedFormEntity(ls, "utf-8");  
	            httpPost.setEntity(paramEntity);  
	              
	            response = httpclient.execute(httpPost);  
	            HttpEntity entity = response.getEntity();  
	              
	            if (entity != null) {  
	                return EntityUtils.toString(entity, "utf-8");  
	            }  
	              
	        } catch (Exception e) {  
	            e.printStackTrace();  
	        } finally {  
	            try {  
	                response.close();  
	                httpclient.close();  
	            } catch (IOException e) {  
	                e.printStackTrace();  
	            }  
	        }  
	          
	        return "";  
	    }  
	}
关于网易云音乐API加密可以参见我另外一篇博客。

下面是运行结果

![运行结果](http://img.blog.csdn.net/20161028113908400?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

[INFO] 2016-10-28 10:51:49 Spider http://music.163.com started!
[INFO] 2016-10-28 10:51:49 downloading page http://music.163.com/album?id=34720827
get page: http://music.163.com/album?id=34720827
[INFO] 2016-10-28 10:51:50 downloading page http://music.163.com/song?id=415792916
get page: http://music.163.com/song?id=415792916
title:    床边故事
author:    周杰伦
album:    周杰伦的床边故事
URL:    http://music.163.com/song?id=415792916
commentCount:    52535
[INFO] 2016-10-28 10:51:52 downloading page http://music.163.com/song?id=418602084
get page: http://music.163.com/song?id=418602084
title:    说走就走
author:    周杰伦
album:    周杰伦的床边故事
URL:    http://music.163.com/song?id=418602084
commentCount:    24256
[INFO] 2016-10-28 10:51:53 downloading page http://music.163.com/song?id=418603076
get page: http://music.163.com/song?id=418603076
title:    一点点
author:    周杰伦
album:    周杰伦的床边故事
URL:    http://music.163.com/song?id=418603076
commentCount:    33127
[INFO] 2016-10-28 10:51:54 downloading page http://music.163.com/song?id=415792918
get page: http://music.163.com/song?id=415792918
title:    前世情人
author:    周杰伦
album:    周杰伦的床边故事
URL:    http://music.163.com/song?id=415792918
commentCount:    26761
[INFO] 2016-10-28 10:51:55 downloading page http://music.163.com/song?id=418602085
get page: http://music.163.com/song?id=418602085
title:    英雄
author:    周杰伦
album:    周杰伦的床边故事
URL:    http://music.163.com/song?id=418602085
commentCount:    13321
[INFO] 2016-10-28 10:51:57 downloading page http://music.163.com/song?id=417250561
get page: http://music.163.com/song?id=417250561
title:    不该(with aMEI)
author:    周杰伦
album:    周杰伦的床边故事
URL:    http://music.163.com/song?id=417250561
commentCount:    37877
[INFO] 2016-10-28 10:51:58 downloading page http://music.163.com/song?id=418602086
get page: http://music.163.com/song?id=418602086
title:    土耳其冰淇淋
author:    周杰伦
album:    周杰伦的床边故事
URL:    http://music.163.com/song?id=418602086
commentCount:    21751
[INFO] 2016-10-28 10:51:59 downloading page http://music.163.com/song?id=418603077
get page: http://music.163.com/song?id=418603077
title:    告白气球
author:    周杰伦
album:    周杰伦的床边故事
URL:    http://music.163.com/song?id=418603077
commentCount:    117647
[INFO] 2016-10-28 10:52:00 downloading page http://music.163.com/song?id=417247652
get page: http://music.163.com/song?id=417247652
title:    Now You See Me
author:    周杰伦
album:    周杰伦的床边故事
URL:    http://music.163.com/song?id=417247652
commentCount:    17485
[INFO] 2016-10-28 10:52:01 downloading page http://music.163.com/song?id=418602087
get page: http://music.163.com/song?id=418602087
title:    爱情废柴
author:    周杰伦
album:    周杰伦的床边故事
URL:    http://music.163.com/song?id=418602087
commentCount:    24268
[INFO] 2016-10-28 10:52:02 downloading page http://music.163.com/album?id=34685590
get page: http://music.163.com/album?id=34685590
[INFO] 2016-10-28 10:52:04 downloading page http://music.163.com/album?id=34588039
get page: http://music.163.com/album?id=34588039
[INFO] 2016-10-28 10:52:05 downloading page http://music.163.com/album?id=3084335
get page: http://music.163.com/album?id=3084335
[INFO] 2016-10-28 10:52:06 downloading page http://music.163.com/album?id=2662137
get page: http://music.163.com/album?id=2662137
[INFO] 2016-10-28 10:52:07 downloading page http://music.163.com/song?id=411921852
get page: http://music.163.com/song?id=411921852
title:    惊叹号(Live)
author:    周杰伦
album:    魔天伦 世界巡回演唱会
URL:    http://music.163.com/song?id=411921852
commentCount:    16768
[INFO] 2016-10-28 10:52:08 downloading page http://music.163.com/song?id=412327297
get page: http://music.163.com/song?id=412327297
title:    龙拳 (Live)
author:    周杰伦
album:    魔天伦 世界巡回演唱会
URL:    http://music.163.com/song?id=412327297
commentCount:    2563
[INFO] 2016-10-28 10:52:09 downloading page http://music.163.com/song?id=412327298
get page: http://music.163.com/song?id=412327298
title:    最后的战役 (Live)
author:    周杰伦
album:    魔天伦 世界巡回演唱会
URL:    http://music.163.com/song?id=412327298
commentCount:    3849
[INFO] 2016-10-28 10:52:11 downloading page http://music.163.com/song?id=412327299
get page: http://music.163.com/song?id=412327299
title:    天台 (Live)
author:    周杰伦
album:    魔天伦 世界巡回演唱会
URL:    http://music.163.com/song?id=412327299
commentCount:    1283
[INFO] 2016-10-28 10:52:12 downloading page http://music.163.com/song?id=412327300
get page: http://music.163.com/song?id=412327300
title:    比较大的大提琴 (Live)
author:    周杰伦
album:    魔天伦 世界巡回演唱会
URL:    http://music.163.com/song?id=412327300
commentCount:    1223
[INFO] 2016-10-28 10:52:13 downloading page http://music.163.com/song?id=412327301
get page: http://music.163.com/song?id=412327301
title:    快门慢舞 (Live)
author:    周杰伦
album:    魔天伦 世界巡回演唱会
URL:    http://music.163.com/song?id=412327301
commentCount:    1287
[INFO] 2016-10-28 10:52:14 downloading page http://music.163.com/song?id=412327302
get page: http://music.163.com/song?id=412327302
title:    打架舞 (Live)
author:    周杰伦
album:    魔天伦 世界巡回演唱会
URL:    http://music.163.com/song?id=412327302
commentCount:    null
[INFO] 2016-10-28 10:52:15 downloading page http://music.163.com/song?id=412327303
get page: http://music.163.com/song?id=412327303
title:    哪里都是你 (Live)
author:    周杰伦
album:    魔天伦 世界巡回演唱会
URL:    http://music.163.com/song?id=412327303
commentCount:    2674
[INFO] 2016-10-28 10:52:16 downloading page http://music.163.com/song?id=412327304
get page: http://music.163.com/song?id=412327304
title:    一路向北 (Live)
author:    周杰伦
album:    魔天伦 世界巡回演唱会
URL:    http://music.163.com/song?id=412327304
commentCount:    11502
[INFO] 2016-10-28 10:52:17 downloading page http://music.163.com/song?id=412327305
get page: http://music.163.com/song?id=412327305
title:    不能说的秘密 (Live)
author:    周杰伦
album:    魔天伦 世界巡回演唱会
URL:    http://music.163.com/song?id=412327305
commentCount:    3595
[INFO] 2016-10-28 10:52:18 downloading page http://music.163.com/song?id=412327306
get page: http://music.163.com/song?id=412327306
title:    双截棍 (Live)
author:    周杰伦
album:    魔天伦 世界巡回演唱会
URL:    http://music.163.com/song?id=412327306
commentCount:    1892
[INFO] 2016-10-28 10:52:20 downloading page http://music.163.com/song?id=412327307
get page: http://music.163.com/song?id=412327307
title:    明明就 (Live)
author:    周杰伦
album:    魔天伦 世界巡回演唱会
URL:    http://music.163.com/song?id=412327307
commentCount:    1898
[INFO] 2016-10-28 10:52:21 downloading page http://music.163.com/song?id=412327308
get page: http://music.163.com/song?id=412327308
title:    Mine Mine (Live)
author:    周杰伦
album:    魔天伦 世界巡回演唱会
URL:    http://music.163.com/song?id=412327308
commentCount:    1957
[INFO] 2016-10-28 10:52:22 downloading page http://music.163.com/song?id=412327309
get page: http://music.163.com/song?id=412327309
title:    龙卷风 (Live)
author:    周杰伦
album:    魔天伦 世界巡回演唱会
URL:    http://music.163.com/song?id=412327309
commentCount:    1923
[INFO] 2016-10-28 10:52:23 downloading page http://music.163.com/song?id=412327310
get page: http://music.163.com/song?id=412327310
title:    公公偏头痛 (Live)
author:    周杰伦
album:    魔天伦 世界巡回演唱会
URL:    http://music.163.com/song?id=412327310
commentCount:    1348
[INFO] 2016-10-28 10:52:24 downloading page http://music.163.com/song?id=412327311
get page: http://music.163.com/song?id=412327311
title:    青花瓷 (Live)
author:    周杰伦
album:    魔天伦 世界巡回演唱会
URL:    http://music.163.com/song?id=412327311
commentCount:    1916
[INFO] 2016-10-28 10:52:25 downloading page http://music.163.com/song?id=412327312
get page: http://music.163.com/song?id=412327312
title:    斗牛 / 水手怕水 / 大笨钟 (Live)
author:    周杰伦
album:    魔天伦 世界巡回演唱会
URL:    http://music.163.com/song?id=412327312
commentCount:    1595
[INFO] 2016-10-28 10:52:27 downloading page http://music.163.com/song?id=412327313
get page: http://music.163.com/song?id=412327313
title:    彩虹 / 轨迹 (Live)
author:    周杰伦
album:    魔天伦 世界巡回演唱会
URL:    http://music.163.com/song?id=412327313
commentCount:    3209
[INFO] 2016-10-28 10:52:28 downloading page http://music.163.com/song?id=412327314
get page: http://music.163.com/song?id=412327314
title:    手语 (Live)
author:    周杰伦
album:    魔天伦 世界巡回演唱会
URL:    http://music.163.com/song?id=412327314
commentCount:    1403
[INFO] 2016-10-28 10:52:29 downloading page http://music.163.com/song?id=412327315
get page: http://music.163.com/song?id=412327315
title:    开不了口 (Live)
author:    周杰伦
album:    魔天伦 世界巡回演唱会
URL:    http://music.163.com/song?id=412327315
commentCount:    3067
[INFO] 2016-10-28 10:52:30 downloading page http://music.163.com/song?id=412327316
get page: http://music.163.com/song?id=412327316
title:    乌克丽丽 (Live)
author:    周杰伦
album:    魔天伦 世界巡回演唱会
URL:    http://music.163.com/song?id=412327316
commentCount:    1664
[INFO] 2016-10-28 10:52:32 downloading page http://music.163.com/song?id=412327317
get page: http://music.163.com/song?id=412327317
title:    阳光宅男 (Live)
author:    周杰伦
album:    魔天伦 世界巡回演唱会
URL:    http://music.163.com/song?id=412327317
commentCount:    2018
[INFO] 2016-10-28 10:52:33 downloading page http://music.163.com/song?id=407679169
get page: http://music.163.com/song?id=407679169
title:    英雄
author:    周杰伦
album:    英雄
URL:    http://music.163.com/song?id=407679169
commentCount:    72800
[INFO] 2016-10-28 10:52:34 downloading page http://music.163.com/song?id=29822016
get page: http://music.163.com/song?id=29822016
title:    阳明山
author:    周杰伦
album:    哎呦，不错哦
URL:    http://music.163.com/song?id=29822016
commentCount:    3858
[INFO] 2016-10-28 10:52:35 downloading page http://music.163.com/song?id=29822010
get page: http://music.163.com/song?id=29822010
title:    窃爱
author:    周杰伦
album:    哎呦，不错哦
URL:    http://music.163.com/song?id=29822010
commentCount:    5511
[INFO] 2016-10-28 10:52:36 downloading page http://music.163.com/song?id=29818120
get page: http://music.163.com/song?id=29818120
title:    算什么男人
author:    周杰伦
album:    哎呦，不错哦
URL:    http://music.163.com/song?id=29818120
commentCount:    24813
[INFO] 2016-10-28 10:52:37 downloading page http://music.163.com/song?id=29822015
get page: http://music.163.com/song?id=29822015
title:    天涯过客
author:    周杰伦
album:    哎呦，不错哦
URL:    http://music.163.com/song?id=29822015
commentCount:    10229
[INFO] 2016-10-28 10:52:39 downloading page http://music.163.com/song?id=29822033
get page: http://music.163.com/song?id=29822033
title:    怎么了
author:    周杰伦
album:    哎呦，不错哦
URL:    http://music.163.com/song?id=29822033
commentCount:    5023
[INFO] 2016-10-28 10:52:40 downloading page http://music.163.com/song?id=29822032
get page: http://music.163.com/song?id=29822032
title:    一口气全念对
author:    周杰伦
album:    哎呦，不错哦
URL:    http://music.163.com/song?id=29822032
commentCount:    5846
[INFO] 2016-10-28 10:52:41 downloading page http://music.163.com/song?id=29822017
get page: http://music.163.com/song?id=29822017
title:    我要夏天
author:    周杰伦
album:    哎呦，不错哦
URL:    http://music.163.com/song?id=29822017
commentCount:    5938
[INFO] 2016-10-28 10:52:42 downloading page http://music.163.com/song?id=29822018
get page: http://music.163.com/song?id=29822018
title:    手写的从前
author:    周杰伦
album:    哎呦，不错哦
URL:    http://music.163.com/song?id=29822018
commentCount:    22051
[INFO] 2016-10-28 10:52:47 downloading page http://music.163.com/song?id=29818117
get page: http://music.163.com/song?id=29818117
title:    鞋子特大号
author:    周杰伦
album:    哎呦，不错哦
URL:    http://music.163.com/song?id=29818117
commentCount:    null
[INFO] 2016-10-28 10:52:48 downloading page http://music.163.com/song?id=29822013
get page: http://music.163.com/song?id=29822013
title:    听爸爸的话
author:    周杰伦
album:    哎呦，不错哦
URL:    http://music.163.com/song?id=29822013
commentCount:    null
[INFO] 2016-10-28 10:52:49 downloading page http://music.163.com/song?id=29822012
get page: http://music.163.com/song?id=29822012
title:    美人鱼
author:    周杰伦
album:    哎呦，不错哦
URL:    http://music.163.com/song?id=29822012
commentCount:    10662
[INFO] 2016-10-28 10:52:50 downloading page http://music.163.com/song?id=29822014
get page: http://music.163.com/song?id=29822014
title:    听见下雨的声音
author:    周杰伦
album:    哎呦，不错哦
URL:    http://music.163.com/song?id=29822014
commentCount:    17453
[INFO] 2016-10-28 10:52:51 downloading page http://music.163.com/song?id=27698922
get page: http://music.163.com/song?id=27698922
title:    你怎么说+红尘客栈+千里之外
author:    周杰伦
album:    周杰伦2013《魔天伦》台北小巨蛋演唱会
URL:    http://music.163.com/song?id=27698922
commentCount:    1915



