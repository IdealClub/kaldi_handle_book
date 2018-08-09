[toc]

在kaldi 的工具集里有好几个程序可以用于在线识别。这些程序都位在src/onlinebin文件夹里，他们是由src/online文件夹里的文件编译而成(你现在可以用make ext 命令进行编译)。这些程序大多还需要tools文件夹中的portaudio 库文件支持，portaudio 库文件可以使用tools文件夹中的相应脚本文件下载安装。

```bash
# 安装portaudio
yum -y install *alsa*
cd kaldi/tools/
./install_portaudio.sh

# 编译在线识别工具
cd src/
make ext
或者进入kaldi/src/online和kaldi/src/onlinebin，分别make clean ,make就完美解决
```
##一、服务器客户端识别系统建立方法
建立整个在线识别系统需要：

- 准备两台机器，都安装kaldi；
- 作为服务器的机器，准备好声音模型、词典、解码网络、特征转换矩阵（我还没有使用转换矩阵）
- 首先启动服务器，待服务器运行后，再启动客户端连接。

###1. Command line to start the **server**（服务器端启动方式）:
使用如下指令`online-audio-server-decode-faster`启动服务器：
```bash
online-audio-server-decode-faster --verbose=1 --rt-min=0.5 --rt-max=3.0 --max-active=6000 \
--beam=72.0 --acoustic-scale=0.0769 final.mdl graph/HCLG.fst graph/words.txt '1:2:3:4:5' \
graph/word_boundary.int 5010 final.mat
```

####1.1 Arguments are as follow（参数意义）:

- `final.mdl` - the acoustic model
- `HCLG.fst` - the complete FST
- `words.txt` - word dictionary (mapping word ids to their textual representation)
- `'1:2:3:4:5'` - list of silence phoneme ids
- `5010` - port the server is listening on
- `word_boundary.int`- a list of phoneme boundary information required for word alignemnt
- `final.mat` - feature LDA matrix

配置选项：

```
选项：
   - 声学尺度：声学似然的缩放因子（浮点数，默认值= 0.1）
  --batch-size：无中断处理的特征向量数（int，default = 27）
  --beam：解码光束。更大 - >更慢，更准确。 （float，默认= 16）
  --beam-delta：解码器中使用的增量[模糊设置]（浮点数，默认值= 0.5）
  --beam-update：光束更新率（浮点数，默认值= 0.01）
  --cmn-window：壮举数量。运行平均CMN计算中使用的向量（int，默认= 600）
  --delta-order：delta计算顺序（int，default = 2）
  --delta-window：增量计算的参数控制窗口（每个增量顺序的实际窗口大小为1 + 2 * delta-window-size）（int，default = 2）
  --hash-ratio：解码器中用于控制散列行为的设置（float，default = 2）
  --inter-utt-sil：触发新话语的最大静音帧数（i​​nt，default = 50）
  --left-context：左上下文的帧数（int，default = 4）
  --max-active：解码器最大活动状态。 Larger->慢;更准确（int，默认= 2147483647）
  --max-beam-update：最大光束更新率（浮点数，默认值= 0.05）
  --max-utt-length：如果话语变得比这个帧数长，则可以接受较短的静音作为话语分隔符（int，default = 1500）
  --min-active：解码器最小活动状态（如果#active小于此值，则不要修剪）。 （int，默认= 20）
  --min-cmn-window：解码开始时使用的Minumum CMN窗口（仅在启动时添加延迟）（int，default = 100）
  --num-attempts：终止stream之前连续重复超时的次数（int，default = 5）
  --right-context：正确上下文的帧数（int，default = 4）
  --rt-max：近似最大解码运行时间因子（float，default = 0.75）
  --rt-min：近似最小解码运行时间因子（浮点数，默认值= 0.7）
  --update-interval：帧中的波束更新间隔（int，默认= 3）

标准选项：
  --config：要读取的配置文件（此选项可能会重复）（string，default =“”）
  --help：打印出用法消息（bool，default = false）
  --print-args：打印命令行参数（到stderr）（bool，default = true）
  --verbose：详细级别（更高 - >更多日志记录）（int，default = 0）
```

**注意**：如果没有`word_boundary.int` 需要重新运行`prepare_lang.sh`生成。修改如下：

