# embedding-disruptiveness

A Python package and workflow for measuring how disruptive a paper or patent is, using graph embeddings on citation networks.

This repository trains node2vec-style embeddings from citation graphs and computes an *Embedding Disruptiveness Measure (EDM)* that captures whether a work disrupts or consolidates its field. It also provides the classic *Disruption Index (DI)* as a built-in utility.

[**Paper**](https://arxiv.org/abs/2502.16845) | [**Blog Post**](https://munjungkim.github.io/embedding-disruptiveness-blog/)

---

## Table of Contents

- [pip Package](#pip-package)
- [Environment Setup](#environment-setup)
- [Input Format](#input-format)
- [Usage with Snakemake](#usage-with-snakemake)
- [Usage without Snakemake](#usage-without-snakemake)
  - [Step 1: Embedding Calculation](#step-1-embedding-calculation)
  - [Step 2: Distance Calculation](#step-2-distance-calculation)
  - [Step 3: Disruption Index](#step-3-disruption-index)
- [Project Structure](#project-structure)
- [Authors](#authors)
- [Citation](#citation)
- [License](#license)

---

## pip Package

We also provide a standalone pip package for computing the Embedding Disruptiveness Measure:

```bash
pip install embedding-disruptiveness
```

- **Source code**: [github.com/MunjungKim/embedding-disruptiveness](https://github.com/MunjungKim/embedding-disruptiveness)
- **Example notebook**: [`notebooks/embedding-disruptiveness package.ipynb`](https://github.com/yy/embedding-disruptiveness/blob/main/notebooks/embedding-disruptiveness%20package.ipynb)

---

## Environment Setup

### Using uv (recommended)

```bash
uv venv
source .venv/bin/activate
uv pip install -r requirements.txt
```

### Using pip

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

---

## Input Format

Your citation network should be a **scipy sparse matrix** saved as `.npz`. Rows and columns represent nodes (papers or patents), and non-zero entries represent citation edges.

```python
import scipy.sparse as sp

net = sp.csr_matrix(adjacency_data)
sp.save_npz("citation_net.npz", net)
```

---

## Usage with Snakemake

[Snakemake](https://snakemake.readthedocs.io/) automates the full pipeline — embedding, distance calculation, and disruption index computation. This is the recommended way to run the workflow since it handles parameter management and dependency resolution automatically.

From the `workflow/embedding_computation/` directory:

```bash
# Compute embedding distances for a specific configuration
snakemake '{data_dir}/{network_name}/{dim}_{win}_q_{q}_ep_{ep}_bs_{bs}_embedding/distance.npy' -j

# Run the full pipeline (all targets defined in the rule `all`)
snakemake -j
```

Snakemake resolves the dependency chain automatically:

1. **`embedding_all_network`** — trains node2vec embeddings, producing `in.npy` and `out.npy`
2. **`calculating_distance`** — computes cosine distances from the embedding vectors
3. **`calculating_disruption`** / **`calculating_disruption_nok`** / **`calculating_disruption_mutistep`** — computes disruption indices

Parameters (embedding dimension, window size, q, epochs, batch size, etc.) are configured in `config.yaml` and the Snakefile wildcards.

---

## Usage without Snakemake

### Step 1: Embedding Calculation

Train node2vec embeddings on a citation network:

```bash
python3 scripts/Embedding.py <network.npz> <dim> <window> <device1> <device2> <name> <q> <epochs> <batch_size> <work_dir>
```

| Parameter | Description |
|-----------|-------------|
| `network.npz` | Path to the citation network (scipy sparse `.npz`) |
| `dim` | Embedding dimension (e.g., `100`, `200`, `300`) |
| `window` | Context window size (e.g., `1`, `3`, `5`) |
| `device1` | CUDA device for in-vectors (e.g., `0`) |
| `device2` | CUDA device for out-vectors (e.g., `1`) |
| `name` | Network name identifier |
| `q` | Node2Vec return parameter |
| `epochs` | Number of training epochs |
| `batch_size` | Training batch size |
| `work_dir` | Working directory for output |

**Example:**

```bash
python3 scripts/Embedding.py /data/derived/APS/citation_net.npz 200 5 0 1 derived/APS 1 5 1024 /data
```

This saves in-vectors and out-vectors to:
- `{work_dir}/{name}/{dim}_{win}_q_{q}_ep_{ep}_bs_{bs}_embedding/in.npy`
- `{work_dir}/{name}/{dim}_{win}_q_{q}_ep_{ep}_bs_{bs}_embedding/out.npy`

### Step 2: Distance Calculation

Compute cosine distances from the embedding vectors:

```bash
python3 scripts/Distance_disruption.py distance <in.npy> <out.npy> <network.npz> <device> None
```

**Example:**

```bash
python3 scripts/Distance_disruption.py distance \
    /data/derived/APS/200_5_q_1_ep_5_bs_1024_embedding/in.npy \
    /data/derived/APS/200_5_q_1_ep_5_bs_1024_embedding/out.npy \
    /data/derived/APS/citation_net.npz \
    cuda:0 \
    None
```

### Step 3: Disruption Index

Compute the classic disruption index (and variants) from the citation network:

```bash
# First, generate reference/citation dictionaries
python3 scripts/reference_citation_dict.py <network.npz>

# Then compute disruption indices
python3 scripts/Distance_disruption.py disruption <ref_dict.pkl> <cit_dict.pkl> <network.npz> None None
python3 scripts/Distance_disruption.py disruption_nok <ref_dict.pkl> <cit_dict.pkl> <network.npz> None None
python3 scripts/Distance_disruption.py multistep <ref_dict.pkl> <cit_dict.pkl> <network.npz> None multistep
```

---

## Project Structure

```
├── workflow/embedding_computation/
│   ├── Snakefile              # Snakemake workflow definition
│   ├── config.yaml            # Workflow configuration
│   └── scripts/
│       ├── Embedding.py               # Node2Vec embedding training
│       ├── Distance_disruption.py     # Distance and disruption computation
│       ├── Configuration_network.py   # Random network generation
│       └── reference_citation_dict.py # Reference/citation dict builder
├── libs/
│   ├── node2vec/              # Node2Vec implementation (models, loss, datasets, random walks)
│   └── util/                  # Utility functions (data loading, disruption calculation)
├── notebooks/                 # Example notebooks
├── requirements.txt           # Python dependencies
└── README.md
```

---

## Authors

- [Munjung Kim](https://github.com/MunjungKim)
- [Sadamori Kojaku](https://github.com/skojaku)
- [Yong-Yeol Ahn](https://github.com/yy)

## Citation

If you use this code in your research, please cite:

```bibtex
@article{kim2025uncovering,
  title={Uncovering simultaneous breakthroughs with a robust measure of disruptiveness},
  author={Kim, Munjung and Kojaku, Sadamori and Ahn, Yong-Yeol},
  journal={arXiv preprint arXiv:2502.16845},
  year={2025}
}
```

## License

MIT License. See [LICENSE](LICENSE) for details.
