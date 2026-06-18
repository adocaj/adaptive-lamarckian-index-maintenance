# adaptive-lamarckian-index-maintenance

Reproducibility companion to the ICDE submission **Adaptive Lamarckian Index Maintenance for Exact Metric Search over Growing Vector Data**.

This repository contains the experiment notebooks, dataset-preparation notebooks, workload sequences, and horizon-scaling plots used for the submitted paper. The experiment notebooks include the Lamarck policy, the implementations of the methods it is compared against, and the saved notebook outputs used to verify the reported speedup, ablation, and horizon-scaling results.

Lamarck is a runtime-calibrated index-maintenance policy for exact metric search over growing vector data. It predicts rebuild, insert, and query costs; updates cost-model parameters from measured runtimes; maintains a small population of candidate parameter vectors; applies Lamarckian writeback to elite candidates; and adjusts the rebuild margin when post-reconstruction insertions increase predicted query cost. The experiments instantiate the policy with dynamic VP-trees for direct comparison with VPWV and the static Log-Threshold rule.

## Maintenance decision

After each insertion batch, the policy selects one of two maintenance actions. It may *rebuild*, reconstructing the metric index over the current dataset, or *insert*, absorbing the new vectors into the existing index. Insertion defers the cost of reconstruction but allows effective node occupancy to rise, which can raise the cost of later queries, whereas reconstruction restores a well-organized index at an immediate cost that is wasted when the current index remains effective. The policy resolves this tradeoff online, calibrating its cost model from the rebuild, insert, and query times it observes and adjusting the reconstruction margin as post-reconstruction insertions accumulate.

## Datasets

The evaluation spans four large metric datasets chosen to cover heterogeneous dimensional regimes.

| Dataset  | Source                                      | Dimension `d` | Notes                                                |
|----------|---------------------------------------------|---------------|-----------------------------------------------------|
| Higgs    | UCI Machine Learning Repository             | 7             | High-energy physics measurements                    |
| Hepmass  | UCI Machine Learning Repository             | 6             | High-energy physics measurements                    |
| Sensory  | UCI Gas Sensor Array under Dynamic Mixtures | 19            | CO + methane readings concatenated; > 8.4M vectors  |
| GloVe    | Stanford NLP                                | 25            | Pretrained word embeddings                          |

## Workloads

Each experiment interleaves insertion and query phases as an ordered sequence of pairs `(b_k, q_k)` indexed by decision rounds `k`, where `b_k` is the size of the insertion batch applied at round `k` and `q_k` is the number of exact nearest-neighbor queries evaluated after the maintenance action. Following the growing-data protocol, the insertion sizes are drawn from `EB = 0`, `SB = 10,000`, `MB = 100,000`, and `LB = 1,000,000`, and the query loads from `EQ = 0`, `SQ = 20`, `MQ = 100`, and `LQ = 500`; either component of a pair may be empty, so the sequences exercise insertion-only, query-only, and mixed rounds.

The paper reports sequence lengths `L = 10`, `L = 30`, and `L = 100`. Each length is evaluated under three independent permutations of the tuple multiset, with five independent experiments per configuration on a fixed CPU allocation and on the same sampled insertions and queries; the coefficient of variation of end-to-end runtime remains below 10%. The repository additionally provides `L = 20` notebooks used for supplemental horizon-scaling and hardware-verification checks.

## Methods compared

Four methods are evaluated. Exhaustive is a naive exact nearest-neighbor search that scans all stored vectors; it serves as the reference against which every speedup in the tables is computed. VPWV is the incremental VP index. Log-Threshold is the static reconstruction-threshold policy from prior growing-data benchmarking. Lamarck is the adaptive index-maintenance policy proposed in the paper. All runtimes are end-to-end wall-clock measurements over the full insertion–query sequence, and speedups are reported relative to Exhaustive.

## Main ICDE results

Across all four datasets, Lamarck improves consistently over both VPWV and the static Log-Threshold rule. At sequence length `L = 30`, the average speedups relative to Exhaustive are:

