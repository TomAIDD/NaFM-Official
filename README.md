# NaFM: Pre-training a Foundation Model for Small-Molecule Natural Products
**Abstract**:
Natural products, as metabolites derived from microorganisms, animals, or plants, exhibit diverse biological activities, making them indispensable resources for drug discovery. Currently, most deep learning approaches in natural product research are based on supervised learning tailored to specific downstream tasks. However, the one-model-one-task paradigm often lacks generalization capability and still leaves much room for performance improvement. Moreover, conventional molecular representation techniques are not well-suited to the unique structural and evolutionary features of natural products.
To address these challenges, we introduce NaFM, a foundation model specifically pre-trained on natural products. Our method integrates contrastive learning with masked graph modeling, effectively encoding scaffold-derived evolutionary patterns alongside diverse side-chain information. The proposed framework achieves state-of-the-art (SOTA) performance across a wide range of downstream tasks in natural product mining and drug discovery.
We first benchmark NaFM on taxonomy classification against models pre-trained on synthetic molecules, demonstrating their inadequacy for capturing natural synthesis patterns. Through detailed analysis at both gene and microbial levels, NaFM reveals a strong capacity for learning evolutionary information. Finally, we apply NaFM to virtual screening tasks, showing its potential to provide meaningful molecular representations and facilitate the discovery of novel bioactive compounds.

## Environment Setup

Before running the code, please configure the Python environment. We provide two options:

### Option 1: One-click Conda Installation

This method is fast and convenient but may face compatibility issues depending on server configuration.

`conda env create -f NaFM-Official.yml`

### Option 2: Manual Installation

This method is more stable and compatible across systems:

```
conda create -n nafm python=3.9
conda activate nafm
conda install pytorch==2.4.1 torchvision==0.19.1 torchaudio==2.4.1 pytorch-cuda=12.1 -c pytorch -c nvidia
conda install tensorboard
conda install tqdm
pip install lightning==2.4.0
pip install numpy==1.23.0
pip install pandas==2.2.2
pip install torch_geometric
pip install pyg_lib torch_scatter torch_sparse torch_cluster torch_spline_conv -f https://data.pyg.org/whl/torch-2.4.0+cu121.html
pip install scikit-learn==1.5.1
pip install networkx==2.4.1
pip install scipy==1.13.1
```

If you're using older CUDA versions, please install compatible versions of PyTorch, PyTorch Lightning, and PyG manually.

## Data and checkpoint Preparation

### Automated Setup (Recommended)

We provide automated scripts to download and organize all required data:

**Quick Setup (Recommended):**
```bash
# One-command complete setup and validation
python setup.py
```

**Seperate Setup:**
```bash
# Download all data and weights
python scripts/setup_data.py

# Skip specific downloads if needed
python scripts/setup_data.py --skip-zenodo    # Skip pre-trained weights
python scripts/setup_data.py --skip-figshare  # Skip datasets

# Verify existing setup
python scripts/setup_data.py --verify-only
```

### Manual Setup

If you prefer manual setup, download the files from:

- **Pre-trained weights**: [Zenodo](https://zenodo.org/records/15385335) → `NaFM.ckpt`
- **Datasets**: [Figshare](https://doi.org/10.6084/m9.figshare.28980254.v1) → Extract to appropriate directories

Place the files as follows:
```
NaFM/
├── NaFM.ckpt                          # Pre-trained weights (from Zenodo)
├── raw_data/
│   └── raw/
│       └── pretrain_smiles.pkl        # Pre-training SMILES data
├── downstream_data/
│   ├── Ontology/
│   │   └── raw/classification_data.csv
│   ├── Regression/
│   │   └── raw/regression_data.csv
│   ├── Lotus/
│   │   └── raw/lotus_data.csv
│   ├── Bgc/
│   │   └── raw/bgc_data.csv
│   └── External/
│       └── raw/external_data.csv
```

### Verification

After setup, verify your installation:

```bash
python scripts/validate_setup.py
```

This will check:
- Python version compatibility
- All dependencies
- Data file integrity  
- Model loading capability

## Scripts Directory

The `scripts/` directory contains utility scripts for setup and validation:

- `scripts/setup_data.py` - Automated data download and organization
- `scripts/validate_setup.py` - Comprehensive setup validation  
- `scripts/run_setup.py` - Combined setup and validation
- `scripts/README.md` - Detailed script documentation

See `scripts/README.md` for complete script documentation and options.

## Pre-training

We provide recommended pretraining hyperparameters in examples/Pretrain.yml. Use the following command:

```
python train.py --conf examples/Pretrain.yml
```

You can also use your own SMILES files for custom pretraining. First, convert your SMILES data to a `.csv` file and place it in `raw_data/raw`. Then run:

```
cd NaFM/raw_data/raw
python filter.py
```
This will standardize SMILES, remove salt and duplicate atoms, and generate `pretrain_smiles.pkl`.

## Downstream Tasks

The following sections demonstrate how to run natural product classification and bioactivity prediction tasks. These two tasks serve as representative examples for classification and regression tasks respectively.

### Natural Product Classification

Supports hierarchical classification at **Class**, **Superclass**, and **Pathway** levels. Finetuning with pretrained weights can be done using:

```
python train.py --task finetune \
--num-epochs 300 \
--emb-dim 1024 \
--feat-dim 512 \
--num-layer 6 \
--drop-ratio 0.15 \
--dataset Ontology \
--dataset-root downstream_data/Ontology \
--pretrained-path [Your pretrained model path] \
--lr 1.0e-4 \
--lr-min 1.0e-5 \
--batch-size 256 \
--save-interval 5 \
--early-stopping-patience 50 \
--dataset-arg Class \
--log-dir [your finetuned model path] \
--seed 0
```

Or directly use our config file:

`python train.py --conf examples/Finetune.yml`

For inference on new molecules (CSV with a "SMILES" column):

```
python inference.py --task classification \
--downstream-data [data location] \
--checkpoint-path [your finetuned model path]
```

Results will be saved to `NaFM/predictions.csv`.

### Bioactivity Regression

Using the regression data we provided, you can run regression finetuning with:

```
python train.py --task finetune \
--num-epochs 300 \
--emb-dim 1024 \
--num-layer 6 \
--drop-ratio 0.1 \
--dataset Regression \
--dataset-root downstream_data/Regression \
--pretrained-path cosine-0.2.ckpt \
--lr 5.0e-5 \
--lr-min 1.0e-5 \
--batch-size 128 \
--save-interval 5 \
--log-dir [your finetuned model path] \
--early-stopping-patience 50 \
--dataset-arg 178 \
--seed 0
```

For inference on new SMILES:

```
python inference.py --task regression \
--downstream-data [data location] \
--checkpoint-path [your finetuned model path] \
--num-classes [data classes]
```

Output will be saved in `NaFM/predictions.csv`.

### important Notes on Evaluation Scripts

The repository includes lightweight demonstration scripts intended to illustrate the basic inference workflow and input/output usage of NaFM.

In particular, ```test.py``` should be regarded as a minimal demonstration template rather than the exact production evaluation pipeline used to generate the benchmark results reported in the paper.

**Note:** The configs under ```examples/``` are example settings used for running the provided tasks. Some downstream tasks may require minor adjustment of parameters such as learning rate, training epochs, or early stopping patience depending on the dataset and training environment.


