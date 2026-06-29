# Spatial organization of the breast tumor microenvironment

**Spatial domains, cell-type composition, and tissue niches of a 10x Visium breast-cancer slide — using a graph neural network implemented from scratch, probabilistic deconvolution, and spatial statistics.**

This project asks a simple question with a spatial answer: *how is molecular state organized across the architecture of a breast tumor?* It identifies spatial domains directly from the tissue graph, resolves the cell types composing each region, quantifies how those cell types are arranged, and connects molecular state back to tissue appearance (H&E).

> Status: complete (analysis). Four notebooks, end to end.

---

## Why this project

My PhD studies how the *mechanical and morphological* organization of multicellular systems shapes their behavior. This project is the molecular counterpart of that idea: instead of asking how spatial structure shapes mechanics, it asks how spatial structure shapes the transcriptome. The methods (graph neural networks, variational deconvolution, spatial statistics) are the transferable core; the breast-cancer biology is the application.

It is also a deliberate methods-learning piece — the spatial-domain step is a graph autoencoder written by hand in PyTorch Geometric rather than a black-box package call, so the message-passing mechanism is fully exposed.

---

## Data

- **Spatial:** 10x Genomics Visium human breast cancer (`V1_Breast_Cancer_Block_A_Section_1`) — spot x gene counts, spot coordinates, and the H&E histology image. Each spot is ~55 um (~1-10 cells), so spot expression is a *mixture* of cell types.
- **Reference (for deconvolution):** Wu et al. 2021 breast-cancer single-cell atlas (GEO `GSE176078`, ~100k cells, `celltype_major` annotation), the standard reference for the breast tumor microenvironment.

Both are public; the notebooks download them.

---

## Approach

| Notebook | Step | Method |
|---|---|---|
| `01` | Spatial domains | QC (MAD-based) -> PCA -> spatial graph -> **graph autoencoder (PyTorch Geometric)** -> Leiden clustering |
| `02` | Cell-type deconvolution | **CondSCVI + DestVI** (scvi-tools) against the reference atlas -> per-spot cell-type proportions |
| `03` | Niches + morphology | neighborhood enrichment, co-occurrence vs distance, H&E texture features |
| `04` | Going further | marker genes per domain, tumor invasive front (EMT/unjamming), morphology->domain classifier |

**The graph neural network (Notebook 1).** Visium spots form a hexagonal lattice, so the tissue is a graph: spots are nodes, spatial adjacency defines edges, 50 PCA components are node features. A two-layer graph autoencoder encodes each spot into a 16-dim embedding via message passing — each spot aggregates information from its neighbors, so neighboring spots acquire similar embeddings — and a linear decoder reconstructs the input (self-supervised; no labels). Clustering the embeddings yields *spatially coherent* domains, an upgrade over clustering raw expression.

**Deconvolution (Notebook 2).** Each spot's expression is modeled as a mixture of cell-type signatures learned from the reference atlas: CondSCVI learns the signatures, DestVI estimates per-spot proportions. Linking proportions back to the domains gives each domain a cell-type identity.

---

## Results

**Notebook 1 - spatial domains.** The graph autoencoder recovers 11 spatial domains (`resolution = 0.5`) that are spatially compact on the tissue and well separated in a UMAP of the embeddings — the two views agree, indicating the representation captures real structure.

**Notebook 2 - cell-type deconvolution.** Cell types occupy spatially distinct territories (Cancer Epithelial in compact tumor masses, CAFs in stromal bands, Myeloid in localized foci). Averaging proportions within each domain assigns clear identities: several Cancer-Epithelial tumor domains, a Normal-Epithelial domain, CAF-rich stroma, and a single Plasmablast-dominated domain (a localized plasma-cell aggregate). Whole-slide mean composition is biologically sensible (Cancer Epithelial most abundant, ~28%).

**Notebook 3 - spatial niches and morphology.** Quantitative neighborhood analysis reveals a **compartmentalized tissue with immune exclusion**, supported by three convergent lines of evidence:
- **Proportion correlation (Spearman):** Cancer Epithelial is anti-correlated with T-cells (-0.29) and, most strongly, with Plasmablasts (-0.64). T-cells co-localize with Myeloid (+0.33) and CAFs (+0.26) — lymphocytes reside in a stromal/immune niche, not inside tumor masses.
- **Neighborhood enrichment (z-score):** independently confirms strong tumor-plasmablast segregation.
- **Co-occurrence vs distance:** B-/T-cell enrichment is lowest adjacent to the tumor and rises with distance — the signature of peripheral immune exclusion ("cold tumor").

