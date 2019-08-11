---
title: fastDFS与java整合文件上传下载
date: 2017-07-20 10:08:03
tags: [fastDFS,java]
categories: 学习笔记
---

  

### 准备
1. 下载``fastdfs-client-java``源码

<a rel="external nofollow" target="_blank" href="http://pan.baidu.com/s/1i5QQXcL">源码地址</a> 密码：`s3sw`

<!--more-->

2. 修改`pom.xml`  
 **第一个plugins是必需要的，是maven用来编译的插件，第二个是maven打源码包的，可以不要。**
```xml
<build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.5.1</version>
        <configuration>
          <encoding>UTF-8</encoding>
          <source>${jdk.version}</source>
          <target>${jdk.version}</target>
        </configuration>
      </plugin>
      <plugin>
          <groupId>org.apache.maven.plugins</groupId>
          <artifactId>maven-source-plugin</artifactId>
          <version>3.0.1</version>
          <executions>
              <execution>
                  <id>attach-sources</id>
                  <goals>
                      <goal>jar</goal>
                  </goals>
              </execution>
          </executions>
      </plugin>
    </plugins>
</build>
```

3. 将 ``fastdfs-client-java`` 打包  
  直接项目右键，run as maven install  
  install成功后，fastdfs-client-java就成功的被安装到本地仓库了。
--- 
### 编写工具类：
* 把``fdfs_client.conf``文件复制一份放到自己项目的resource下面;修改里面的``tracker.server``,其它的都不用动：  
* 在项目的``pom.xml``中添加依赖
```xml
<dependency>
	<groupId>org.csource</groupId>
	<artifactId>fastdfs-client-java</artifactId>
	<version>1.27-SNAPSHOT</version>
</dependency>
```
  
* 首先来实现文件上传，``fastdfs-client-java``的上传是通过传入一个byte[ ]来完成的，简单看一下源码：
```java
public String[] upload_file(byte[] file_buff, String file_ext_name, 
           NameValuePair[] meta_list) throws IOException, MyException{
    final String group_name = null;
    return this.upload_file(group_name, file_buff, 0, file_buff.length, file_ext_name, meta_list);
}
```


