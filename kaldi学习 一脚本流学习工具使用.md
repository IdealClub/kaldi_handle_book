[toc]
> kaldi中脚本东西比较多，一层嵌一层，不易阅读。
> 本文以yesno为例，直接使用kaldi编译的工具，书写简易训练步骤，方便学习kaldi工具的使用。
> 注意：转载请注明出处。

##yesno训练
- 准备数据
	- 在yesno/s5下新建文件夹：`mkdir easy`，后续的操作将在easy文件夹中执行。
	- 拷贝s5下`./path`到`easy`文件夹中，./path的作用是能直接调用工具，不用添加工具所在路径，类似于设置环境变量。
	- 本脚主要便于理解kaldi工具的使用，一些批处理和数据下载并没有做，需要运行一遍`yesno/s5/./run.sh`生成训练所需输入。
	- 到`s5/data/train`下拷贝`wav.scp`到easy目录下作为训练输入，因为`wav.scp`是相对路径也需要拷贝`waves_yesno/`到easy下。
	- 拷贝词典到目录下：拷贝`s5/input`到easy目录下。
	准备数据结束，可以写自己的脚本了。
###先给出整体脚本如下：
```shell
#!/bin/bash
. ./path
# feature extraction:
# a series of light command	[ compute-mfcc + copy-feats -> compute-cmvn-stats -> apply-cmvn -> add-deltas ] 
# the data flow transition	[ wav -> mfcc.ark,scp -> cmvn.ark,scp -> delta.ark ]
mkdir mfcc
compute-mfcc-feats --verbose=2 --config="../conf/mfcc.conf" scp,p:wav.scp ark:- | copy-feats --compress=true ark:- ark,scp:mfcc/mfcc.ark,mfcc/mfcc.scp
compute-cmvn-stats scp:mfcc/mfcc.scp ark:mfcc/cmvn.ark
apply-cmvn ark:mfcc/cmvn.ark scp:mfcc/mfcc.scp ark:- | add-deltas ark:- ark:mfcc/delta.ark


# prepare dict for lang:
# input data 	[ lexicon_nosil.txt  lexicon.txt  phones.txt ]
# output data 	[ lexicon.txt  lexicon_words.txt  nonsilence_phones.txt  optional_silence.txt  silence_phones.txt ]
mkdir -p lang/dict
cp input/lexicon_nosil.txt lang/dict/lexicon_words.txt
cp input/lexicon.txt lang/dict/lexicon.txt
cat input/phones.txt | grep -v SIL > lang/dict/nonsilence_phones.txt
echo "SIL" > lang/dict/silence_phones.txt
echo "SIL" > lang/dict/optional_silence.txt
echo "Dictionary preparation succeeded"


# generate [ topo ] for acoustic model
utils/gen_topo.pl 3 5 2:3 1 > lang/lang/topo
# from [lexicoin phone word] -> [L.fst word.txt] for [G.fst train.fst HCLG.fst]
utils/prepare_lang.sh --position-dependent-phones false lang/dict "<SIL>" lang/local lang/lang


# train monophic acoustic model
# 1.from [topo 39] -> 0.mdl tree
gmm-init-mono --train-feats=ark:mfcc/delta.ark lang/lang/topo 39 mono/0.mdl mono/tree
# 2.from [L.fst 0.mdl tree word.txt text] -> train.fst
# compile-train-graphs [options] <tree-in> <model-in> <lexicon-fst-in> <transcriptions-rspecifier> <graphs-wspecifier>
compile-train-graphs mono/tree mono/0.mdl lang/lang/L.fst 'ark:sym2int.pl -f 2- lang/lang/words.txt text|' ark:lang/lang/graphs.fsts
# 3.from [graphs.fst] equally align the train data -> [ euqal.ali ]
# align-equal-compiled <graphs-rspecifier> <features-rspecifier> <alignments-wspecifier>
align-equal-compiled ark:lang/lang/graphs.fsts ark:mfcc/delta.ark ark:mono/equal.ali
# 4.from [equal.ali delta.ark mdl] ->  [ 0.acc ]
gmm-acc-stats-ali mono/0.mdl ark:mfcc/delta.ark ark:mono/equal.ali mono/0.acc
# 5.from [0.mdl 0.acc] -> [ 1.mdl ] 
# parameter est: 
gmm-est mono/0.mdl mono/0.acc mono/1.mdl

x=1
numliter=40
numgauss=11
while [ $x -lt $numliter ]; do
	# 6.from [1.mdl graphs.fst] align the data by new model -> [ 1.ali ]
	gmm-align-compiled --beam=6 --retry-beam=20 mono/$x.mdl ark:lang/lang/graphs.fsts ark:mfcc/delta.ark ark:mono/$x.ali
	# 4.from [equal.ali delta.ark mdl] ->  [ 0.acc ]
	gmm-acc-stats-ali mono/$x.mdl ark:mfcc/delta.ark ark:mono/equal.ali mono/$x.acc
	# 5.from [x.mdl x.acc] -> [ x+1.mdl ] 
	gmm-est --mix-up=$numgauss --power=0.25 mono/$x.mdl mono/$x.acc mono/$[$x+1].mdl
	numgauss=$[$numgauss+25]
	x=$[$x+1]
done
cp mono/$x.mdl mono/final.mdl


# Graph compilation  
# from [input/task.arpabo word.txt] -> G.fst
arpa2fst --disambig-symbol=#0 --read-symbol-table=lang/lang/words.txt input/task.arpabo lang/lang/G.fst
fstisstochastic lang/lang/G.fst

# from [final.mdl G.fst L.fst tree] -> HLCG.fst
utils/mkgraph.sh lang/lang mono mono/graph   

```
### 分块详解
####首先进行特征提取：
```bash
#!/bin/bash
. ./path
# 特征提取:	compute-mfcc-feats, copy-feats
# 输入为：wav.scp 		输出为:mfcc.ark,mfcc.scp
compute-mfcc-feats --verbose=2 --config="../conf/mfcc.conf" scp,p:wav.scp ark:- | copy-feats --compress=true ark:- ark,scp:mfcc/mfcc.ark,mfcc/mfcc.scp
# 计算均方归一化矩阵：
# 输入为：mfcc.ark,mfcc.scp		输出为：mfcc/cmvn.ark,mfcc/cmvn.scp
compute-cmvn-stats scp:mfcc/mfcc.scp ark,scp:mfcc/cmvn.ark,mfcc/cmvn.scp
# 计算一阶二阶差分：
# 输入为：mfcc/cmvn.ark,mfcc/cmvn.scp 	输出为：delta.ark
apply-cmvn scp:mfcc/cmvn.scp scp:mfcc/mfcc.scp ark:- | add-deltas ark:- ark:mfcc/delta.ark
```
####然后，准备训练所需的词典，音素文件，词文件等。
yesno里准备好了，直接拷贝即可。
```bash
# prepare dict for lang:
# input data 	[ lexicon_nosil.txt  lexicon.txt  phones.txt ]
# output data 	[ lexicon.txt  lexicon_words.txt  nonsilence_phones.txt  optional_silence.txt  silence_phones.txt ]
mkdir -p lang/dict
cp input/lexicon_nosil.txt lang/dict/lexicon_words.txt
cp input/lexicon.txt lang/dict/lexicon.txt
cat input/phones.txt | grep -v SIL > lang/dict/nonsilence_phones.txt
echo "SIL" > lang/dict/silence_phones.txt
echo "SIL" > lang/dict/optional_silence.txt
echo "Dictionary preparation succeeded"
```
####生成声学拓扑结构。
生成 `L.fst` `word.txt`用来生成`G.fst` `train.fst` `HCLG.fst`。其中`utils/prepare_lang.sh`所需全部输入为上一步生成的`dict`文件。
```bash
# generate [ topo ] for acoustic model
utils/gen_topo.pl 3 5 2:3 1 > lang/lang/topo
# from [lexicoin phone word] -> [L.fst word.txt] for [G.fst train.fst HCLG.fst]
utils/prepare_lang.sh --position-dependent-phones false lang/dict "<SIL>" lang/local lang/lang
```
####训练单音素模型
- 流程如下 ：
	- 利用生成的声学拓扑初始化模型
	- 生成训练图
	- 初始化对齐
	- 生成统计量
	- 模型参数估计
	- {重新对齐生，成统计量，模型参数估计}x10
	- 生成并导出最终模型：

