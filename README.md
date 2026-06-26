## Results

**Notebook 1 — spatial domains.** A from-scratch graph autoencoder (PyTorch Geometric) recovers 11 spatial domains (`resolution = 0.5`) that are *spatially compact and continuous* on the tissue and *well separated* in a UMAP of the embeddings. The two views agree — domains distinct in embedding space also form contiguous tissue regions — confirming the learned representation captures real structure, not noise.

**Notebook 2 — cell-type deconvolution.** DestVI (conditioned on a CondSCVI model of the Wu et al. 2021 reference) estimates the composition of every spot across the 9 major cell types of the breast tumor microenvironment. Cell types occupy spatially distinct territories (Cancer Epithelial in compact tumor masses, CAFs in stromal bands, Myeloid in localized foci), and averaging proportions within each Notebook-1 domain assigns each domain a clear identity: several Cancer-Epithelial tumor domains, a Normal-Epithelial domain (residual healthy tissue), CAF-rich stroma, and a single Plasmablast-dominated domain — a localized plasma-cell aggregate. Whole-slide mean composition is biologically sensible (Cancer Epithelial most abundant at ~28%).

**Notebook 3 — spatial niches and morphology.** Quantitative neighborhood analysis reveals a **compartmentalized tissue with immune exclusion**, supported by three convergent lines of evidence:

- **Proportion correlation (Spearman, all spots):** Cancer Epithelial is anti-correlated with T-cells (−0.29) and, most strongly, with Plasmablasts (−0.64) — the plasma-cell aggregate is the structure most segregated from the tumor. Conversely, T-cells co-localize with Myeloid (+0.33) and CAFs (+0.26), i.e. lymphocytes reside in a stromal/immune niche rather than inside tumor masses.
- **Neighborhood enrichment (z-score):** independently confirms strong tumor–plasmablast segregation (the most negative pair in the matrix).
- **Co-occurrence vs distance:** anchored on tumor spots, B-cell and T-cell enrichment is lowest adjacent to the tumor and *rises with distance* (peaking several hundred µm away), the spatial signature of peripheral immune exclusion ("cold tumor").

A methodological note worth highlighting: a naive *dominant–cell-type* (argmax) assignment labelled only 6 spots as T-cells and would have made the exclusion signal invisible. Switching to continuous proportions recovered it — argmax discards minority populations that are present but rarely the plurality of a mixed spot.

**Morphology bridge.** Per-spot features extracted from the H&E image (summary intensity + Haralick texture) differ markedly across the molecular domains — tumor domains show high-contrast texture, the normal-epithelial domain a smoother/more homogeneous signature. Tissue *appearance* therefore carries a signature of molecular identity: structure and state are spatially coupled.

*Figures are in `figures/`.*

### Limitations / what I'd do next
- Benchmark slide: QC thresholds, the number of domains, and cell-type identities are not validated against ground truth here. On new data I would test domain **stability** (across seeds/subsampling), check domains do **not** track technical covariates (`total_counts`, batch, array position), and **replicate** across sections/patients.
- Deconvolution is bounded by the reference: cell types absent from Wu et al. would be silently reassigned; proportions need orthogonal validation (markers, IHC).
- The morphology link is correlational, and Haralick features are generic descriptors; a classifier predicting domain from image features alone would quantify the coupling.
- 55 µm spots are not single-cell; some questions would require Xenium/MERFISH.
