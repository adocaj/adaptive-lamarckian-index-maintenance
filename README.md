# adaptive-lamarckian-index-maintenance

Code, Colab notebooks, workload sequences, dataset-preparation notebooks, and experimental logs for Adaptive Lamarckian Index Maintenance, an exact metric-search policy for growing vector data.

This repository accompanies the ICDE submission:

**Adaptive Lamarckian Index Maintenance for Exact Metric Search over Growing Vector Data**

The paper studies exact nearest-neighbor search over growing vector data, where insertion batches and query batches arrive over a sequence of decision rounds. After each insertion batch, the system decides whether to insert the new vectors into the current metric index or reconstruct the index over the current dataset. The proposed method uses an analytic rebuild-versus-insert cost model, online runtime calibration, population-based exploration of cost-model parameters, Lamarckian writeback into elite population members, and a degradation-aware reconstruction margin.

## Repository organization

The main Colab notebooks are organized by dataset. The top-level notebook folder has the following form:

```text
lamarckian_colab_notebooks/
├── Glove/
├── Hepmass/
├── Higgs/
└── Sensory/
```

Each dataset folder contains sequence-length-specific experiment folders. For example, the `Glove/` folder is organized as:

```text
Glove/
├── glove_evoAlg_10/
├── glove_evoAlg_20/
├── glove_evoAlg_30/
├── glove_evoAlg_100/
└── VP_nsDyn_Eps_Opt_Glove.ipynb
```

The same organization is used for the other datasets, with the dataset name changed accordingly.

Inside each sequence folder, the notebooks store the individual permutation runs and the corresponding result-summary notebook. For example:

```text
glove_evoAlg_10/
├── evo_alg_seq_10_p1_glove.ipynb
├── evo_alg_seq_10_p2_glove.ipynb
├── evo_alg_seq_10_p3_glove.ipynb
├── EvoAlg_Results_Glove_10.ipynb
└── intelXeon/
```

The `p1`, `p2`, and `p3` notebooks correspond to the three workload-sequence permutations. The `EvoAlg_Results_*` notebook summarizes the runs for that dataset and sequence length.

The `intelXeon/` subfolder is included only for sequence lengths `L = 10`, `L = 20`, and `L = 100`. Sequence length `L = 30` does not contain an Intel Xeon subfolder. The Intel Xeon folders provide supplemental hardware-verification runs, while the main ICDE paper experiments use the AMD Colab Pro environment described in the paper.

## Method summary

Adaptive Lamarckian Index Maintenance treats exact nearest-neighbor search over growing vector data as a physical index-maintenance problem. The policy chooses between two actions after each insertion batch:

```text
rebuild: reconstruct the metric index over the current dataset
insert:  absorb the new batch into the current index
```

The method preserves exact search. It does not change the metric, learn a new vector representation, or introduce approximate nearest-neighbor search. The adaptive component only changes the maintenance decision and the runtime cost-model parameters used to make that decision.

At a high level, the policy combines:

```text
1. an analytic predictor for rebuild, insert, and query costs;
2. online updates of live cost-model parameters from observed runtimes;
3. a population of nearby candidate parameter vectors;
4. Lamarckian writeback from the updated live parameters into elite candidates;
5. a degradation-aware margin that accounts for query-cost growth after post-rebuild insertions.
```

## Datasets

The experiments use four metric vector datasets:

```text
Higgs      high-energy physics measurements
Hepmass    high-energy physics measurements
Sensory    gas-sensor readings
GloVe      word-embedding vectors
```

The submitted paper evaluates heterogeneous dimensional regimes: Higgs with 7 dimensions, Hepmass with 6 dimensions, Sensory with 19 dimensions, and GloVe with 25 dimensions.

## Workloads

Experiments use interleaved insertion and query batches. A workload sequence consists of ordered pairs:

```text
(b_k, q_k)
```

where `b_k` is the insertion-batch size at decision round `k`, and `q_k` is the number of exact nearest-neighbor queries evaluated after the maintenance action.

The paper reports the main sequence lengths:

```text
L = 10
L = 30
L = 100
```

The repository also includes `L = 20` notebooks used for supplemental horizon-scaling and hardware-verification checks.

## Baselines

The main comparisons are:

```text
Exhaustive       naive exact nearest-neighbor search
VPWV             incremental VP index baseline
Log-Threshold    static reconstruction-threshold policy
Lamarck          adaptive Lamarckian index-maintenance policy
```

Runtime results are reported as end-to-end wall-clock measurements over the full insertion-query sequence.

## Main ICDE results

The submitted paper reports that Lamarck consistently improves over VPWV and the static Log-Threshold rule on Higgs, Hepmass, Sensory, and GloVe. At sequence length `L = 30`, the reported average speedups relative to Exhaustive are:

```text
Higgs      108.5x
Hepmass    114.9x
Sensory    107.8x
GloVe       74.2x
```

The paper also includes `L = 100` ablations showing that population adaptation, Lamarckian writeback, and the degradation-aware margin each contribute to the final runtime improvement and repeatability.

## Supplemental hardware-verification runs

The submitted paper uses AMD EPYC 7B12 runs from Google Colab Pro as the main experimental environment. The repository additionally includes Intel Xeon verification runs inside the relevant sequence folders.

For each dataset, Intel Xeon supplemental folders are included only under:

```text
*_evoAlg_10/
*_evoAlg_20/
*_evoAlg_100/
```

There is no Intel Xeon folder under `*_evoAlg_30/`.

The AMD supplemental comparison checks a more modest horizon increase from `L = 20` to `L = 100`. The same qualitative pattern persists: the time saving of Lamarck over Log-Threshold increases with the longer decision horizon.

The Intel Xeon folders provide an additional check under different hardware. These runs are included for verification and robustness; they are not intended to replace the controlled AMD experimental protocol used in the submitted ICDE paper.

## Reproducibility notes

Wall-clock timings depend on hardware, runtime scheduling, memory pressure, and Colab allocation. Therefore, exact seconds may vary across reruns. The main reproducibility target is the comparative pattern under matched conditions: each method should be evaluated on the same dataset sample, the same workload sequence, the same insertion batches, the same query batches, and the same runtime environment.

For verification, use the saved notebooks and logs before changing parameters.

## Dataset preparation

A separate dataset-preparation folder contains Colab notebooks for preparing the datasets used in the experiments. These notebooks document the preprocessing steps used to form the vector inputs for Higgs, Hepmass, Sensory, and GloVe.

Large raw datasets may not be stored directly in this repository. When necessary, the preparation notebooks indicate how the processed benchmark arrays were generated from the original public datasets.

## Running the notebooks

The easiest way to reproduce the experiments is to open the corresponding Colab notebooks and run them in order.

A typical workflow is:

```text
1. Prepare or load the dataset.
2. Select the dataset folder.
3. Select the sequence length folder.
4. Run the three permutation notebooks.
5. Run the corresponding result-summary notebook.
6. Compare Lamarck with Exhaustive, VPWV, and Log-Threshold under the same workload sequence.
```

For example, the GloVe `L = 10` AMD runs are stored in:

```text
lamarckian_colab_notebooks/Glove/glove_evoAlg_10/
```

The corresponding Intel Xeon supplemental runs, when present, are stored inside:

```text
lamarckian_colab_notebooks/Glove/glove_evoAlg_10/intelXeon/
```

## Citation

If this repository is useful, please cite the corresponding ICDE submission once citation information is available.

```text
Andris Docaj and Yu Zhuang.
Adaptive Lamarckian Index Maintenance for Exact Metric Search over Growing Vector Data.
Submitted to ICDE.
```

## License

License information will be added here.