| Dataset | Average speedup at `L = 30` |
|---------|-----------------------------|
| Higgs   | 108.5x                      |
| Hepmass | 114.9x                      |
| Sensory | 107.8x                      |
| GloVe   | 74.2x                       |

The `L = 100` ablation study confirms that population adaptation, Lamarckian writeback, and the degradation-aware margin each contribute to the final runtime and to repeatability: removing any one of them degrades performance on every dataset.

## Repository contents

The top level of the repository is organized as follows:

```text
adaptive-lamarckian-index-maintenance/
├── README.md
├── lamarckian_colab_notebooks.zip   # all experiment notebooks, organized by dataset
├── dataset_preparation.zip          # dataset preprocessing notebooks
└── plots/                           # horizon-scaling delta-comparison figures
    ├── amd_delta_comparison_sequences_L20_to_L100.pdf
    └── xeon_delta_comparison_sequences_L20_to_L100.pdf
```

The experiment notebooks and the dataset-preparation notebooks are provided as zip archives so that the repository landing page remains compact; both should be unpacked locally before use.

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
glove_evoAlg_20/
├── evo_alg_seq_20_p1_glove.ipynb
├── evo_alg_seq_20_p2_glove.ipynb
├── evo_alg_seq_20_p3_glove.ipynb
├── EvoAlg_Results_Glove_20.ipynb
└── intelXeon/
```

The `p1`, `p2`, and `p3` notebooks correspond to the three workload-sequence permutations. Each permutation notebook records five independent runs for each compared method and reports the mean runtime for that permutation. The corresponding `EvoAlg_Results_*` notebook takes the three permutation-level mean runtimes, computes the combined mean for each method, and reports the final speedups relative to the combined Exhaustive mean. The `intelXeon/` subfolder is included only for sequence lengths `L = 20` and `L = 100`; sequence lengths `L = 10` and `L = 30` do not contain Intel Xeon subfolders.

## Dataset preparation

The notebooks in `dataset_preparation.zip` document the preprocessing used to construct the vector inputs for Higgs, Hepmass, Sensory, and GloVe. The large raw datasets are not stored in the repository; the preparation notebooks specify how the processed benchmark arrays were derived from the original public sources.

## Supplemental hardware-verification runs

The submitted paper uses AMD EPYC 7B12 runs from Google Colab Pro as the main experimental environment. The repository additionally includes Intel Xeon verification runs inside the relevant sequence folders of `lamarckian_colab_notebooks.zip`, under:

```text
*_evoAlg_20/intelXeon/
*_evoAlg_100/intelXeon/
```

There is no Intel Xeon folder under `*_evoAlg_10/` or `*_evoAlg_30/`. These runs provide an additional check under different hardware; they are included for verification and robustness and are not intended to replace the controlled AMD experimental protocol used in the submitted ICDE paper.

The `plots/` folder collects the horizon-scaling figures that isolate the advantage of Lamarck over the strongest static rule, Log-Threshold, through the absolute time saving `Δ(L) = T_Log-Threshold(L) − T_Lamarck(L)`. Both the AMD and Intel Xeon figures report the same horizon increase, from `L = 20` to `L = 100`, so the supplemental comparison is matched by sequence length and differs only in hardware. In both hardware settings, the same qualitative pattern holds: the time saving of Lamarck over Log-Threshold increases with the longer decision horizon.

## Reproducibility notes

Wall-clock timings depend on hardware, runtime scheduling, memory pressure, and Colab allocation, so absolute seconds may vary across reruns. The intended reproducibility target is therefore the comparative pattern under matched conditions: each method is evaluated on the same dataset sample, the same workload sequence, the same sampled insertions and queries, and the same runtime environment. Because every notebook retains its executed cell outputs, the measurements from the original runs can be inspected directly before any parameters are changed and the experiments rerun.

## Running the notebooks

To reproduce a configuration, unpack `lamarckian_colab_notebooks.zip`, open the relevant Colab notebooks, and execute them in order: prepare or load the dataset, select the dataset folder and then the sequence-length folder, run the three permutation notebooks, and finally run the corresponding result-summary notebook.

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
