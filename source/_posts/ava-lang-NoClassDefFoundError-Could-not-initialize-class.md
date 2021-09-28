title: 'java.lang.NoClassDefFoundError: Could not initialize class'
author: Xiang Chuang
tags:
  - NoClassDefFoundError
  - 类加载异常
categories:
  - 爱学爱问
date: 2019-06-05 20:42:00
---
问题描述：项目启动没问题，运行也没有问题，但是当运行到某个静态工具类的时候，出问题了，且没有异常， 抛出的是NoClassDefFoundError

经分析：原因为静态类第一次被使用，尚未加载到虚拟机，使用时加载并编译，但static块中代码有异常，导致类加载失败，从而抛出此Error

```
public class LogTool {
	
	public static boolean isPrint = true;
	static {
		if (System.getProperty("spring.profiles.active").equals("local")
			|| System.getProperty("spring.profiles.active").equals("pdev")
			|| System.getProperty("spring.profiles.active").equals("sdev")) {
			isPrint = true;
		}
	}
```