#### 文件属性
```java
package com.wuwii.utils;

import java.io.Serializable;
import java.util.Arrays;

/**
 * @ClassName FastDFSFile
 * @Description FastDFS上传文件业务对象
 * @author zhangkai
 * @date 2017年7月18日
 */
public class FastDFSFile implements Serializable{

	private static final long serialVersionUID = 2637755431406080379L;
	/**
	 * 文件二进制
	 */
	private byte[] content;
	/**
	 * 文件名称
	 */
	private String name;
	/**
	 * 文件长度
	 */
	private Long size;
	
	public FastDFSFile(){
		
	}
	public FastDFSFile(byte[] content, String name, Long size){
		this.content = content;
		this.name = name;
		this.size = size;
	}
	
	public byte[] getContent() {
		return content;
	}
	public void setContent(byte[] content) {
		this.content = content;
	}
	public String getName() {
		return name;
	}
	public void setName(String name) {
		this.name = name;
	}
	public Long getSize() {
		return size;
	}
	public void setSize(Long size) {
		this.size = size;
	}
	public static long getSerialversionuid() {
		return serialVersionUID;
	}
}


```
#### 编写FastDFS工具类
```java
package com.wuwii.utils;

import java.io.Serializable;

import org.apache.commons.io.FilenameUtils;
import org.csource.common.NameValuePair;
import org.csource.fastdfs.ClientGlobal;
import org.csource.fastdfs.StorageClient1;
import org.csource.fastdfs.TrackerClient;
import org.csource.fastdfs.TrackerServer;
import org.jetbrains.annotations.NotNull;
import org.springframework.core.io.ClassPathResource;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;

/**
 * @ClassName FastDFSUtils
 * @Description FastDFS工具类
 * @author zhangkai
 * @date 2017年7月18日
 */
public class FastDFSUtils implements Serializable{
	/**
	 * 
	 */
	private static final long serialVersionUID = -4462272673174266738L;
	private static TrackerClient trackerClient;
    private static TrackerServer trackerServer;
    private static StorageClient1 storageClient1;
    
    static {
        try {
        	//clientGloble读配置文件
        	ClassPathResource resource = new ClassPathResource("fdfs_client.conf");
        	ClientGlobal.init(resource.getClassLoader().getResource("fdfs_client.conf").getPath());
			//trackerclient
        	trackerClient = new TrackerClient();
			trackerServer = trackerClient.getConnection();
			//storageclient
			storageClient1 = new StorageClient1(trackerServer,null); 
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
	
	/**
	 * fastDFS文件上传
	 * @param file 上传的文件 FastDFSFile
	 * @return String 返回文件的绝对路径
	 */
	public static String uploadFile(FastDFSFile file){
		String path = null;
		try {
			//文件扩展名
			String ext = FilenameUtils.getExtension(file.getName());
			//mata list是表文件的描述
			NameValuePair[] mata_list = new NameValuePair[3];
			mata_list[0] = new NameValuePair("fileName",file.getName());
			mata_list[1] = new NameValuePair("fileExt",ext);
			mata_list[2] = new NameValuePair("fileSize",String.valueOf(file.getSize()));
			path = storageClient1.upload_file1(file.getContent(), ext, mata_list);
		} catch (Exception e) {
			e.printStackTrace();
		} 
		return path;
	}
	
	/**
	 * fastDFS文件下载
	 * @param groupName 组名
	 * @param remoteFileName 文件名
	 * @param specFileName 真实文件名
	 * @return ResponseEntity<byte[]>
	 */
	@org.jetbrains.annotations.NotNull
	public static ResponseEntity<byte[]> downloadFile(String groupName, String remoteFileName, String specFileName){
		byte[] content = null;
	    HttpHeaders headers = new HttpHeaders();
	    try {
	        content = storageClient1.download_file(groupName, remoteFileName);
	        headers.setContentDispositionFormData("attachment",  new String(specFileName.getBytes("UTF-8"),"iso-8859-1"));
	        headers.setContentType(MediaType.APPLICATION_OCTET_STREAM);
	    } catch (Exception e) {
	        e.printStackTrace();
	    }
	    return new ResponseEntity<byte[]>(content, headers, HttpStatus.CREATED);
	}
	
	/**
	 * 根据fastDFS返回的path得到文件的组名
	 * @param path fastDFS返回的path
	 * @return
	 */
	public static String getGroupFormFilePath(String path){
		return path.split("/")[0];
	}
	
	/**
	 * 根据fastDFS返回的path得到文件名
	 * @param path fastDFS返回的path
	 * @return
	 */
	@NotNull
	public static String getFileNameFormFilePath(String path) {
		return path.substring(path.indexOf("/")+1);
	}
}


```
#### 测试Controller
```java
package com.wuwii.controller;

import java.io.IOException;

import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.multipart.MultipartFile;

import com.wuwii.utils.FastDFSFile;
import com.wuwii.utils.FastDFSUtils;
import com.wuwii.utils.PropertyUtil;

/**
 * FastFDS控制器
 * @author zhangkai
 *
 */
@Controller
@RequestMapping(value = "/fastdfs")
public class FastFDSController {
	 @RequestMapping(value = "/upload", method = RequestMethod.POST)
	public String upload (MultipartFile file){
		try {
			FastDFSFile fastDFSFile = new FastDFSFile(file.getBytes(), file.getOriginalFilename(), file.getSize());
			String path = FastDFSUtils.uploadFile(fastDFSFile);
			return "redirect:"+ PropertyUtil.get("fastFDS_server") + path;
		} catch (IOException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		return null;
	}
	 
	 @RequestMapping(value = "/download")
	 public ResponseEntity<byte[]> download (String path, String specFileName){
		 String filename = FastDFSUtils.getFileNameFormFilePath(path);
		 String group = FastDFSUtils.getGroupFormFilePath(path);
		 return FastDFSUtils.downloadFile(group, filename, specFileName);
	 }
}


```
#### 最后附上读取配置文件的工具类PropertyUtil
```java
package com.wuwii.utils;

import java.io.InputStream;
import java.util.Properties;

/**
 * @ClassName PropertyUtil
 * @Description 读取配置文件的内容（key，value）
 * @author zhangkai
 * @date 2017年7月18日
 */
public class PropertyUtil {
	public static final Properties PROP = new Properties();

	/** 
	 * @Method: get 
	 * @Description: 读取配置文件的内容（key，value）
	 * @param key
	 * @return String
	 * @throws 
	 */
	public static String get(String key) {
		if (PROP.isEmpty()) {
			try {
				InputStream in = PropertyUtil.class.getResourceAsStream("/config.properties");
				PROP.load(in);
				in.close();
			} catch (Exception e) {}
		}
		return PROP.getProperty(key);
	}
}

```
### 测试
#### 上传!

![猫咪](http://img.blog.csdn.net/20170917111957613?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzU5MTUzODQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
#### 下载
下载就很简单了，因为直接获得到二进制流了，返回给客户端即可。  
下载需要主要的是，要注意从FastDFS返回的文件名是这种随机码，因此我们需要在上传的时候将文件本身的名字存到数据库，再到下载的时候对文件头重新设置名字即可。

