# adaptive-lamarckian-index-maintenance

Code, Colab notebooks, workload sequences, dataset-preparation notebooks, and experimental logs for **Adaptive Lamarckian Index Maintenance for Exact Metric Search over Growing Vector Data**.

This repository is the reproducibility companion to the ICDE submission of the same name. The paper studies exact nearest-neighbor search over growing vector data, where insertion batches and query batches arrive over a sequence of decision rounds. After each insertion batch, the system must decide whether to insert the new vectors into the current metric index or reconstruct the index over the current dataset. The proposed policy uses an analytic cost model for rebuild and insert decisions, updates its cost-model parameters from observed runtimes, transfers updated parameters into elite population members through Lamarckian writeback, and uses a degradation-aware margin to account for query-cost growth from post-reconstruction insertions. The method remains exact: it does not change the metric, learn a new vector representation, or introduce approximate search. It changes only the index-maintenance decision and the cost-model parameters used to make that decision.

## Method summary

After each insertion batch the policy selects one of two actions:

```text
rebuild: reconstruct the metric index over the current dataset
insert:  absorb the new batch into the current index
```

The policy is instantiated on a dynamic VP-tree and combines an analytic predictor for rebuild, insert, and query costs; online calibration of live cost-model parameters from observed runtimes; a small population of nearby candidate parameter vectors; projected Lamarckian writeback that copies the updated live parameters into elite candidates; and a degradation-aware reconstruction margin that prices the query-cost growth caused by insertions accumulated since the last rebuild. Together these track the observed runtime regime over a growing-data workload while preserving exact search.

## Datasets

The experiments use four large, diverse metric datasets spanning heterogeneous dimensional regimes.

| Dataset  | Source                                      | Dimension `d` | Notes                                                |
|----------|---------------------------------------------|---------------|-----------------------------------------------------|
| Higgs    | UCI Machine Learning Repository             | 7             | High-energy physics measurements                    |
| Hepmass  | UCI Machine Learning Repository             | 6             | High-energy physics measurements                    |
| Sensory  | UCI Gas Sensor Array under Dynamic Mixtures | 19            | CO + methane readings concatenated; > 8.4M vectors  |
| GloVe    | Stanford NLP                                | 25            | Pretrained word embeddings; highest-dimensional set |

## Workloads

Experiments interleave batched insertions and query phases as ordered pairs `(b_k, q_k)` indexed by decision rounds `k`, where `b_k` is the insertion-batch size and `q_k` is the number of exact nearest-neighbor queries evaluated after the maintenance action. Following the growing-dataset protocol, batch sizes are drawn from `EB = 0`, `SB = 10,000`, `MB = 100,000`, `LB = 1,000,000`, and query loads from `EQ = 0`, `SQ = 20`, `MQ = 100`, `LQ = 500`, with tuples allowed to leave either component empty to stress insertion-only, query-only, and mixed rounds.

The paper reports sequence lengths `L = 10`, `L = 30`, and `L = 100`. Each length is evaluated under three independent permutations of the tuple multiset, with five independent experiments per configuration on a fixed CPU allocation and identical sampled batches and queries; the coefficient of variation of end-to-end runtime stays below 10%. The repository additionally includes `L = 20` notebooks used for supplemental horizon-scaling and hardware-verification checks.

## Baselines

```text
Exhaustive       naive exact nearest-neighbor search
VPWV             incremental VP index baseline
Log-Threshold    static reconstruction-threshold policy
Lamarck          adaptive Lamarckian index-maintenance policy
```

Runtimes are reported as end-to-end wall-clock measurements over the full insertion–query sequence, and speedups are taken relative to Exhaustive.

## Main ICDE results

Lamarck consistently improves over VPWV and the static Log-Threshold rule on all four datasets. At sequence length `L = 30`, the reported average speedups relative to Exhaustive are:

| Dataset | Average speedup at `L = 30` |
|---------|-----------------------------|
| Higgs   | 108.5x                      |
| Hepmass | 114.9x                      |
| Sensory | 107.8x                      |
| GloVe   | 74.2x                       |

The `L = 100` ablation study shows that population adaptation, Lamarckian writeback, and the degradation-aware margin each contribute to the final runtime improvement and repeatability: removing any one of them reduces performance on every dataset.

## Repository contents

The top level of the repository is organized as follows:

```text
adaptive-lamarckian-index-maintenance/
├── LICENSE
├── README.md
├── lamarckian_colab_notebooks.zip   # all experiment notebooks, organized by dataset
├── dataset_preparation.zip          # dataset preprocessing notebooks
└── plots/                           # horizon-scaling delta-comparison figures
    ├── amd_delta_comparison_sequences_L20_to_L100.pdf
    ├── xeon_delta_comparison_sequences_L10_to_L100.pdf
    └── xeon_delta_comparison_sequences_L20_to_L100.pdf
```

