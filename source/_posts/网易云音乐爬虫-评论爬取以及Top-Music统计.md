---
title: 网易云音乐爬虫--评论爬取以及Top Music统计
date: 2016-11-01 15:11:00
tags: [网易云音乐,crawler,爬虫,java]
category: 网络爬虫
---

   网易云云音乐评论十分有趣，于是就想写个爬虫爬取评论。但是不熟悉Python，就用java写了个。
   主要使用了HttpClient,，Jsoup， 队列， 线程， log4j，poi生成Excel保存结果， 书写过程中主要一个问题就是评论获取，网易对其进行了加密，进行好一番搜索才找到解决方法。爬取歌单数，top歌曲数都可以动态进行配置.

# 目录结构
![目录结构](http://img.blog.csdn.net/20161026170154772?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

# 主程序
	package personal.mario.main;  
	  
	import java.io.IOException;  
	import java.util.List;  
	import org.apache.http.client.ClientProtocolException;  
	import org.apache.log4j.Logger;  
	import org.apache.poi.hssf.usermodel.HSSFSheet;  
	import org.apache.poi.hssf.usermodel.HSSFWorkbook;  
	import personal.mario.bean.MusicCommentMessage;  
	import personal.mario.service.HtmlFetcherService;  
	import personal.mario.service.HtmlParserService;  
	import personal.mario.service.MusicListQueueService;  
	import personal.mario.service.MusicQueueService;  
	import personal.mario.service.TopMusicCalculateService;  
	import personal.mario.utils.Constants;  
	import personal.mario.utils.GenerateExcelUtils;  
	  
	/* 
	 * 主逻辑 
	 * author timeless.li 
	 * 2016-10-26 
	 * */  
	public class NetEaseCrawler implements Runnable {  
	      
	    private int totalMusicList = Constants.MUSIC_LIST_COUNT;  
	    private int limit = Constants.PER_PAGE;  
	    private int offset =Constants.OFFSET;  
	    private HSSFWorkbook commentMessageWorkbook = new HSSFWorkbook();  
	    private List<MusicCommentMessage> ms = null;  
	    private static Logger logger = Logger.getLogger(NetEaseCrawler.class);  
	      
	    @Override  
	    public void run() {  
	        try {  
	            //初始化待爬取的歌单URL队列  
	            initUncrawledMusicListQueue();  
	              
	            //记录所有爬取出来的歌曲数，包含重复歌曲  
	            int count = 0;  
	              
	            //歌曲信息Excel初始化  
	            HSSFSheet commentMessageSheet = GenerateExcelUtils.generateCommentMessageExcelInit(commentMessageWorkbook);  
	              
	            //开始根据歌单爬取  
	            while (!MusicListQueueService.isUncrawledMusicListEmpty()) {  
	                  
	                //填充待爬取歌曲队列  
	                fillUncrawledMusicQueue(MusicListQueueService.getTopMusicList());  
	                  
	                //歌曲队列为空就返回上层循环填充歌曲队列  
	                while (!MusicQueueService.isUncrawledMusicQueueEmpty()) {  
	                      
	                    //取出待爬取歌曲ID  
	                    String songId = MusicQueueService.getTopMusicUrl();  
	                      
	                    //判断是否已经爬取过  
	                    if (!MusicQueueService.isMusicCrawled(songId)) {  
	                        //获取到爬取结果，歌曲信息  
	                        MusicCommentMessage mcm = getCommentMessage(songId);  
	                          
	                        //判断是否加入Top歌曲列表  
	                        ms = TopMusicCalculateService.getTopMusic(mcm);  
	                          
	                        //向歌曲信息Excel插入数据  
	                        GenerateExcelUtils.generateCommentMessageExcelProcess(commentMessageWorkbook, commentMessageSheet, mcm, count);  
	                          
	                        //生成歌曲评论Excel  
	                        GenerateExcelUtils.generateCommentsExcel(mcm);  
	                          
	                        //加入已经爬取的队列，供以后查重判断  
	                        MusicQueueService.addCrawledMusic(songId);  
	                        count++;  
	                    }  
	                }  
	            }  
	              
	            //生成歌曲信息Excel  
	            GenerateExcelUtils.generateCommentMessageExcelWrite(commentMessageWorkbook);  
	              
	            //生成Top歌曲Excel  
	            GenerateExcelUtils.generateTopMusicExcel(ms);  
	              
	            logger.info("count : " + count);  
	              
	            //实际爬取的歌曲数，不包含重复  
	            logger.info("size : " + MusicQueueService.getCrawledMusicSize());  
	        } catch (Exception e) {  
	            e.printStackTrace();  
	        }  
	    }  
	      
	    /* 
	     * 循环请求获取所有歌单 
	     * */  
	    public void initUncrawledMusicListQueue() throws ClientProtocolException, IOException {  
	          
	        if (totalMusicList > limit) {   
	            int tmpLimit = limit;  
	            int tmpOffset = offset;  
	              
	            while (totalMusicList > tmpOffset) {  
	                  
	                String suffix = "limit=" + tmpLimit + "&offset=" + tmpOffset;  
	                tmpOffset += tmpLimit;  
	                  
	                if (tmpOffset + tmpLimit > totalMusicList) {  
	                    tmpLimit =  totalMusicList - tmpOffset;  
	                }  
	                  
	                HtmlParserService.parseAndSaveMusicListUrl(HtmlFetcherService.fetch(Constants.SOURCE_URL + suffix));  
	            }  
	        } else {  
	            String suffix = "limit=" + totalMusicList + "&offset=" + offset;  
	            HtmlParserService.parseAndSaveMusicListUrl(HtmlFetcherService.fetch(Constants.SOURCE_URL + suffix));  
	        }  
	    }  
	      
	    //填充要爬取的歌曲队列  
	    public void fillUncrawledMusicQueue(String musicListUrl) throws IOException {  
	        HtmlParserService.parseMusicListAndGetMusics(musicListUrl);  
	    }  
	      
	    //由于反爬的存在， 一旦被禁止爬取， 休眠几秒后再进行爬取  
	    public MusicCommentMessage getCommentMessage(String songId) {  
	        try {  
	            MusicCommentMessage mc = HtmlParserService.parseCommentMessage(songId);  
	              
	            if (mc == null) {  
	                logger.info("warining: be interceptted by net ease music server..");  
	                Thread.sleep((long) (Math.random() * 30000));  
	                  
	                //递归  
	                return getCommentMessage(songId);  
	            } else {  
	                return mc;  
	            }  
	        } catch (Exception e) {  
	            logger.info("error: be refused by net ease music server..");  
	            return getCommentMessage(songId);  
	        }  
	    }  
	}

# 计算Top歌曲 
    package personal.mario.service;  
      
    import java.util.ArrayList;  
    import java.util.List;  
    import personal.mario.bean.MusicCommentMessage;  
    import personal.mario.utils.Constants;  
      
    /*计算获取TOP 歌曲*/  
    public class TopMusicCalculateService {  
        private static List<MusicCommentMessage> ms = new ArrayList<MusicCommentMessage>();  
          
        public static List<MusicCommentMessage> getTopMusic(MusicCommentMessage mcm) {  
      
            int topSize = ms.size();  
              
            if (topSize == 0) {  
                ms.add(mcm);  
            }  
              
            if (topSize > 0 && topSize < Constants.TOP_MUSIC_COUNT) {  
                for (int j = 0; j < topSize; j++) {  
                    if (mcm.getCommentCount() > ms.get(j).getCommentCount()) {  
                        ms.add(j, mcm);  
                        break;  
                    }  
                      
                    if (j == topSize - 1) {  
                        ms.add(mcm);  
                    }  
                }  
            }  
              
            if (topSize >= Constants.TOP_MUSIC_COUNT) {  
                for (int j = 0; j < topSize; j++) {  
                    if (mcm.getCommentCount() > ms.get(j).getCommentCount()) {  
                        ms.add(j, mcm);  
                        ms.remove(topSize);  
                        break;  
                    }  
                }  
            }  
              
            return ms;  
        }  
    } 

# 生成评论Excel表 
    //歌曲评论Excel生成  
        public static void generateCommentsExcel(MusicCommentMessage musicCommentMessage) throws IOException {  
              
            HSSFWorkbook workbook = new HSSFWorkbook();  
            HSSFSheet sheet = workbook.createSheet("歌曲评论");  
            sheet.setDefaultColumnWidth(15);  
              
            HSSFRow rowHead = sheet.createRow(0);  
              
            HSSFCellStyle style = workbook.createCellStyle();  
            style.setAlignment(HSSFCellStyle.ALIGN_CENTER);  
              
            HSSFFont font = workbook.createFont();  
            font.setColor(HSSFColor.LIGHT_BLUE.index);  
            font.setFontHeightInPoints((short) 8);  
            font.setBoldweight(HSSFFont.BOLDWEIGHT_BOLD);  
            style.setFont(font);  
      
            HSSFCell cellHead = rowHead.createCell(0);  
            cellHead.setCellValue("歌名");  
            cellHead.setCellStyle(style);  
              
            cellHead = rowHead.createCell(1);  
            cellHead.setCellValue("评论类型");  
            cellHead.setCellStyle(style);  
              
            cellHead = rowHead.createCell(2);  
            cellHead.setCellValue("评论用户昵称");  
            cellHead.setCellStyle(style);  
              
            cellHead = rowHead.createCell(3);  
            cellHead.setCellValue("评论时间");  
            cellHead.setCellStyle(style);  
              
            cellHead = rowHead.createCell(4);  
            cellHead.setCellValue("评论内容");  
            cellHead.setCellStyle(style);  
              
            cellHead = rowHead.createCell(5);  
            cellHead.setCellValue("获赞数");  
            cellHead.setCellStyle(style);  
              
            HSSFCellStyle cellStyle = workbook.createCellStyle();  
            cellStyle.setAlignment(HSSFCellStyle.ALIGN_CENTER);  
              
            List<MusicComment> comments = musicCommentMessage.getComments();  
              
            for (int i = 0; i < comments.size(); i++) {  
                MusicComment comment = comments.get(i);  
                HSSFRow row = sheet.createRow(i + 1);  
                          
                HSSFCell cell = row.createCell(0);  
                cell.setCellValue(musicCommentMessage.getSongTitle());  
                cell.setCellStyle(cellStyle);  
                  
                cell = row.createCell(1);  
                cell.setCellValue(comment.getType());  
                cell.setCellStyle(cellStyle);  
                  
                cell = row.createCell(2);  
                cell.setCellValue(comment.getNickname());  
                cell.setCellStyle(cellStyle);  
                  
                cell = row.createCell(3);  
                cell.setCellValue(comment.getCommentDate());  
                cell.setCellStyle(cellStyle);  
                  
                cell = row.createCell(4);  
                cell.setCellValue(comment.getContent());  
                cell.setCellStyle(cellStyle);  
                  
                cell = row.createCell(5);  
                cell.setCellValue(comment.getAppreciation());  
                cell.setCellStyle(cellStyle);  
            }  
              
            String path = Constants.COMMENTS_PATH + StringUtils.dealWithFilename(musicCommentMessage.getSongTitle()) + Constants.COMMENTS_SUFFIX;  
            logger.info(path);  
            FileOutputStream fos = new FileOutputStream(path);  
            workbook.write(fos);  
            fos.close();  
        }

# 歌曲队列
    package personal.mario.service;  
      
    import java.util.Queue;  
    import java.util.concurrent.ConcurrentLinkedQueue;  
      
    /*歌曲队列*/  
    public class MusicQueueService {  
        private static Queue<String> uncrawledMusics = new ConcurrentLinkedQueue<String>();  
        private static Queue<String> crawledMusics = new ConcurrentLinkedQueue<String>();  
          
        public static void addUncrawledMusic(String e) {  
            uncrawledMusics.offer(e);  
        }  
          
        public static String getTopMusicUrl() {  
            if (!uncrawledMusics.isEmpty()) {  
                return uncrawledMusics.poll();  
            }  
              
            return null;  
        }  
          
        public static void addCrawledMusic(String e) {  
            crawledMusics.offer(e);  
        }  
          
        public static boolean isMusicCrawled(String id) {  
            return crawledMusics.contains(id);  
        }  
          
        public static boolean isUncrawledMusicQueueEmpty() {  
            return uncrawledMusics.isEmpty();  
        }  
          
        public static void printAll() {  
            while (!uncrawledMusics.isEmpty()) {  
                System.out.println(uncrawledMusics.poll());  
            }  
        }  
          
        public static int getCrawledMusicSize() {  
            return crawledMusics.size();  
        }  
    }  

# 爬取结果
![result](http://img.blog.csdn.net/20161026170849842?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![result](http://img.blog.csdn.net/20161026170918745?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![result](http://img.blog.csdn.net/20161026171026667?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![result](http://img.blog.csdn.net/20161026171546982?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![result](http://img.blog.csdn.net/20161026171654978?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)