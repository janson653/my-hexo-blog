---
title: 使用JavaCV实现RTSP流转FLV存储
date: 2025-03-26 21:00:00
tags:
  - JavaCV
  - RTSP
  - 视频处理
  - Java
categories:
  - 技术教程
---

# 使用JavaCV实现RTSP流转FLV存储

## 背景介绍

在视频监控、直播系统等场景中，经常需要将RTSP视频流保存为本地文件。本文将介绍如何使用JavaCV实现RTSP流的接收和FLV格式的本地存储，并加入定时录制功能。

## 环境准备

在开始之前，我们需要在项目中添加以下依赖：

```xml
<dependencies>
    <!-- JavaCV平台接口 -->
    <dependency>
        <groupId>org.bytedeco</groupId>
        <artifactId>javacv-platform</artifactId>
        <version>1.5.9</version>
    </dependency>
</dependencies>
```

## 核心实现

我们将创建一个`RtspToFlvRecorder`类来实现核心功能：

```java
import org.bytedeco.ffmpeg.avcodec.AVPacket;
import org.bytedeco.javacv.*;

public class RtspToFlvRecorder {
    private final String rtspUrl;        // RTSP流地址
    private final String outputFile;     // 输出文件路径
    private final int duration;          // 录制时长（秒）
    private volatile boolean isRunning;  // 运行状态标志

    public RtspToFlvRecorder(String rtspUrl, String outputFile, int duration) {
        this.rtspUrl = rtspUrl;
        this.outputFile = outputFile;
        this.duration = duration;
    }

    public void startRecording() {
        isRunning = true;
        
        try (FFmpegFrameGrabber grabber = new FFmpegFrameGrabber(rtspUrl)) {
            // 配置RTSP相关参数
            grabber.setOption("rtsp_transport", "tcp"); // 使用TCP方式
            grabber.setOption("stimeout", "2000000");   // 设置超时时间
            grabber.start();

            // 创建录制器
            FFmpegFrameRecorder recorder = new FFmpegFrameRecorder(outputFile, 
                grabber.getImageWidth(), 
                grabber.getImageHeight());
            
            // 配置录制参数
            recorder.setVideoCodec(grabber.getVideoCodec());
            recorder.setFormat("flv");
            recorder.setFrameRate(grabber.getFrameRate());
            recorder.setVideoBitrate(grabber.getVideoBitrate());
            recorder.start();

            // 获取流的帧率
            double frameRate = grabber.getFrameRate();
            int totalFrames = (int) (frameRate * duration); // 计算总帧数

            // 开始时间
            long startTime = System.currentTimeMillis();
            int videoFrameCount = 0; // 视频帧计数器
            
            // 录制循环
            while (isRunning && videoFrameCount < totalFrames) {
                // 获取视频帧
                Frame frame = grabber.grabFrame();
                if (frame != null && frame.image != null) { // 确保帧有效且包含图像
                    recorder.record(frame); // 录制视频帧
                    videoFrameCount++; // 增加视频帧计数
                }
                // 检查时间是否到达
                if (isTimeUp(startTime)) {
                    break; // 时间到达，退出循环
                }
            }

            // 正常结束录制
            recorder.stop();
            recorder.close();
            
        } catch (Exception e) {
            throw new RuntimeException("录制过程发生错误: " + e.getMessage(), e);
        }
    }

    private boolean isTimeUp(long startTime) {
        return (System.currentTimeMillis() - startTime) >= duration * 1000L;
    }

    public void stopRecording() {
        isRunning = false;
    }
}
```

## 使用示例

```java
public class RecordingDemo {
    public static void main(String[] args) {
        String rtspUrl = "rtsp://your-camera-ip:554/stream";
        String outputFile = "D:/recordings/output.flv";
        int duration = 60; // 录制60秒
        
        RtspToFlvRecorder recorder = new RtspToFlvRecorder(rtspUrl, outputFile, duration);
        
        try {
            recorder.startRecording();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

## 注意事项

1. RTSP地址格式：确保RTSP URL格式正确，通常格式为：`rtsp://ip:port/stream`
2. 存储路径：确保输出文件路径存在且有写入权限
3. 网络状况：RTSP流的稳定性受网络状况影响，建议添加重试机制
4. 资源释放：程序结束时要正确释放资源，避免内存泄漏
5. 异常处理：在生产环境中需要完善的异常处理机制

## 扩展优化

1. 添加录制状态监控
2. 实现断线重连机制
3. 添加录制质量参数配置
4. 支持多路流同时录制
5. 添加录制进度回调

## 总结

本文介绍了如何使用JavaCV实现RTSP流转FLV的录制功能，并加入了定时控制机制。通过合理的参数配置和异常处理，可以实现稳定的视频流录制功能。在实际应用中，可以根据具体需求进行功能扩展和优化。 