[toc]




##acc-tree-stats

```shell
 Accumulate statistics for phonetic-context tree building.
Usage:  acc-tree-stats [options] <model-in> <features-rspecifier> <alignments-rspecifier> <tree-accs-out>
e.g.: 
 acc-tree-stats 1.mdl scp:train.scp ark:1.ali 1.tacc 
```

##cluster-phones

```shell
 Cluster phones (or sets of phones) into sets for various purposes
Usage:  cluster-phones [options] <tree-stats-in> <phone-sets-in> <clustered-phones-out>
e.g.: 
 cluster-phones 1.tacc phonesets.txt questions.txt 
```

##compile-questions
```shell
Compile questions
Usage:  compile-questions [options] <topo> <questions-text-file> <questions-out>
e.g.: 
 compile-questions questions.txt questions.qst 
```


##build-tree
```shell
Train decision tree
Usage:  build-tree [options] <tree-stats-in> <roots-file> <questions-file> <topo-file> <tree-out>
e.g.: 
 build-tree treeacc roots.txt 1.qst topo tree 
```




##gmm-init-model

初始化三因素模型，通过

```shell
Initialize GMM from decision tree and tree stats
Usage:  gmm-init-model [options] <tree-in> <tree-stats-in> <topo-file> <model-out> [<old-tree> <old-model>]
e.g.: 
  gmm-init-model tree treeacc topo 1.mdl
or (initializing GMMs with old model):
  gmm-init-model tree treeacc topo 1.mdl prev/tree prev/30.mdl 
```

##gmm-init-mono

```shell
Initialize monophone GMM.
Usage:  gmm-init-mono <topology-in> <dim> <model-out> <tree-out> 
e.g.: 
 gmm-init-mono topo 39 mono.mdl mono.tree 
```

##gmm-mixup

```
 Does GMM mixing up (and Gaussian merging)
Usage:  gmm-mixup [options] <model-in> <state-occs-in> <model-out>
e.g. of mixing up:
 gmm-mixup --mix-up=4000 1.mdl 1.occs 2.mdl
e.g. of merging:
 gmm-mixup --merge=2000 1.mdl 1.occs 2.mdl
```

##convert-ali 

```shell
 Convert alignments from one decision-tree/model to another
Usage:  convert-ali  [options] <old-model> <new-model> <new-tree> <old-alignments-rspecifier> <new-alignments-wspecifier>
e.g.: 
 convert-ali old/final.mdl new/0.mdl new/tree ark:old/ali.1 ark:new/ali.1
```


##compile-train-graphs

```shell
 Creates training graphs (without transition-probabilities, by default)
Usage:   compile-train-graphs [options] <tree-in> <model-in> <lexicon-fst-in> <transcriptions-rspecifier> <graphs-wspecifier>
e.g.: 
 compile-train-graphs tree 1.mdl lex.fst 'ark:sym2int.pl -f 2- words.txt text|' ark:graphs.fsts 
```