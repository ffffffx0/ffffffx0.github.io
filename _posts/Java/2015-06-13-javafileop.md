---
title: Java File OP
tagline: ""
category : Java
layout: post
tags : [Java, File, tools]
---


#### 读取CSV文件

依赖

```
<dependency>
	<groupId>net.sf.opencsv</groupId>
	<artifactId>opencsv</artifactId>
	<version>2.3</version>
</dependency>
```
读取文件

```
 public static void main(String[] args)  {
  	String name = "C:/Users/Desktop/suggest.csv";
  	 Reader inputStreamReader;
	try {
		inputStreamReader = new InputStreamReader(new FileInputStream(new java.io.File(name)));
		 CSVReader reader = new CSVReader(inputStreamReader); 
		   String[] nextLine;  
		  while (( nextLine = reader.readNext()) != null)  {
			  for (int i = 0; i < nextLine.length; i++) {
				System.out.println(nextLine[i]);
			}
		  }
	} catch (FileNotFoundException e) {
		e.printStackTrace();
	}catch (IOException e) {
		e.printStackTrace();
	}
}
```
