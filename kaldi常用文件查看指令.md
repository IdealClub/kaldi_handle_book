
[TOC]

> **资料来自kaldi[官方文档](http://www.kaldi-asr.org/doc/tools.html)**。
> **转载注明出处。**

# 1.  ark特征文件
`copy-feats` 可以用来改变特征数据的格式，因此可以转换ark格式文件为txt格式：
用法: `  copy-feats [options] <feats-rxfilename> <feats-wxfilename>`

**例子：**
先查找`copy-feats`的目录（每个人可能不一样）：`find /home/speech.AI/kaldi/ -name copy-feats`
得到`copy-feats`的目录：
```bash
/home/speech.AI/kaldi/src/featbin/copy-feats
```
然后执行指令：
`~/kaldi/src/featbin/copy-feats ark:foo.ark ark,t:foo.txt`  
ark存的是二进制文件，该指令为复制ark文件至txt文件下。

# 2.  FST文件 

查找`fstprint`的目录（每个人可能不一样）：`find /home/speech.AI/kaldi/ -name fstprint `
得到`fstprint`所在目录：
```bash
/home/speech.AI/kaldi/tools/openfst-1.6.7/src/bin/.libs/fstprint
/home/speech.AI/kaldi/tools/openfst-1.6.7/src/bin/fstprint
/home/speech.AI/kaldi/tools/openfst-1.6.7/bin/fstprint
```
使用`fstprint`打印fst为文本格式：
	`~/kaldi/tools/openfst-1.6.7/bin/fstprint --isymbols=phones.txt --osymbols=words.txt L.fst L.txt`    

同理可以查看pdf格式的图：
`fstdraw [--isymbols=phones.txt --osymbols=words.txt] L.fst | dot –Tps  |  ps2pdf – L.pdf`    
例子：
`~/kaldi/tools/openfst-1.6.7/bin/fstdraw --isymbols=phones.txt --osymbols=words.txt HCLG.fst`  

#3.  mdl模型文件
**gmm模型查看指令: **` gmm-copy [options] <model-in> <model-out>`
如:` gmm-copy --binary=false 1.mdl 1_txt.mdl`

**实例：**
查找`gmm-copy`的目录：`find /home/speech.AI/kaldi/ -name gmm-copy`
得到`gmm-copy`所在目录（每个人可能不一样）：
```bash
/home/speech.AI/kaldi/src/gmmbin/gmm-copy
```
`~/kaldi/src/gmmbin/gmm-copy --binary=false final.mdl final.txt`  

**dnn模型查看用`nnet-copy`:**
`~/kaldi/src/nnetbin/nnet-copy --binary=false 0.mdl final.txt`


#4.  决策树文件

	
 转化为文本格式指令为： 
`copy-tree [--binary=false] <tree-in> <tree-out>`
如：
`copy-tree [--binary=false] tree tree.txt>`

转化为图形格式指令为：
`draw-tree [options] <phone-symbols> <tree>`
如:
` draw-tree phones.txt tree | dot -Gsize=8,10.5 -Tps | ps2pdf - tree.pdf `



#5.   ali.gz对齐文件
对齐文件可以通过`copy-int-vector`查看：
` copy-int-vector [options] (vector-in-rspecifier) (vector-out-wspecifier)`
**实例：**
`~/kaldi/src/bin/copy-int-vector "ark:gunzip -c ali.1.gz|" ark,t:ali.txt`  

也可以先解压，然后用`show-alignments`查看 :
`show-alignments  [options] <phone-syms> <model> <alignments-rspecifier>`
**实例**：
`~/kaldi/src/bin/show-alignments phones.txt final.mdl ark:ali.1 > ali.1.txt`  

类似的有: `ali-to-phones`, `copy-int-vector` 

----
> 转载请注明出处
> 谢谢