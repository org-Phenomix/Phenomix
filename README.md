<div align="center">

# Phenomix

**Explainable multimodal phenotypic drug discovery via Phenomix**

</div>

<p align="center">
  <img src="picture/phenomix-overview.png" alt="Overview of the Phenomix framework and its downstream applications" width="1200">
</p>

## Overview

Phenomix is a multi-modal biologically informed explainable AI framework that translates chemical-perturbation-induced cell morphology and transcriptome into mechanistic and therapeutic insights for phenotypic drug discovery. Phenomix can classify drugs, elucidate compound-associated MoA, perform mechanism-guided drug repurposing, and identify high-confidence hits in an XAI way, especially for undruggable targets. Evaluated on the large-scale chemical-perturbation-induced transcriptome and Cell Painting based phenotype screening datasets, the capacity of Phenomix are extensively demonstrated and validated.

By providing mechanistic insights into compound-induced phenotypes, Phenomix helps transform vast collections of compounds from largely unexplored chemical space into actionable therapeutic knowledge, enabling researchers to more efficiently uncover promising drug candidates and their underlying biological mechanisms. We anticipate that Phenomix will contribute to the continued advancement of the phenotypic drug discovery field.


## Model architecture

For an input vector `x = [x_CP, x_GE]`, the network follows four steps:

1. **Reactome-informed GE branch.** `PhenomixNet` creates a linear gene-to-pathway layer. Its weight mask is built from the supplied Reactome mapping, and connections that are not supported by a known gene-pathway relationship are pruned.
2. **Morphology branch.** A fully connected layer maps CP features to a 100-dimensional morphology embedding.
3. **Multimodal fusion.** The pathway representation and morphology embedding are concatenated.
4. **MoA score.** A final linear layer followed by a sigmoid returns a score for the target MoA classifier.

The architecture is biologically informed without forcing the CP features into a pathway space. This preserves complementary morphological information while making the transcriptomic branch easier to interpret.

### One-vs-all training

The tutorial trains a separate binary model for each selected MoA. For a target class, compounds in that class are assigned label `1` and all other compounds are assigned label `0`. The default tutorial recipe uses random oversampling for class balance, binary cross-entropy, and an Adam optimizer. At inference time, the class with the largest score is selected.

When novel-MoA identification is enabled, a prediction is labeled `Novel drug` when the largest one-vs-all score is below the configured threshold. The tutorial uses `NOVEL_DRUG_THR = 0.5`; The default cut-off can be adjusted by users as needed.


## Explainability outputs

`Phenomix.moa_explanation` returns a dictionary of `MoAResult` objects. For each target MoA, the result contains:

- `CP_MoA_pattern`: a broad morphology pattern such as `Nuclei` and `RNA/Protein` pattern.
- `CP_MoA_feature`: the most frequent renamed feature among the top Cell Painting attributions.
- `CP_MoA_feature_occurrence_count`: the number of occurrences supporting that CP feature.
- `top10_genes`: the ten most frequently attributed genes for the target class or compound.
- `top10_pathways`: the ten most frequently attributed Reactome pathways for the target class or compound, which are considered the key MoA-associated pathways.

The explanation workflow uses:

- `IntegratedGradients` to attribute the model output to the concatenated CP and GE input features.
- `LayerIntegratedGradients` on `model.ge_layer` to attribute the output to Reactome pathway units.
- `moa_cp_explain_analysis` to group top CP features into interpretable morphology patterns.

Set `EXPLAIN_FOR_INDIVIDUAL_DRUG = True` in the tutorial to collect the same type of attribution summary for individual compounds within a MoA class.

## Repository structure

```text
.
├── Phenomix.py             # Neural network, dataset wrapper, and explanation functions
├── Phenomix_utils.py       # Reactome mapping and label/probe utilities
├── readProfiles.py         # Cell Painting/L1000 loading and preprocessing
├── reactome.py             # Reactome hierarchy and pathway helpers
├── tutorial.ipynb          # End-to-end LINCS walkthrough
├── requirements.txt        # Recorded runtime dependency bounds
├── picture/
│   └── phenomix-overview.png
└── dataset/
    ├── RepCorrDF.xlsx      # Replicate-correlation support data
    ├── idmap.csv           # L1000 probe-to-gene mapping
    └── Reactome/           # Pathway names, hierarchy, and gene memberships
```

## Installation

Clone the repository and create an environment for the recorded dependencies:

```bash
git clone https://github.com/org-Phenomix/Phenomix.git
cd Phenomix
```

Setup a Python virtual environment (recommended)

* Create the virtual environment: 
```conda create -n phenomix_env python=3.9```

* Activate the environment:
```conda activate phenomix_env```

* Install all the required packages in the virtual environment (this should take a few minutes):  
```pip --no-cache-dir install -r requirements.txt```  
Packages can also be installed individually using the versions 
provided in the ```requirements.txt``` file; for example:
```pip install pandas==1.5.3```

## Prepare the input data

The LINCS and CDRP-bio datasets with matched gene expression and cell painting data used in our study are publicly available at [carpenter-singh-lab/2022_Haghighi_NatureMethods](https://github.com/carpenter-singh-lab/2022_Haghighi_NatureMethods).

The loader recognizes the following dataset keys and directory names:

| Dataset key | Expected directory |
| --- | --- |
| `LINCS` | `LINCS-Pilot1` |
| `CDRP-bio` | `CDRPBIO-BBBC036-Bray` |



## Quick start

The reference workflow is in [tutorial.ipynb](tutorial.ipynb). The core data-loading step is:

```python
from readProfiles import read_paired_treatment_level_profiles

dataset = "LINCS"
procProf_dir = "/absolute/path/to/data-root"
profileType = "normalized_variable_selected"
filter_repCorr_params = [
    "highRepUnion_and_negcon",
    "dataset/RepCorrDF.xlsx",
]

merged_profiles, cp_features, l1k_features = \
    read_paired_treatment_level_profiles(
        procProf_dir,
        dataset,
        profileType,
        filter_repCorr_params,
        per_plate_normalized_flag=1,
    )
```

Build the Reactome mapping and model after the tutorial preprocessing has assembled the CP and gene-expression columns:

```python
from Phenomix import PhenomixNet
from Phenomix_utils import get_layer_maps

# The LINCS tutorial has 119 CP features after its selected-feature preprocessing.
cp_dim = len(cp_features)
genes = data.columns[cp_dim:-2].tolist()
gene_dim = len(genes)
reactome_knowledge = get_layer_maps(genes)[0]

model = PhenomixNet(
    reactome_knowledge=reactome_knowledge,
    gene_dim=gene_dim,
    cp_dim=cp_dim,
)
```

The default demonstration trains models for `MEK inhibitor` and `PLK inhibitor`. To train all retained MoA classes, replace the selected class list in the notebook with the unique values in `data['Metadata_moa_num']`.

## Citation
Shaoqi Chen et al. Explainable multimodal phenotypic drug discovery via Phenomix, biorxiv, 2026.

## Contacts
bm2-lab@tongji.edu.cn

csq_@tongji.edu.cn
