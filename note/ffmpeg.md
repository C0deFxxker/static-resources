# ffmpeg
## 码率、帧率和文件大小
码率和帧率是视频文件的最重要的基本特征，对于他们的特有设置会决定视频质量。如果我们知道码率和时长那么可以很容易计算出输出文件的大小。  
帧率：帧率也叫帧频率，帧率是视频文件中每一秒的帧数，肉眼想看到连续移动图像至少需要15帧。  
码率：比特率(也叫码率，数据率)是一个确定整体视频/音频质量的参数，秒为单位处理的字节数，码率和视频质量成正比，在视频文件中中比特率用bps来表达

## 视频流截图
```sh
ffmpeg -i sintel_trailer-480p.webm -r 5 pics/frame%4d.jpg
```
* -i: 指定视频文件位置
* -r: 调整帧率，指定一秒提取多少帧

也可以通过过滤器形式控制截图帧率：
```sh
ffmpeg -i sintel_trailer-480p.webm -vf fps=fps=5 pics/frame%4d.jpg
```
* -vf fps: 与上面-r参数效果一样。

## 调整视频分辨率
ffmpeg -i input.mpg -vf scale=320:240 output.mp4
* -vf: 说明后面紧跟着视频过滤器
* scale(视频过滤器之一): 指定帧缩放大小，语法：scale=width:height\[:interl={1|-1}\]。参数可以指定按比例缩放，例如：scale=iw*0.5:ih*0.5

## 使用显卡加速
**-init_hw_device type\[=name\]\[:device[,key=value...\]\]**

**type** 可选为：
* cuda
* dxva2
* vaapi
* vdpau
* qsv
* opencl

# Nvidia显卡监控
测试机只有一个T4显卡，所以只监控这个显卡的变化，使用 **nvidia-smi** 指令获取GPU信息：
```sh
nvidia-smi -q
```

需要记录的参数有：
* GPU使用率
* 显存占用情况
* 功耗
* 温度

# CPU、内存监控

机器性能：
CPU： xxxx 12核
内存： 64G

任务素材：
1080p视频（时长20秒，MKV格式，6.76MB） 
每秒截取5帧成JPEG图片

# 纯CPU解码
总耗时：57秒
CPU占用率变化情况：全程所有核接近100%
内存占用：约3.5G

# GPU加速
总耗时：57秒
CPU占用率变化情况：全程所有核接近100%
内存占用：约3.5G