```bash
# train monophic acoustic model
# 1.from [topo 39] -> 0.mdl tree
gmm-init-mono --train-feats=ark:mfcc/delta.ark lang/lang/topo 39 mono/0.mdl mono/tree
# 2.from [L.fst 0.mdl tree word.txt text] -> train.fst
# compile-train-graphs [options] <tree-in> <model-in> <lexicon-fst-in> <transcriptions-rspecifier> <graphs-wspecifier>
compile-train-graphs mono/tree mono/0.mdl lang/lang/L.fst 'ark:sym2int.pl -f 2- lang/lang/words.txt text|' ark:lang/lang/graphs.fsts
# 3.from [graphs.fst] equally align the train data -> [ euqal.ali ]
# align-equal-compiled <graphs-rspecifier> <features-rspecifier> <alignments-wspecifier>
align-equal-compiled ark:lang/lang/graphs.fsts ark:mfcc/delta.ark ark:mono/equal.ali
# 4.from [equal.ali delta.ark mdl] ->  [ 0.acc ]
gmm-acc-stats-ali mono/0.mdl ark:mfcc/delta.ark ark:mono/equal.ali mono/0.acc
# 5.from [0.mdl 0.acc] -> [ 1.mdl ] 
# parameter est: 
gmm-est mono/0.mdl mono/0.acc mono/1.mdl

x=1
numliter=40
numgauss=11
while [ $x -lt $numliter ]; do
	# 6.from [1.mdl graphs.fst] align the data by new model -> [ 1.ali ]
	gmm-align-compiled --beam=6 --retry-beam=20 mono/$x.mdl ark:lang/lang/graphs.fsts ark:mfcc/delta.ark ark:mono/$x.ali
	# 4.from [equal.ali delta.ark mdl] ->  [ 0.acc ]
	gmm-acc-stats-ali mono/$x.mdl ark:mfcc/delta.ark ark:mono/equal.ali mono/$x.acc
	# 5.from [x.mdl x.acc] -> [ x+1.mdl ] 
	gmm-est --mix-up=$numgauss --power=0.25 mono/$x.mdl mono/$x.acc mono/$[$x+1].mdl
	numgauss=$[$numgauss+25]
	x=$[$x+1]
done
cp mono/$x.mdl mono/final.mdl
```
####最后合成语言模型：

