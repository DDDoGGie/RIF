---
sort: 1
---

# 10X Visium DLPFC

Tutorial for spatial domain identification on Visium DLPFC slice 151674.
The datasets can be download from [https://github.com/LieberInstitute/spatialLIBD](https://github.com/LieberInstitute/spatialLIBD).


```python
import os
import warnings
import argparse

import scanpy as sc
import pandas as pd
from sklearn.metrics import adjusted_rand_score
from matplotlib import pyplot as plt

import Riff
os.environ['R_HOME'] = '/usr/lib/R'
os.environ['CUDA_LAUNCH_BLOCKING'] = '1'
warnings.filterwarnings("ignore")
```

```python
parser = argparse.ArgumentParser(description="GAT")
parser.add_argument("--seeds", type=int, default=0)
parser.add_argument("--device", type=int, default=4)
parser.add_argument("--warmup_steps", type=int, default=-1)
parser.add_argument("--num_heads", type=int, default=4, help="number of hidden attention heads")
parser.add_argument("--num_out_heads", type=int, default=1, help="number of output attention heads")
parser.add_argument("--residual", action="store_true", default=False, help="use residual connection")
parser.add_argument("--in_drop", type=float, default=0.2, help="input feature dropout")
parser.add_argument("--attn_drop", type=float, default=0.1, help="attention dropout")
parser.add_argument("--weight_decay", type=float, default=2e-4, help="weight decay")
parser.add_argument("--negative_slope", type=float, default=0.2, help="the negative slope of leaky relu for GAT")
parser.add_argument("--drop_edge_rate", type=float, default=0.0)
parser.add_argument("--optimizer", type=str, default="adam")
parser.add_argument("--lr_f", type=float, default=0.01, help="learning rate for evaluation")
parser.add_argument("--weight_decay_f", type=float, default=1e-4, help="weight decay for evaluation")
parser.add_argument("--linear_prob", action="store_true", default=True)
parser.add_argument("--load_model", action="store_true")
parser.add_argument("--save_model", action="store_true")
parser.add_argument("--use_cfg", action="store_true")
parser.add_argument("--logging", action="store_true")
parser.add_argument("--scheduler", action="store_true", default=True)

# for graph classification
parser.add_argument("--pooling", type=str, default="mean")
parser.add_argument("--deg4feat", action="store_true", default=False, help="use node degree as input feature")
parser.add_argument("--batch_size", type=int, default=32)

# adjustable parameters
parser.add_argument("--encoder", type=str, default="gin")
parser.add_argument("--decoder", type=str, default="gin")
parser.add_argument("--num_hidden", type=int, default=64, help="number of hidden units")
parser.add_argument("--num_layers", type=int, default=2, help="number of hidden layers")
parser.add_argument("--activation", type=str, default="elu")
parser.add_argument("--max_epoch", type=int, default=200, help="number of training epochs")
parser.add_argument("--lr", type=float, default=0.001, help="learning rate")
parser.add_argument("--alpha_l", type=float, default=4, help="`pow`inddex for `sce` loss")
parser.add_argument("--beta_l", type=float, default=2, help="`pow`inddex for `weighted_mse` loss")   
parser.add_argument("--loss_fn", type=str, default="weighted_mse")
parser.add_argument("--mask_gene_rate", type=float, default=0.8)
parser.add_argument("--replace_rate", type=float, default=0.05)
parser.add_argument("--remask_rate", type=float, default=0.5)
parser.add_argument("--warm_up", type=int, default=50)
parser.add_argument("--norm", type=str, default="batchnorm")

# GSG parameter
parser.add_argument("--num_neighbors", type=int, default=7)
parser.add_argument("--confidence_threshold", type=float, default=3e-3)
parser.add_argument("--pre_aggregation", type=int, default=[1, 1]) 
parser.add_argument("--min_pseudo_label", type=int, default=3000)
parser.add_argument("--num_features", type=int, default=3000)
parser.add_argument("--seq_tech", type=str, default="Visuim")
parser.add_argument("--sample_name", type=str, default="151674")
parser.add_argument("--cluster_label", type=str, default= "layer_guess")
parser.add_argument("--folder_name", type=str, default="/home/wcy/code/datasets/10X/")  
parser.add_argument("--num_classes", type=int, default=12, help = "The number of clusters")
parser.add_argument("--top_num", type=int, default=10)
parser.add_argument("--radius", type=int, default=50)
```

```python
args = parser.parse_args(args=['--sample_name', '151674']) 
args
```

```python
data_path = os.path.join(args.folder_name, args.sample_name)
adata = Riff.read_10X_Visium_with_label(data_path)
if(args.cluster_label == ""):
    num_classes = args.num_classes
else:
    num_classes = adata.obs[args.cluster_label].nunique()
    adata.obs[args.cluster_label] = adata.obs[args.cluster_label].astype('category')
    
# graph construction and training
adata, graph = Riff.build_graph(args, adata)
adata, num_classes
```

```python
adata, _ = Riff.GSG_train(args, adata, graph, num_classes)

adata.obs["pred1_refine"] = Riff.refine_label(adata, args.radius, key='cluster_pred1')
adata.obs["pred2_refine"] = Riff.refine_label(adata, args.radius, key='cluster_pred2')
adata.obs["combined"] = Riff.HBGF(adata, ["pred1_refine", "pred2_refine"], num_classes, top_num=args.top_num)
adata.obs["combined_refine"] = Riff.refine_label(adata, args.radius, key='combined')
```

```python
adata, new_key = Riff.test_refine(adata, num_classes, max_neigh=args.radius, key='combined', refined_key='combined_refine')
adata.obs[new_key] = adata.obs[new_key].astype(int).astype('category') 
adata_reduce = adata[~pd.isna(adata.obs['layer_guess'])]
ari = round(adjusted_rand_score(adata_reduce.obs['layer_guess'], adata_reduce.obs[new_key]), 4)
sc.pl.spatial(adata, color=['layer_guess', new_key], title=['Manually Annotation', 'RIF    ARI: '+str(ari)], 
              s=8, frameon=False)
```

<img src='Figures/SDI/SDI_DLPFC_domain.png'>


```python
sc.pp.neighbors(adata, use_rep='Riff_embedding')
sc.tl.umap(adata)
sc.pl.umap(adata, color=['layer_guess', new_key], title=['Manually Annotation', 'RIF'], frameon=False)
```

<img src='Figures/SDI/SDI_DLPFC_umap.png'>

```python
sc.pp.neighbors(adata_reduce, use_rep='Riff_embedding')
sc.tl.paga(adata_reduce, groups='layer_guess')
sc.pl.paga(adata_reduce, fontsize=14)
```

<img src='Figures/SDI/SDI_DLPFC_paga.png'>