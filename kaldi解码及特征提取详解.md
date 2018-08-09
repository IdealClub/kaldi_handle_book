
[toc]

---
##1. 注意事项

- **首先**要训练好模型，用到3个文件，分别是：
    - `final.mdl`(训练模型得到的模型文件)  
    - `final.mat`（用来特征转换） 
    - `HCLG.fst`（fst文件）
- **此外**要提供待解码音频文件或路径.scp文件：
    - `wav.scp`(音频路径.scp文件)  

##2. 流程图：

```flow
st=>start: 开始
op1=>operation: compute-mfcc-feats
op2=>operation: copy-feats
op3=>operation: compute-cmvn-stats
op4=>operation: apply-cmvn
op5=>operation: splice-feats
op6=>operation: transform-feats
op7=>operation: nnet-latgen-faster
st->op1->op2->op3->op4->op5->op6->op7
```
> 流程每一步意义如下：
> 
2.  使用`compute-mfcc-feats`提取特征，生成对应的特征文件`feats.ark`；
3. 使用`copy-feats`来拷贝特征文件，并创建特征的scp文件，生成`feat.scp` `feat.ark `  ；
4. 使用`compute-cmvn-stats`计算CMVN归一化，得到`cmvn.scp`  `cmvn.ark  ` ；
5. 使用`apply-cmvn`得到了`applycmvn.ark`文件；
6. 使用`splice-feats`来继续变换特征 ，拼接相邻帧的特征；
7. 使用`transform-feats`来进行特征转换，为了解码调用 ；
8. 最后通过得到的`transform.ark`进行解码的操作，得到解码后的lattice文件 。

## 3. 具体流程指令：

1.  首先列出具体文件，这里我就按照自己的文件给出了，如果用别的，改相应文件就行了
    2. `wav.scp`(里面是保存了wav的绝对路径) 
    3. `final.mdl`(训练模型得到的模型文件)  
    4. `final.mat`（用来特征转换）  
    5. `HCLG.fst`（fst文件，用于解码）  
2.  使用compute-mfcc-feats生成对应的特征文件feats.ark:  
    `compute-mfcc-feats --use-energy=false scp:wav.scp ark:feats.ark`  
3. 使用copy-feats来拷贝特征文件，并创建特征的scp文件，生成feat.scp feat.ark   
    `copy-feats ark:feats.ark ark,scp:feat.ark,feat.scp`  
4. 使用compute-cmvn-stats计算CMVN归一化，得到cmvn.scp  cmvn.ark   
    `compute-cmvn-stats scp:feat.scp ark,scp:cmvn.ark,cmvn.scp`
5. 使用apply-cmvn，得到了applycmvn.ark文件   
    `apply-cmvn scp:cmvn.scp scp:feat.scp ark:applycmvn.ark`    
6. 使用splice-feats来继续变换特征   
    `splice-feats --left-context=3 --right-context=3 ark:applycmvn.ark ark:splice.ark`  
7. 使用transform来进行特征转换，为了解码调用  
    `transform-feats final.mat ark:splice.ark ark:transform.ark`
8. 最后通过得到的transform.ark进行解码的操作，得到一个晶格文件  
   `nnet-latgen-faster [options] <nnet-in> <fst-in|fsts-rspecifier> <features-rspecifier> <lattice-wspecifier>`

> 版权声明：转载请注明出处，谢谢	