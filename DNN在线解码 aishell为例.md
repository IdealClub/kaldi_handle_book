在kaldi 的工具集里有好几个程序可以用于在线识别。这些程序都位在src/onlinebin文件夹里，他们是由src/online文件夹里的文件编译而成(你现在可以用make ext 命令进行编译)。这些程序大多还需要tools文件夹中的portaudio 库文件支持，portaudio 库文件可以使用tools文件夹中的相应脚本文件下载安装。

```bash
# 安装portaudio
yum -y install *alsa*
cd kaldi/tools
./install_portaudio.sh
# 编译在线识别工具
cd src/
make ext
```

> 注：online官方不再维护，新版本为online2，先从简单的online开始。

下面以aishell的chain模型为例，学习使用`online2-wav-nnet3-latgen-faster`对训练好的aishell模型进行解码。

- 首先，生成解码所需配置文件：
```bash
# 注意：需要[ --add_pitch=true ]加入pitch特征。
steps/online/nnet3/prepare_online_decoding.sh --add_pitch=true data/lang_chain exp/nnet3/extractor \
exp/chain/tdnn_1a_sp exp/chain/nnet_online
```

- 如果直接解码会提示出错：
```
LOG (wav-copy[5.3]:main():wav-copy.cc:68) Copied 1 wave files
ERROR (online2-wav-nnet3-latgen-faster[5.3]:OnlineTransform():online-feature.cc:421)Dimension \
mismatch: source features have dimension 91 and LDA #cols is 281
```
- 不要紧张，是因为特征维度不匹配。我们生成的配置文件里的特征类型是MFCC，而aishell训练nnet和chain模型输入的是更高维度的MFCC，叫mfcc_hire，hire是high resolution单词的缩写[[引用文章]](https://blog.csdn.net/it_king1/article/details/80109398)。所以我们要修改第一步生成的MFCC配置文件，打开mfcc.conf文件，我们看到：
```bash
# 原文件内容：
--use-energy=false   # only non-default option.
--sample-frequency=16000
# 我们需要添加新内容，如下：
--num-mel-bins=40     # similar to Google's setup.
--num-ceps=40     # there is no dimensionality reduction.
--low-freq=40    # low cutoff frequency for mel bins
--high-freq=-200 # high cutoff frequently,relative to Nyquist of 8000 (=3800)
```
- 完成后就可以解码了，使用在线解码工具`online2-wav-nnet3-latgen-faster`进行解码：
```
online2-wav-nnet3-latgen-faster --config=exp/chain/nnet_online/conf/online.conf --do-\
endpointing=false --frames-per-chunk=20 --extra-left-context-initial=0 --online=true --frame-\
subsampling-factor=3 --max-active=7000 --beam=15.0 --lattice-beam=6.0 --online=false --acoustic-\
scale=0.1 --word-symbol-table=data/lang/words.txt exp/chain/tdnn_1a_sp/final.mdl \
exp/chain/tdnn_1a_sp/graph/HCLG.fst ark:data/test/spk2utt scp:data/test/wav.scp ark,t:20180803.txt
```
- 得到识别结果！！！！：

```
BAC009S0764W0121 甚至 出现 交易 几乎 停滞 的 情况 
LOG (online2-wav-nnet3-latgen-faster[5.4.208~1-6f214]:main():online2-wav-nnet3-latgen-faster.cc:286) Decoded utterance BAC009S0764W0121
BAC009S0764W0122 一二 线 城市 虽然 也 处于 调整 中 
LOG (online2-wav-nnet3-latgen-faster[5.4.208~1-6f214]:main():online2-wav-nnet3-latgen-faster.cc:286) Decoded utterance BAC009S0764W0122
BAC009S0764W0123 但 因为 聚集 了 过多 公共 四元 
LOG (online2-wav-nnet3-latgen-faster[5.4.208~1-6f214]:main():online2-wav-nnet3-latgen-faster.cc:286) Decoded utterance BAC009S0764W0123
BAC009S0764W0124 为了 规避 三四 线 城市 明显 过剩 的 市场 风险 
LOG (online2-wav-nnet3-latgen-faster[5.4.208~1-6f214]:main():online2-wav-nnet3-latgen-faster.cc:286) Decoded utterance BAC009S0764W0124
BAC009S0764W0125 标杆 房企 必然 调整 市场 战略 
LOG (online2-wav-nnet3-latgen-faster[5.4.208~1-6f214]:main():online2-wav-nnet3-latgen-faster.cc:286) Decoded utterance BAC009S0764W0125
BAC009S0764W0126 因此 土地 储备 至关 重要 
LOG (online2-wav-nnet3-latgen-faster[5.4.208~1-6f214]:main():online2-wav-nnet3-latgen-faster.cc:286) Decoded utterance BAC009S0764W0126
BAC009S0764W0127 中原 地产 首席 分析 师 张大 伟 说 
LOG (online2-wav-nnet3-latgen-faster[5.4.208~1-6f214]:main():online2-wav-nnet3-latgen-faster.cc:286) Decoded utterance BAC009S0764W0127
BAC009S0764W0128 一 线 城市 土地 供应 量 减少 
LOG (online2-wav-nnet3-latgen-faster[5.4.208~1-6f214]:main():online2-wav-nnet3-latgen-faster.cc:286) Decoded utterance BAC009S0764W0128
BAC009S0764W0129 也 助推 了 土地 市场 的 火爆 

```
转载请注明出处：[https://blog.csdn.net/chinatelecom08/article/details/81392535](https://blog.csdn.net/chinatelecom08/article/details/81392535)
参考：[IT_King1《DNN在线解码（以aishell的chain模型为例）》](https://blog.csdn.net/it_king1/article/details/80109398)