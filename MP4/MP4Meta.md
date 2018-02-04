# MP4 Meta

## MP4格式

>mp4是由一个个``box``组成的，大``box``中存放小``box``，**一级嵌套一级** 来存放媒体信息。



## 视频时长

方法1

从mvhd - movie header atom中找到``time scale``和duration，duration除以time scale即是整部电影的长度。
time scale相当于定义了标准的1秒在这部电影里面的刻度是多少。

例如audio track的time scale = 8000, duration = 560128，所以总长度是70.016，video track的time scale = 600, duration = 42000，所以总长度是70

- 方法2

首先计算出共有多少个帧，也就是sample（从sample size atoms中得到），然后
整部电影的duration = 每个帧的duration之和（从Time-to-sample atoms中得出）。

例如``audio track``共有547个``sample``，每个``sample``的长度是1024，则总``duration``是560128，电影长度是70.016；
``video track``共有1050个``sample``，每个``sample``的长度是40，则总``duration``是42000，电影长度是70



## 宽高

从tkhd – track header atom中找到宽度和高度即是。



## 电影声音采样频率

从tkhd – track header atom中找出audio track的time scale即是声音的采样频率


## 计算视频帧率

首先计算出整部电影的duration，和帧的数目然后

帧率 = 整部电影的duration / 帧的数目

## 比特率

整部电影的尺寸除以长度，即是比特率，此电影的比特率为846623/70 = 12094 bps



## 查找``sample``

当播放一部电影或者一个track的时候，对应的media handler必须能够正确的解析数据流，对一定的时间获取对应的媒体数据。如果是视频媒体， media handler可能会解析多个atom，才能找到给定时间的sample的大小和位置。具体步骤如下：

1. 确定时间，相对于媒体时间坐标系统
2. 检查time-to-sample atom来确定给定时间的sample序号。
3. 检查sample-to-chunk atom来发现对应该sample的chunk。
4. 从chunk offset atom中提取该trunk的偏移量。
5. 利用sample size atom找到sample在trunk内的偏移量和sample的大小。



例如，如果要找第1秒的视频数据，过程如下：

1. 第1秒的视频数据相对于此电影的时间为600
2. 检查time-to-sample atom，得出每个sample的duration是40，从而得出需要寻找第600/40 = 15 + 1 = 16个sample
3. 检查sample-to-chunk atom，得到该sample属于第5个chunk的第一个sample，该chunk共有4个sample
4. 检查chunk offset atom找到第5个trunk的偏移量是20472
5. 由于第16个sample是第5个trunk的第一个sample，所以不用检查sample size atom，trunk的偏移量即是该sample的偏移量20472。如果是这个trunk的第二个sample，则从sample size atom中找到该trunk的前一个sample的大小，然后加上偏移量即可得到实际位置。
6. 得到位置后，即可取出相应数据进行解码，播放


## 查找关键帧

查找过程与查找sample的过程非常类似，只是需要利用sync sample atom来确定key frame的sample序号

确定给定时间的sample序号
检查sync sample atom来发现这个sample序号之后的key frame
检查sample-to-chunk atom来发现对应该sample的chunk
从chunk offset atom中提取该trunk的偏移量
利用sample size atom找到sample在trunk内的偏移量和sample的大小


## Random access

Seeking主要是利用sample table box里面包含的子box来实现的，还需要考虑edit list的影响。

可以按照以下步骤seek某一个track到某个时间T，注意这个T是以movie header box里定义的time scale为单位的：

如果track有一个edit list，遍历所有的edit，找到T落在哪个edit里面。将Edit的开始时间变换为以movie time scale为单位，得到EST，T减去EST，得到T'，就是在这个edit里面的duration，注意此时T'是以movie的time scale为单位的。然后将T'转化成track媒体的time scale，得到T''。T''与Edit的开始时间相加得到以track媒体的time scale为单位的时间点T'''。
这个track的time-to-sample表说明了该track中每个sample对应的时间信息，利用这个表就可以得到T'''对应的sample NT。
sample NT可能不是一个random access point，这样就需要其他表的帮助来找到最近的random access point。一个表是sync sample表，定义哪些sample是random access point。使用这个表就可以找到指定时间点最近的sync sample。如果没有这个表，就说明所有的sample都是synchronization points，问题就变得更容易了。另一个shadow sync box可以帮助内容作者定义一些特殊的samples，它们不用在网络中传输，但是可以作为额外的random access point。这就改进了random access，同时不会影响正常的传输比特率。这个表指出了非random access point和random access point之间的关系。如果要寻找指定sample之前最近的shadow sync sample，就需要查询这个表。总之，利用sync sample和shadow sync表，就可以seek到NT之前的最近的access point sample Nap。
找到用于access point的sample Nap之后，利用sample-to-chunk表来确定sample位于哪个chunk内。
找到chunk后，使用chunk offset找到这个chunk的开始位置。
使用sample-to-chunk表和sample size表中的数据，找到Nap在此chunk内的位置，再加上此chunk的开始位置，就找到了Nap在文件中的位置。
