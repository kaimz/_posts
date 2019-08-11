---
title: MQTT在JAVA中使用
date: 2017-09-12 18:08:03
tags: [mq,java]
categories: 学习笔记
---


### 介绍
>由于项目中需求，多个客户机终端不断发送位置给服务机，服务机根据消息，准确判断信息，并返回响应，回复该客户机。

在这里我们的服务机，不但要订阅所有客户机的主题，还要根据客户机消息做出相应的响应，服务机同时充当客户机使用，客户机也推送主题消息，充当服务器。 
<!--more--> 
>关键问题：
1. 服务器怎么区分各个客户机
2. 主题配置方面的问题，不可能每个机器配个主题
3. 通信方面，选择哪种消息级别


### 环境配置
#### 使用maven项目，刚好在仓库导包了，可以使用（推荐）
```xml
<dependency>
     <groupId>org.eclipse.paho</groupId>
     <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
     <version>1.0.2</version>
 </dependency>
```
#### 仓库没有包，直接去网上下，可以直接导包到Lib中

### 编写
#### 服务器
```java
package com.devframe.mqtt.test;

import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.eclipse.paho.client.mqttv3.MqttException;
import org.eclipse.paho.client.mqttv3.persist.MemoryPersistence;


/** 
* @ClassName: ServerMQTT 
* @Description:  服务器接收多个客户端的主题，同时要像客户端反馈
* @author zhangkai 
* @date 2017年9月12日 下午12:52:01 
*  
*/
public class ServerMQTT {

	// 连接参数
	private final static String CONNECTION_STRING = "tcp://192.168.19.200:8001"; //host
	private final static boolean CLEAN_START = true;  //是否清空session，false保留
	private final static short KEEP_ALIVE = 30;// 低耗网络，但是又需要及时获取数据，心跳30*1.5s
	private final static short KEEP_TIME_OUT = 10; //连接超时
	private final static String CLIENT_ID = "master";// 客户端标识
	private final static int[] QOS_VALUES = { 0 };// 对应主题的消息级别
	private final static String[] TOPICS = { "agri/#"}; //匹配agri/下所有的主题
	private final String userName = "agri";
	private final String passWord = "admin@123";
	private MqttConnectOptions options;
	private MqttClient mqttClient;

	/**
	 * 构造函数
	 * 
	 * @throws MqttException
	 */
	public ServerMQTT() throws MqttException {

	}

	/**
	 * 发送消息
	 * @param topic 主题
	 * @param message 消息
	 * @param qos 消息级别{0,1,2}
	 * @param retained 是否是实时发送的消息(false=实时，true=服务器上保留的消息)
	 */
	public void sendMessage(String topic, String message, int qos, boolean retained) {
		try {
			//断开重连
			if (mqttClient == null || !mqttClient.isConnected()) {
				connect();
			}
			// 发布消息
			mqttClient.publish(topic, message.getBytes(), qos, retained);
		} catch (MqttException e) {
			e.printStackTrace();
		}
	}

	/**
	 * 重新连接服务
	 */
	private void connect() {
		try {
			mqttClient = new MqttClient(CONNECTION_STRING, CLIENT_ID, new MemoryPersistence());
			// MQTT的连接设置
			options = new MqttConnectOptions();
			// 设置是否清空session,这里如果设置为false表示服务器会保留客户端的连接记录，这里设置为true表示每次连接到服务器都以新的身份连接
			options.setCleanSession(CLEAN_START);
			// 设置连接的用户名
			options.setUserName(userName);
			// 设置连接的密码
			options.setPassword(passWord.toCharArray());
			// 设置超时时间 单位为秒
			options.setConnectionTimeout(KEEP_TIME_OUT);
			// 设置会话心跳时间 单位为秒 服务器会每隔1.5*30秒的时间向客户端发送个消息判断客户端是否在线，但这个方法并没有重连的机制
			options.setKeepAliveInterval(KEEP_ALIVE); 
			// 设置回调
			mqttClient.setCallback(new PushCallback());

			mqttClient.connect(options);
			mqttClient.subscribe( TOPICS , QOS_VALUES);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	/**
	 * 启动入口
	 * 
	 * @param args
	 * @throws MqttException
	 */
	public static void main(String[] args) throws MqttException {
		ServerMQTT server = new ServerMQTT();
		server.connect();
	}
}

```
#### 客户机
```java
package com.devframe.mqtt.test;

import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

import org.eclipse.paho.client.mqttv3.MqttClient;
import org.eclipse.paho.client.mqttv3.MqttConnectOptions;
import org.eclipse.paho.client.mqttv3.MqttException;
import org.eclipse.paho.client.mqttv3.MqttSecurityException;
import org.eclipse.paho.client.mqttv3.persist.MemoryPersistence;

public class ClientMQTT implements Runnable {

	// 连接参数
	private final static String CONNECTION_STRING = "tcp://192.168.19.200:8001"; // host
	private final static boolean CLEAN_START = true; // 是否清空session，false保留
	private final static short KEEP_ALIVE = 30;// 低耗网络，但是又需要及时获取数据，心跳30*1.5s
	private final static short KEEP_TIME_OUT = 10; // 连接超时
	private final static String CLIENT_ID = "client1";// 客户端标识
	private final static int[] QOS_VALUES = { 0 };// 对应主题的消息级别
	private final static String PUBLISH_TOPIC = "agri/index1";
	private final static String[] RECEIVE_TOPIC = {"agri/index1/back"};
	private MqttClient client;
	private MqttConnectOptions options;
	private final String userName = "agri";
	private final String passWord = "admin@123";
	private ScheduledExecutorService scheduler;

	// 重新连接
	public void startReconnect() {
		scheduler = Executors.newSingleThreadScheduledExecutor();
		scheduler.scheduleAtFixedRate(new Runnable() {
			public void run() {
				if (!client.isConnected()) {
					try {
						client.connect(options);
					} catch (MqttSecurityException e) {
						e.printStackTrace();
					} catch (MqttException e) {
						e.printStackTrace();
					}
				}
			}
		}, 0 * 1000, 10 * 1000, TimeUnit.MILLISECONDS);
	}

	/**
	 * 发送消息
	 * 
	 * @param topic
	 *            主题
	 * @param message
	 *            消息
	 * @param qos
	 *            消息级别{0,1,2}
	 * @param retained
	 *            是否是实时发送的消息(false=实时，true=服务器上保留的最后消息)
	 */
	public void sendMessage(String topic, String message, int qos, boolean retained) {
		try {
			if (client == null || !client.isConnected()) {
				connect();
			}
			// 发布消息
			client.publish(topic, message.getBytes(), qos, retained);
		} catch (MqttException e) {
			e.printStackTrace();
		}
	}

	/**
	 * 重新连接服务
	 */
	private void connect() {
		try {
			client = new MqttClient(CONNECTION_STRING, CLIENT_ID, new MemoryPersistence());
			// MQTT的连接设置
			options = new MqttConnectOptions();
			// 设置是否清空session,这里如果设置为false表示服务器会保留客户端的连接记录，这里设置为true表示每次连接到服务器都以新的身份连接
			options.setCleanSession(CLEAN_START);
			// 设置连接的用户名
			options.setUserName(userName);
			// 设置连接的密码
			options.setPassword(passWord.toCharArray());
			// 设置超时时间 单位为秒
			options.setConnectionTimeout(KEEP_TIME_OUT);
			// 设置会话心跳时间 单位为秒 服务器会每隔1.5*30秒的时间向客户端发送个消息判断客户端是否在线，但这个方法并没有重连的机制
			options.setKeepAliveInterval(KEEP_ALIVE);
			// 设置回调
			client.setCallback(new PushCallback());

			client.connect(options);
			client.subscribe(RECEIVE_TOPIC, QOS_VALUES);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	@Override
	public void run() {
		// TODO 每秒一次向服务端发送消息
		try {
			int i = 0;
			while (true) {
				sendMessage(PUBLISH_TOPIC, "hello,topic" + i, 0, false);
				i++;
				Thread.sleep(1000);
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
	
	public static void main(String[] args) throws MqttException {
		ClientMQTT client = new ClientMQTT();
		Thread thread = new Thread(client, "th1");
		thread.start();
	}
}

```
测试的时候就是把客户机复制了几个，主题和clientid改下，clientid不能一样，不然不能登陆。