```bash
#原指令：
utils/prepare_lang.sh --position-dependent-phones false data/local/dict "<SPOKEN_NOISE>" \
data/local/lang data/lang
#改为：
utils/prepare_lang.sh data/local/dict "<SPOKEN_NOISE>" data/local/lang data/lang
```
启动后结果如下：
![这里写图片描述](https://img-blog.csdn.net/20180807165939323?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NoaW5hdGVsZWNvbTA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

###2. Command line to start the **client**（客户端启动方式）:
直接运行如下指令即可启动客户端：
```bash
 online-audio-client --htk --vtt localhost 5010 scp:test.scp
```
####2.1 Arguments are as follow（参数意义）:

- `–htk` - save results as an HTK label file
- `–vtt` - save results as a WebVTT file
- `localhost` - server to connect to
- `5010` - port to connect to
- `scp:test.scp` - list of WAV files to send
启动后客户端不断传输数据，服务器实时进行解码！结果如下：
![这里写图片描述](https://img-blog.csdn.net/20180807170201340?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NoaW5hdGVsZWNvbTA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
结果是边传输边识别的：
![这里写图片描述](https://img-blog.csdn.net/20180807170502703?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NoaW5hdGVsZWNvbTA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
###* Command line to start the Java client（移动客户端）:
移动客户端我还未尝试：
```bash
java -jar online-audio-client.jar
```
Or simply double-click the JAR file in the graphical interface.

##二、使用麦克风建立客户端与服务器的实时解码
kaldi提供了读取客户端麦克风数据的解码工具，可以在客户端使用麦克风发送音频，服务器实时返回解码数据。

###1. 使用`online-server-gmm-decode-faster`启动服务器：
-  通过网络接收特征进行解码。话语分词是即时完成的。如果给出可选（最后）参数，则使用特征拼接/ LDA变换。 否则默认使用delta / delta-delta（2阶）特征。
```bash
Usage: online-server-gmm-decode-faster [options] model-infst-in word-symbol-table silence-phones udp-port [lda-matrix-in]
Example: online-server-gmm-decode-faster --rt-min=0.3 --rt-max=0.5 --max-active=4000 --beam=12.0 --acoustic-scale=0.0769 model HCLG.fst words.txt '1:2:3:4:5' 1234 lda-matrix 
```
配置选项：

```
选项：
   - 声学尺度：声学似然的缩放因子（浮点数，默认值= 0.1）
  --batch-size：无中断处理的特征向量数（int，default = 27）
  --beam：解码光束。更大 - >更慢，更准确。 （float，默认= 16）
  --beam-delta：解码器中使用的增量[模糊设置]（浮点数，默认值= 0.5）
  --beam-update：光束更新率（浮点数，默认值= 0.01）
  --cmn-window：壮举数量。运行平均CMN计算中使用的向量（int，默认= 600）
  --delta-order：delta计算顺序（int，default = 2）
  --delta-window：增量计算的参数控制窗口（每个增量顺序的实际窗口大小为1 + 2 * delta-window-size）（int，default = 2）
  --hash-ratio：解码器中用于控制散列行为的设置（float，default = 2）
  --inter-utt-sil：触发新话语的最大静音帧数（i​​nt，default = 50）
  --left-context：左上下文的帧数（int，default = 4）
  --max-active：解码器最大活动状态。 Larger->慢;更准确（int，默认= 2147483647）
  --max-beam-update：最大光束更新率（浮点数，默认值= 0.05）
  --max-utt-length：如果话语变得比这个帧数长，则可以接受较短的静音作为话语分隔符（int，default = 1500）
  --min-active：解码器最小活动状态（如果#active小于此值，则不要修剪）。 （int，默认= 20）
  --min-cmn-window：解码开始时使用的Minumum CMN窗口（仅在启动时添加延迟）（int，default = 100）
  --num-attempts：终止stream之前连续重复超时的次数（int，default = 5）
  --right-context：正确上下文的帧数（int，default = 4）
  --rt-max：近似最大解码运行时间因子（float，default = 0.75）
  --rt-min：近似最小解码运行时间因子（浮点数，默认值= 0.7）
  --update-interval：帧中的波束更新间隔（int，默认= 3）

标准选项：
  --config：要读取的配置文件（此选项可能会重复）（string，default =“”）
  --help：打印出用法消息（bool，default = false）
  --print-args：打印命令行参数（到stderr）（bool，default = true）
  --verbose：详细级别（更高 - >更多日志记录）（int，default = 0）
```


###2. 使用`online-net-client`启动客户端：
- 通过`online-net-client`工具，使用麦克风（portaudio）作为输入，提取特征并通过网络连接发送它们到服务器上。具体设置如下：
```bash
Usage: online-net-client server-address server-port 

Options:
  --batch-size                : The number of feature vectors to be extracted and sent in one go (int, default = 27)
  
Standard options:
  --config                    : Configuration file to read (this option may be repeated) (string, default = "")
  --help                      : Print out usage message (bool, default = false)
  --print-args                : Print the command line arguments (to stderr) (bool, default = true)
  --verbose                   : Verbose level (higher->more logging) (int, default = 0)
```


引用：[kaldi-asr](http://kaldi-asr.org/doc/online_decoding.html)
转载请注明出处：[https://blog.csdn.net/chinatelecom08/article/details/81476698](https://blog.csdn.net/chinatelecom08/article/details/81476698)
> 参考：kaldi首页