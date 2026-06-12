# 基于Openseeface方法的在Archlinux系统下绕过VTS直接进行面部检测使用VTS虚拟形象的方法

# 摘要  Abstact

VTS采用经典的Openseeface方法来进行面部检测，但是这样的检测方式如果自行配置的话在Windows环境下对用户非常的不友好，于是VTS自行做了封装。而Archlinux下的用户明显是没法享受到这一用户便利性封装的（图形GPU的API方法不同）因此如果想要在Archlinux下使用VTS进行虚拟形象直播，需要深入了解Openseeface方法的基本原理。



# 方法 Methods

### 1\.在AUR仓库下载openseeface\-bin

直接使用yay即可:yay \-S openseeface\-git

下载好了以后，使用facetracker命令即可启用openseeface

### 2\.配置网络连接直链VTS

在下载好了openseeface以后，VTS是没有办法直接检测到OSF的连接的，我们需要手动进行配置。

根据https://github\.com/DenchiSoft/VTubeStudio/wiki/Running\-VTS\-on\-Linux/的Wiki介绍，如果需要手动配置的话需要到VTS的目录下自行创建ip\.txt文件，文件内容如下:



ip=0\.0\.0\.0

port=11573



这里的port是官方推荐的port,无脑使用即可。

### 3\.配置Openseeface的命令

推荐配置:
facetracker \-c 2 \-i 127\.0\.0\.1 \-p 11573 \-W 640 \-H 480 \-\-model 3 \-\-fps 30 \-\-pnp\-points 1


参数解释:
\-c是摄像头的编号

\-c 0一般是笔记本自带摄像头 \-c 2是我的外接摄像头 如果发现\-c 0/1/2都不是自己想要的那个摄像头，不妨多试试几个参数

\-i是网络ip 本地回环 不解释

\-p是端口

\-W \-H 即weight hight，摄像头捕捉的长宽

\-\-model 捕捉时采用的模型，\-1 到 3 由恶劣到优秀

\-\-fps 捕捉的帧率

\-\-pnp\-points 1 优化参数



推荐原因:


![Image](https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=ZWE2MzU1YjBhZDk3YjRlODM1NGU2NTVkYTYzY2YxY2FfYzdjYzg3Nzk0YzkwNThhMjQ4N2UyZjEzODZhN2QzMTJfSUQ6NzYxNzIwNTc0MTg1Nzg1MjYzOF8xNzgxMTQ0Njg0OjE3ODEyMzEwODRfVjM)

个人配置了一下threshold = 0\.3,这样有方便置信度低于百分之三十的推理结果直接被舍弃。

![Image](https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=ZjRkOGMzYTMwMDYyYWFkYWVmYjg5YjUwOWE3ZmIwYTFfZGEzNWNjMGY4MDdkMWNhMThmYzAxZmYxNGIwZjBjZDhfSUQ6NzYxNzIwNzI1MDAwNTQxMjgyN18xNzgxMTQ0Njg0OjE3ODEyMzEwODRfVjM)

当然，为了方便，还可以把facetracker设置为systemd,开机自启动后台运行:


![Image](https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=ZDJmODY1MDJlNTNlNTgyMDM2MjdmYmRlMzU2NDQ3ZDZfZjMxZmQ3Y2NlYjEyZTAxNzc5YTQzY2RlMzM0MGJjOGZfSUQ6NzYxNzIwNzUxMTkyMTIzMjg1Nl8xNzgxMTQ0Njg0OjE3ODEyMzEwODRfVjM)



### 4\.检查细致结果

只是单单配置Openseeface和VTS可能能直接运行，也可能出现脸部捕捉直接乱飘的结果。而一般出现脸部推理乱飘的原因一般是因为AI没能识别到眼窝与鼻梁线的原因，而这两个面部特征的识别很大程度上依赖合适的光源。太暗或者太曝都有可能影响到推断结果。因此可能需要添加 \-v 1 参数来直接读取摄像头原生画面来判断是否过暗/过曝来解决问题。一般和v4l2\-ctl（linux下的摄像头驱动）的设置有关 

这一步的摸索是最繁杂的，也是最关键的。

### 5\.使用

在配置好了以后，直接启动VTS即可，在使用Live2d的界面选择摄像头的选单将会直接消失，被替换为OpenSeeFace \- Network,采用的ip就是本地回环与刚刚填写的port,如果发现没有出现OSF\-N选项，去检查ip\.txt内和facetracker \-c 2 \-i 127\.0\.0\.1 \-p 11573\.\.\.下的\-p参数传入的端口是否一致。

![Image](https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=ZTljOWFmNmNiZWViODE3YmY1YmU0ZmJhZjEwMzRjNmJfODVhNjUwNWU2YWJkZjgzZDk4MjdiZGI1ODA4MDU3OWRfSUQ6NzYxNzIwNjc3OTgzOTA0MDcwM18xNzgxMTQ0Njg0OjE3ODEyMzEwODRfVjM)

选择好了以后，导入合适的Live2D形象即可正常使用。

![Image](https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=N2UzZjdkYTc1NDU5NDQ0YTA2YmViNWY1MDZkNTRiYjZfYTMxYzc0MzdhZjcyZTRkNTU3MTZjYWQ2ZDA1ZDY0NzJfSUQ6NzYxNzIwNzY0NjAwODkyMTMxMF8xNzgxMTQ0Njg0OjE3ODEyMzEwODRfVjM)

## 