*Morphology bridge:* per-spot H&E features (intensity + Haralick texture) differ markedly across molecular domains — tissue appearance carries a signature of molecular identity.

**Notebook 4 - going further (hypothesis-generating).**
- **Marker genes per domain** (full transcriptome) give each domain an independent gene-level identity matching its cell-type composition (immunoglobulin-rich lymphoid domains, keratin-positive tumor domains, decorin/vimentin stromal domains). One domain is dominated by ribosomal/mitochondrial/MALAT1 genes with no coherent biology and is flagged as likely **technical** (low-quality spots) — consistent with it being poorly predicted by both QC and morphology.
- **Invasive front (tumor boundary vs core).** At the tumor-stroma interface, **E-cadherin (CDH1) is significantly down-regulated, robustly across thresholds (0.25 and 0.3)**, with mesenchymal markers (VIM, MMP9) significant only at the more inclusive 0.25 threshold and **no significant activation of canonical EMT transcription factors (ZEB1/2, SNAI1/2, TWIST1)**. This is consistent with a **partial / hybrid E-M state** (associated with unjamming) rather than a complete EMT, and with signal dilution at 55 um resolution. The boundary is otherwise dominated by an antigen-presentation/immune signature (HLA-II, CD74), reflecting the tumor-microenvironment interface.
- **Morphology -> molecular state.** A random forest predicting domain from H&E image features alone achieves **balanced accuracy 0.53 vs a 0.09 chance level (5.8x better than chance)**, quantifying the structure-state coupling. Domains with distinctive histology (tumor, stroma, normal epithelium) are well predicted; mixed and the flagged technical domain are not.

---

## Methodological notes
- **Continuous proportions over argmax.** A dominant-cell-type (argmax) assignment labelled only 6 spots as T-cells and would have hidden the exclusion signal; continuous proportions recovered it. Argmax discards minority populations present but rarely the plurality of a mixed spot.
- **Tumor threshold.** The Cancer-Epithelial proportion is **unimodal** (no natural valley), reflecting the mixed nature of 55 um spots, so the tumor-spot threshold (0.3) is a declared, distribution-guided choice — not a 0.5 default — and the invasive-front result was checked for robustness across thresholds.
- **Statistical significance.** Differential-expression calls use adjusted p-values, not fold-change alone (several EMT genes had large fold-changes but non-significant p-values).
- **Reproducibility.** Notebook 2 pins a working configuration for `scvi-tools` 1.4.3 (`CondSCVI(prior="mog")`) so that `DestVI.from_rna_model` reads the learned prior correctly; the deconvolution workflow was validated on synthetic data before running on real data.

---

## Limitations / what I'd do next
- Single benchmark slide: findings are **hypotheses**, not conclusions. Validation needs replication across sections/patients, domain **stability** across seeds, and checking domains do not track technical covariates (e.g. the flagged technical domain).
- Deconvolution is bounded by the reference: absent cell types are silently reassigned; proportions need orthogonal validation (markers, IHC).
- 55 um spots are not single-cell — the partial-EMT/unjamming signal is at the limit of what Visium resolves; **Xenium/MERFISH would be the next platform** to test the unjamming hypothesis directly.
- The morphology link is correlational; Haralick features are generic descriptors.

---

## Repository structure

```
.
├── 01_data_qc_graph_gnn_domains.ipynb    # data -> QC -> spatial graph -> GNN -> domains
├── 02_deconvolution_destvi.ipynb         # CondSCVI + DestVI -> cell-type proportions
├── 03_niche_analysis_morphology.ipynb    # neighborhood enrichment, co-occurrence, H&E features
├── 04_genesxdomain.ipynb                # markers, invasive front (EMT), morphology classifier
├── requirements.txt
└── README.md
```

Intermediate `.h5ad` files are passed between notebooks.

---

## How to run

Notebook 1 runs on CPU; **Notebook 2 needs a GPU** (Google Colab with a T4 is sufficient). Each notebook installs its dependencies in the first cell and downloads its own data; run them in order, as notebook *n* reads the output of notebook *n-1*.

**Core stack:** `scanpy`, `squidpy`, `anndata`, `torch`, `torch-geometric`, `scvi-tools`, `scikit-learn`, `leidenalg`.

---

## Author

Giulia Ammirati — PhD candidate, Biosystems Science and Engineering, ETH Zurich (D-BSSE, Basel). Background in cell/tissue mechanics (AFM, rheology) and independent computational biology.