```bash
# Graph compilation  
# from [input/task.arpabo word.txt] -> G.fst
arpa2fst --disambig-symbol=#0 --read-symbol-table=lang/lang/words.txt input/task.arpabo lang/lang/G.fst
fstisstochastic lang/lang/G.fst

# from [final.mdl G.fst L.fst tree] -> HLCG.fst
utils/mkgraph.sh lang/lang mono mono/graph  
```
运行结果：



##建立解码脚本
解码指令较简单一个指令即可：

```bash
#Usage: gmm-latgen-faster [options] model-in (fst-in|fsts-rspecifier) features-rspecifier lattice-wspecifier [ words-wspecifier [alignments-wspecifier] ]
gmm-latgen-faster --max-active=7000 --beam=13 --lattice-beam=6 --acoustic-scale=0.083333 \
--allow-partial=true --word-symbol-table=lang/lang/words.txt mono/final.mdl \
mono/graph/HCLG.fst ark:mfcc/delta.ark "ark:|gzip -c > result/lat.gz"
```
可以得到识别结果不是很好，没关系，主要用这个例子来理解kaldi是怎么样使用工具的。
![这里写图片描述](https://img-blog.csdn.net/20180808191229506?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NoaW5hdGVsZWNvbTA4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

转载请注明出处：[https://blog.csdn.net/chinatelecom08/article/details/81392399](https://blog.csdn.net/chinatelecom08/article/details/81392399)