title: 图片压缩-ImageIO.read(inputStream)
author: Xiang Chuang
tags:
  - 图片压缩
  - ImageIO
categories:
  - 爱学爱问
date: 2019-04-11 18:36:00
---
背景：由于目前手机像素越来越富裕，导致用户上传图片时过大3-4M甚至以上，且像素较高，如3024*4032，因此，图片压缩功能就势在必行了。

选型：优先考虑前端压缩，但由于个人对前端无甚兴趣，因此提供了基于java的压缩解决方案。

目前市面上成熟的压缩工具大致为thumbnailator及simpleimage
1、thumbnailator
```
	<dependency>
			<groupId>net.coobird</groupId>
			<artifactId>thumbnailator</artifactId>
			<version>0.4.8</version>
  </dependency>
  ```
  ```
  BufferedImage bufferedImage = Thumbnails.of(input).scale(scale).outputQuality(quality).asBufferedImage();
  ```
  经过使用发现有背景色变红的问题，未着手解决
  
2、simpleimage
参加git
https://github.com/alibaba/simpleimage

由于simpleimage尝试过程中，存在识别不了图片类型的问题（本人操作系统为Win10），故未深究

3、由于上述2个方案存在的问题及本身需求只是缩放图片而且，也用不上它们提供的水印之类的方法，因此，自定义了一个仅支持图片压缩能力的工具包
```
public final static ByteArrayOutputStream scale(InputStream srcImageStream, float scale) throws IOException {
        ByteArrayOutputStream desImageStream = new ByteArrayOutputStream();
        BufferedImage originalImage = ImageIO.read(srcImageStream);
        int width = Float.valueOf(scale * originalImage.getWidth()).intValue();
        int height = Float.valueOf(scale * originalImage.getHeight()).intValue();
        BufferedImage newImage = new BufferedImage(width, height, originalImage.getType());
        Graphics g = newImage.getGraphics();
        g.drawImage(originalImage, 0, 0, width, height, null);
        g.dispose();
        ImageIO.write(newImage, "jpeg", desImageStream);
        return desImageStream;
    }
 ```
   该方案能在压缩质量得到保证的前提的快速完成压缩，约1s，thumbnailator使用时，约2s。
    问题来了：功能上去后，发现内存占用率偶尔超过90%，但垃圾回收后又能绝大数回收，（采用G1回收，只额外配置了暂停时间50ms，堆最大值1536M）因此排除内存泄漏的可能，但功能上去之前未曾出现，初步猜测是图片压缩功能引起的（方案1、3都存在问题，推测2也存在），后经本地java visualVM调试发现， 执行一个大小3M多，像素3024*4032的图片压缩时，会有35M多的老年代被直接使用。因此分析问题在于出现了大对象，大对象会直接分配在老年代，于是通过断点测试结合 java visualVM监控，定位到问题在于：
    ```
  BufferedImage originalImage = ImageIO.read(srcImageStream);
    ```
    此方法创建BufferedImage时，会基于ComponentSampleModel类首先创建数据缓冲区（dataBuffer），而缓冲区大小算法为：
   ```
   private int getBufferSize() {
         int maxBandOff=bandOffsets[0];//jpeg图片，此数组为{2，1，0}
         for (int i=1; i<bandOffsets.length; i++) {
             maxBandOff = Math.max(maxBandOff,bandOffsets[i]);
         }
         if (maxBandOff < 0 || maxBandOff > (Integer.MAX_VALUE - 1)) {
             throw new IllegalArgumentException("Invalid band offset");
         }
         if (pixelStride < 0 || pixelStride > (Integer.MAX_VALUE / width)) {
             throw new IllegalArgumentException("Invalid pixel stride");
         }
         if (scanlineStride < 0 || scanlineStride > (Integer.MAX_VALUE / height)) {
             throw new IllegalArgumentException("Invalid scanline stride");
         }
         int size = maxBandOff + 1;
         int val = pixelStride * (width - 1);//pixelStride jpeg图片时，默认为3
         if (val > (Integer.MAX_VALUE - size)) {
             throw new IllegalArgumentException("Invalid pixel stride");
         }
         size += val;
         val = scanlineStride * (height - 1);//scanlineStride jpeg图片时，默认为12096
         if (val > (Integer.MAX_VALUE - size)) {
             throw new IllegalArgumentException("Invalid scan stride");
         }
         size += val;
         return size;
     }
    
    因此大小为(2+1)+3*(width-1)+12096*(height-1)  god，这是多大的一个内存呀
    public DataBufferByte(int size, int numBanks) {
        super(STABLE, TYPE_BYTE, size, numBanks);
        bankdata = new byte[numBanks][];
        for (int i= 0; i < numBanks; i++) {
            bankdata[i] = new byte[size];
        }
        data = bankdata[0];
    }
    
    ```
    如果整个逻辑处理时间是1s， 那么，可以算一下1s并发量多大，需要配置多大的内存了 ！！！