The experiment notebooks and the dataset-preparation notebooks are distributed as zip archives to keep the landing page compact; unpack them locally before running.

### Notebook organization

`lamarckian_colab_notebooks.zip` unpacks to a top-level folder organized by dataset:

```text
lamarckian_colab_notebooks/
├── Glove/
├── Hepmass/
├── Higgs/
└── Sensory/
```

Each dataset folder contains sequence-length-specific experiment folders together with a per-dataset optimization notebook. For example, `Glove/` is organized as:

```text
Glove/
├── glove_evoAlg_10/
├── glove_evoAlg_20/
├── glove_evoAlg_30/
├── glove_evoAlg_100/
└── VP_nsDyn_Eps_Opt_Glove.ipynb
```

The same organization is used for the other datasets, with the dataset name changed accordingly. Inside each sequence folder, the notebooks store the individual permutation runs and the corresponding result-summary notebook:

```text
glove_evoAlg_10/
├── evo_alg_seq_10_p1_glove.ipynb
├── evo_alg_seq_10_p2_glove.ipynb
├── evo_alg_seq_10_p3_glove.ipynb
├── EvoAlg_Results_Glove_10.ipynb
└── intelXeon/
```

The `p1`, `p2`, and `p3` notebooks correspond to the three workload-sequence permutations, and the `EvoAlg_Results_*` notebook summarizes the runs for that dataset and sequence length. The `intelXeon/` subfolder is included only for sequence lengths `L = 10`, `L = 20`, and `L = 100`; sequence length `L = 30` does not contain an Intel Xeon subfolder.

## Datasets and preparation

The dataset-preparation notebooks in `dataset_preparation.zip` document the preprocessing used to form the vector inputs for Higgs, Hepmass, Sensory, and GloVe. Large raw datasets are not stored directly in the repository; the preparation notebooks indicate how the processed benchmark arrays were generated from the original public sources.

## Supplemental hardware-verification runs

The submitted paper uses AMD EPYC 7B12 runs from Google Colab Pro as the main experimental environment. The repository additionally includes Intel Xeon verification runs inside the relevant sequence folders of `lamarckian_colab_notebooks.zip`, under:

```text
*_evoAlg_10/intelXeon/
*_evoAlg_20/intelXeon/
*_evoAlg_100/intelXeon/
```

There is no Intel Xeon folder under `*_evoAlg_30/`. These runs provide an additional check under different hardware; they are included for verification and robustness and are not intended to replace the controlled AMD experimental protocol used in the submitted ICDE paper.

The `plots/` folder collects the horizon-scaling figures that isolate the advantage of Lamarck over the strongest static rule, Log-Threshold, via the absolute time saving `Δ(L) = T_Log-Threshold(L) − T_Lamarck(L)`. The AMD figure reports a more modest horizon increase from `L = 20` to `L = 100`, and the Intel Xeon figures reproduce the comparison on different hardware from `L = 10` to `L = 100` and from `L = 20` to `L = 100`. In every case the same qualitative pattern persists: the time saving of Lamarck over Log-Threshold increases with the longer decision horizon.

## Reproducibility notes

Wall-clock timings depend on hardware, runtime scheduling, memory pressure, and Colab allocation, so exact seconds may vary across reruns. The main reproducibility target is the comparative pattern under matched conditions: each method is evaluated on the same dataset sample, the same workload sequence, the same insertion batches, the same query batches, and the same runtime environment. For verification, use the saved notebooks and logs before changing parameters.

## Running the notebooks

The easiest way to reproduce the experiments is to unpack `lamarckian_colab_notebooks.zip`, open the corresponding Colab notebooks, and run them in order:

```text
1. Prepare or load the dataset.
2. Select the dataset folder.
3. Select the sequence length folder.
4. Run the three permutation notebooks.
5. Run the corresponding result-summary notebook.
6. Compare Lamarck with Exhaustive, VPWV, and Log-Threshold under the same workload sequence.
```

For example, the GloVe `L = 10` AMD runs are stored in `lamarckian_colab_notebooks/Glove/glove_evoAlg_10/`, and the corresponding Intel Xeon supplemental runs, when present, are stored inside `lamarckian_colab_notebooks/Glove/glove_evoAlg_10/intelXeon/`.

## Citation

```bibtex
@misc{docaj2026adaptive,
  title  = {Adaptive Lamarckian Index Maintenance for Exact Metric Search over Growing Vector Data},
  author = {Docaj, Andris and Zhuang, Yu},
  year   = {2026},
  note   = {Submitted to ICDE 2027}
}
```

## Contact

Andris Docaj, Texas Tech University — Andris.Docaj@ttu.edu

## License

License information will be added here.
