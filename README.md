# Structure-Aware Transformer

__Updates: We have added the script for [model visualization](#model-visualization) (Figure 4 in our paper)!__

The repository implements the Structure-Aware Transformer (SAT) in Pytorch Geometric described in the following paper

>Dexiong Chen*, Leslie O'Bray*, and Karsten Borgwardt.
[Structure-Aware Transformer for Graph Representation Learning][1]. ICML 2022.
<br/>*Equal contribution

**TL;DR**: A class of simple and flexible graph transformers built upon a new self-attention mechanism, which incorporates structural information into the original self-attention by extracting a subgraph representation rooted at each node before computing the attention. Our structure-aware framework can leverage any existing GNN to extract the subgraph representation and systematically improve the peroformance relative to the base GNN.

## Citation

Please use the following to cite our work:

```bibtex
@InProceedings{Chen22a,
	author = {Dexiong Chen and Leslie O'Bray and Karsten Borgwardt},
	title = {Structure-Aware Transformer for Graph Representation Learning},
	year = {2022},
	booktitle = {Proceedings of the 39th International Conference on Machine Learning~(ICML)},
	series = {Proceedings of Machine Learning Research}
}
```


## A short description of SAT

### SAT vs the vanilla Transformer

![SAT vs Transformer](images/sat_vs_transformer.png)

The SAT architecture compared with the vanilla transformer architecture is shown above. We make the self-attention calculation in each transformer layer *structure-aware* by leveraging structure-aware node embeddings. We generate these embeddings using a structure extractor (for example, any GNN) on the $k$-hop subgraphs centered at each node of interest. Then, the updated node embeddings are used to compute the query ($\mathbf{Q}$) and key ($\mathbf{K}$) matrices. We provide example structure extractors in the next figure.

### Example structure extractors

![Overview figure](images/structure_extractor.png)

The figure above shows the two example structure extractors used in our paper ($k$-subtree and $k$-subgraph). Structure-aware node representations are generated in the $k$-subtree GNN extractor by using the $k$-hop subtree centered at each node (here, $k=1$) and using a GNN to generate updated node representations. The explicit extraction of the subtree as an initial step is not strictly necessary, as a GNN by nature will use the $k$-hop subtree and generate updated node embeddings using the subtree information. For the $k$-subgraph GNN extractor, we first extract the $k$-hop subgraph centered at each node, and then use a GNN on each subgraph to generate node representations using the full subgraph information. The updated node embeddings are then used to compute the query ($\mathbf{Q}$) and key ($\mathbf{K}$) matrices shown in the first figure.

### A quick-start example

Below you can find a quick-start example on the ZINC dataset, see `./experiments/train_zinc.py` for more details.

<details><summary>click to see the example:</summary>

```python
import torch
from torch_geometric import datasets
from torch_geometric.loader import DataLoader
from sat.data import GraphDataset
from sat import GraphTransformer

# Load the ZINC dataset using our wrapper GraphDataset,
# which automatically creates the fully connected graph.
# For datasets with large graph, we recommend setting return_complete_index=False
# leading to faster computation
dset = datasets.ZINC('./datasets/ZINC', subset=True, split='train')
dset = GraphDataset(dset)

# Create a PyG data loader
train_loader = DataLoader(dset, batch_size=16, shuffle=True)

# Create a SAT model
dim_hidden = 16
gnn_type = 'gcn' # use GCN as the structure extractor
k_hop = 2 # use a 2-layer GCN

model = GraphTransformer(
    in_size=28, # number of node labels for ZINC
    num_class=1, # regression task
    d_model=dim_hidden,
    dim_feedforward=2 * dim_hidden,
    num_layers=2,
    batch_norm=True,
    gnn_type='gcn', # use GCN as the structure extractor
    use_edge_attr=True,
    num_edge_features=4, # number of edge labels
    edge_dim=dim_hidden,
    k_hop=k_hop,
    se='gnn', # we use the k-subtree structure extractor
    global_pool='add'
)

for data in train_loader:
    output = model(data) # batch_size x 1
    break
```
</details>

## Installation

The dependencies are managed by [miniconda][2]

```
python=3.9
numpy
scipy
pytorch=1.9.1
#pip install torch-geometric
pytorch-geometric=2.0.2
einops
ogb
conda install pytorch-scatter -c pyg
```

Once you have activated the environment and installed all dependencies, run:

```bash
source s
```

Datasets will be downloaded via Pytorch geometric and OGB package.

## Train SAT on graph and node prediction datasets

All our experimental scripts are in the folder `experiments`. So to start with, after having run `source s`, run `cd experiments`. The hyperparameters used below are selected as optimal

#### Graph regression on ZINC dataset

Train a k-subtree SAT with PNA:
```bash
python train_zinc.py --abs-pe rw --se gnn --gnn-type pna2 --dropout 0.3 --k-hop 3 --use-edge-attr
```

Train a k-subgraph SAT with PNA
```bash
python train_zinc.py --abs-pe rw --se khopgnn --gnn-type pna2 --dropout 0.2 --k-hop 3 --use-edge-attr
```

#### Node classification on PATTERN and CLUSTER datasets

Train a k-subtree SAT on PATTERN:
```bash
python train_SBMs.py --dataset PATTERN --weight-class --abs-pe rw --abs-pe-dim 7 --se gnn --gnn-type pna3 --dropout 0.2 --k-hop 3 --num-layers 6 --lr 0.0003
```

and on CLUSTER:
```bash
python train_SBMs.py --dataset CLUSTER --weight-class --abs-pe rw --abs-pe-dim 3 --se gnn --gnn-type pna2 --dropout 0.4 --k-hop 3 --num-layers 16 --dim-hidden 48 --lr 0.0005
```

#### Graph classification on OGB datasets

`--gnn-type` can be `gcn`, `gine` or `pna`, where `pna` obtains the best performance.

```bash
# Train SAT on OGBG-PPA
python train_ppa.py --gnn-type gcn --use-edge-attr
```

```bash
# Train SAT on OGBG-CODE2
python experiments/train_code2.py --gnn-type gcn --use-edge-attr --outdir ./logs --batch-size 4
```

## Model visualization

We showcase here how to visualize the attention weights of the [CLS] node learned by SAT and vanilla Transformer with the random walk positional encoding. We have provided the pre-trained models on the Mutagenecity dataset. To visualize the pre-trained models, you need to install the [`networkx`](https://networkx.org/) and `matplotlib` packages, then run:

```bash
python model_visu.py --graph-idx 2003
```

This will generate the following image, the same as the Figure 4 in our paper:

![Model_interpretation](images/graph2003.png)


[1]: https://arxiv.org/abs/2202.03036
[2]: https://conda.io/miniconda.html
