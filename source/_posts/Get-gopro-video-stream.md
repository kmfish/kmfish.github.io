---
title: 如何获取 Gopro 视频流
date: 2015-08-31 15:28:34
tags:
---

前言：最近的工作在研究gopro的视频流如何获取，通过搜索资料，以及对gopro app的抓包分析，得出了以下经验。这次的分析过程也体会到抓包分析的好处，以后还应进一步学习如何用好抓包工具。

## 如何获取Gopro的视频流

以下的这些url会随gopro的型号版本而有所不同，需要自己抓包分析确定。

gopro提供了wifi和hdmi的视频输出，目前研究的是wifi输出。
通过对gopro app抓包分析，和网上搜索的资料，整理出以下方法：
1. 先通过gopro app完成和gopro的蓝牙配对，配置好wifi热点，并开启wifi热点；
2. 手机切换至gopro的热点网络，在 udp://:8554 上开启监听。
3. 发送http get request( http://10.5.5.9/gp/gpExec?p1=gpStreamA9&c1=restart)，gopro即开始向手机的 8554 端口发送数据。手机即可在udp 8554端口收到数据。
4. 通过抓包分析，gopro app会定期（1次/s）发送心跳请求，维持连接。
udp :　"_GPHD_:1:0:2:0"  0.3次/s  // 若无此心跳，则视频流几秒后会中断
http : GET /gp/gpControl/status   2次/s  //若无此心跳，则gopro的wifi热点几分钟后会关闭

备注：
抓包分析出gopro通信的一些url，响应均为json
**GET /bacpac/cv** 返回gopro热点名称
**GET /gp/gpControl** 返回gopro所有设置项的当前状态，可选项信息，以及所有支持的url command
如： **/gp/gpControl/command/wireless/pair/cancel** 如果已经配对成功则取消配对

其他包
gopro app 会发送mdns请求，获取gopro的ip地址及mac地址等信息；
gopro app 每次进入preview界面时，会向全网发送WOL包，WOL（Wake on Lan）局域网唤醒，远程唤醒设备。

## 视频流格式，码率
h.264，aac，mpegts流
录制视频：
码率范围：20mbps~60mbps 
http://zh.gopro.com/support/articles/hero4-black-recording-time-in-each-video-setting

实时预览下码率为： 
1mbps，帧率为30fps

## 上传视频
1、wifi连接gopro，可以通过移动网络传输视频流
ios下，可以在连接wifi热点的同时，其他数据通信自动走移动网络通道完成。
android，5.0提供了新的网络api，可以给app指定使用特定网络。

