### 回调类
发送完消息，Service需要在这里处理，着我们先做的事情啦。
```java
package com.devframe.mqtt.test;

import org.eclipse.paho.client.mqttv3.IMqttDeliveryToken;  
import org.eclipse.paho.client.mqttv3.MqttCallback;  
import org.eclipse.paho.client.mqttv3.MqttMessage;  
  
/**
 * @ClassName: PushCallback
 * @Description: 发布消息的回调类
 * 
 *               必须实现MqttCallback的接口并实现对应的相关接口方法CallBack 类将实现 MqttCallBack。
 *               每个客户机标识都需要一个回调实例。在此示例中，构造函数传递客户机标识以另存为实例数据。
 *               在回调中，将它用来标识已经启动了该回调的哪个实例。 必须在回调类中实现三个方法：
 * 
 *               public void messageArrived(MqttTopic topic, MqttMessage
 *               message)接收已经预订的发布。
 * 
 *               public void connectionLost(Throwable cause)在断开连接时调用。
 * 
 *               public void deliveryComplete(MqttDeliveryToken token))
 *               接收到已经发布的QoS 0、 QoS 1 或 QoS 2 消息的传递令牌时调用。 由 MqttClient.connect
 *               激活此回调。
 * @author zhangkai
 * @date 2017年9月12日 上午11:30:44
 * 
 */
public class PushCallback implements MqttCallback {  
  
    public void connectionLost(Throwable cause) {  
        // TODO 连接丢失后，一般在这里面进行重连  
        System.out.println("连接断开，可以做重连");  
    }  
  
    public void deliveryComplete(IMqttDeliveryToken token) {  
        System.out.println("deliveryComplete---------" + token.isComplete());  
    }  
  
    public void messageArrived(String topic, MqttMessage message) throws Exception {  
        // TODO subscribe后得到的消息会执行到这里面，后续工作将在这里进行  
        System.out.println("接收消息主题 : " + topic);  
        System.out.println("接收消息Qos : " + message.getQos());  
        System.out.println("接收消息内容 : " + new String(message.getPayload()));  
    } 
    
}  

```

### 总结
1. 使用前缀通配符的方式加上特殊码区分各个机器的主题。
2. 根据不同业务的需求，需要合理的选择不同级别的消息。
3. 实际使用中一般把发消息的参数 ``retained`` 设为 ``false`` ，这个参数的说明是:
    设为true之后把消息保存到本地，每一次去订阅该主题的subscriber都会收到，每次订阅的时候都会收到，导致很多重复多余的消息。

  如果在使用的过程中不小心将它设置成true，怎么去清除这个存着的消息了，mqtt本身没这个功能；
  **解决办法：向该topic重新publish数据，RETAIN=TRUE，``Payload为空``。**

所以，刚开始做这个都得时候就是设置成true，包括上面的测试代码，还没改过来，业务代码已经全部改好了。不然没次去连接mqtt的时候，都会订阅到一大片消息，电脑跑到卡得不行，哈哈。
