# Cluster Count Regression Results and PI Email Priorities

Saved 2026-09-24 for a future email to the PI. This is a briefing note; no email has been drafted or sent in this step.

## Project and artifacts

- Project key is `cluster_analysis`, captured from the session's supplied working-directory basename before accessing the notes repository. That original directory has since been renamed; do not infer the project key from the temporary `/` shell workaround.
- Completed regional analysis: `/Users/flemingtonlab/ethan/de_novo/analysis/cluster_count_analysis/`.
- Main artifacts: `index.html`, `SUMMARY.md`, `ANALYSIS_SPECIFICATION.md`, `summary/primary_model_metrics.tsv`, `summary/all_model_metrics.tsv`, `summary/feature_direction_stability.tsv`, and `summary/validation.json`.
- Original promoter-level analysis: `/Users/flemingtonlab/ethan/de_novo/analysis/promoter_to_cluster_analysis/`.
- Spatial maps and reference audit: `/Users/flemingtonlab/ethan/de_novo/analysis/cluster_spatial_analysis/`.

## Scientific question and fixed scope

The PI asked why clusters form where they do, independently of comparing clustered versus individual isolated promoters. The regional formulation is: which sequence and chromatin features predict the number of clusters in fixed genomic neighborhoods?

Each genomic window is one observation. Primary analysis: 2,627 eligible full-width 1-Mb windows on chromosomes 1–22 containing 15,664 retained clusters. Keep zero-count windows; assign each existing cluster once by its selected-TSS-span midpoint. Clusters consist of at least three promoters linked by consecutive selected-TSS gaps of at most 100 bp, combining strands. Include all activity levels, not the nested top-20/10/5/1% activity categories.

Retain windows with at least 80% screened reference sequence (A/C/G/T outside the existing hg38 ENCODE blacklist), and clusters whose entire selected-TSS span passes that screen. Use screened Mb as an exposure offset. This is not read-specific mappability or experimentally established callability; original alignment/QC and exact sample provenance remain incompletely verified.

Shared predictors: GC fraction; seven separate exact motif densities (TATTAAA, TATTTAA, TGAGTCA, TGAGCCA, TGAGCGA, TGTGCAA, TGTGCGA); annotated protein-coding gene-start density (local Ensembl 113, one gene once); chromosome-end distance. Each condition model adds either mean MC or mean MZ normalized ATAC insertion density. These are separate predictors of the same cluster catalogue, not condition-specific cluster outcomes. Motifs are counted on both strands, wholly within screened sequence and within the bin. No feature is constructed around known clusters.

Fit Poisson and NB2 models for baseline, baseline+MC, and baseline+MZ. Five validation folds hold out whole chromosomes. Primary grid is 1 Mb/origin/absolute end distance. Repeat all six specifications at 100 kb, 500 kb and 1 Mb with original and half-shifted boundaries: 36 configurations. Only on the primary grid, repeat six specifications with relative distance 2d/chromosome_length: six additional configurations, **42 total, not 72**. There are 210 cross-validation fits and 12 all-data coefficient fits, 222 saved fits total. These are model evaluations, not 222 independent hypothesis tests. All saved fits, likelihoods and held-out predictions passed independent numerical checks; all 68,742 eligible window records across the six schemes passed independent cluster recounting.

No additional analysis is authorized merely by discussing possible next steps. The user requires the high-level scope to be agreed in advance and no unannounced implementation changes.

## Main results to retain for the PI email

### Performance and which model to emphasize

For predicting the expected regional cluster count, recommend **MC + Poisson, primary 1-Mb grid**. Negative binomial was prespecified as primary and remains transparently reported; do not retroactively conceal it or claim that Poisson wins every criterion.

| Primary specification | Mean count deviance (lower better) | Mean absolute error (clusters/window) | Spearman rho | Mean log predictive probability (higher better) |
|---|---:|---:|---:|---:|
| Shared-feature baseline + Poisson | 3.0906 | 3.1286 | 0.8391 | -2.7089 |
| MC + Poisson | 2.9026 | 3.0734 | 0.8499 | -2.6148 |
| MZ + Poisson | 2.9932 | 3.0815 | 0.8442 | -2.6601 |
| Shared-feature baseline + NB2 | 3.3457 | 3.5086 | 0.8400 | -2.1464 |
| MC + NB2 | 3.6858 | 3.8959 | 0.8576 | -2.1103 |
| MZ + NB2 | 3.3469 | 3.5632 | 0.8475 | -2.1331 |

- MC Poisson reduces held-out count deviance by 6.1% beyond the shared-feature baseline; MZ reduces it by 3.2%. These measure the incremental value of accessibility, not the entire model's improvement over an uninformative predictor. The baseline already contains sequence, gene density and end proximity.
- Spearman 0.850 indicates strong regional ranking. It is not 85% accuracy or 85% variance explained. It cannot be directly compared with the earlier random forest's classification accuracy/AUC.
- Mean absolute error is 3.07 clusters/window; mean observed count is about 5.96 and counts are highly uneven. This supports identifying relatively cluster-rich neighborhoods better than predicting precise counts in every window.
- MC NB2 has the best primary log probability, but overpredicts the total: 19,240 predicted versus 15,664 observed (22.8% high). MC Poisson predicts about 15,674 overall, but aggregate agreement can conceal window/chromosome errors.
- Nominal 95% prediction intervals cover 87.1% for MC Poisson and 97.1% for MC NB2. Poisson is overconfident. Observed zero-window fraction is 33.2%; MC Poisson predicts 17.2% and MC NB2 27.1%. NB2 is not an unqualified winner.
- Across the six absolute-distance schemes, MC Poisson's count-deviance improvement is 6.1–8.1%, versus MZ's 0.7–4.3%. MC performs better than MZ on this metric across schemes, but not necessarily every chromosome/fold. Both accessibility NB models improve probability scores over their matched baselines; MC NB count deviance nevertheless worsens by 5.0–10.3%.

### Individual sequence/context features: important for the email

The user specifically wants individual sequence features named, rather than only saying "sequence context was informative."

- **GC content** has the largest standardized coefficient in both primary condition models.
- **TGAGCCA density** and **TATTTAA density** have consistent positive adjusted associations with regional cluster counts.
- Accessibility is also positively associated with count. GC, TGAGCCA, TATTTAA, and the condition accessibility coefficient stay positive across all 30 training fits covering the six absolute-distance schemes in each NB condition model.
- **TATTAAA** is comparatively weak and changes sign across NB training folds. Do not generalize the regional TATTTAA finding to all TATT motifs.
- Gene density is positive in the primary condition models but less stable across scales after MC adjustment.

Representative expected-count multipliers per one SD of each transformed feature (other predictors held fixed):

| Feature | MC Poisson (recommended for mean prediction) | MC NB2 (prespecified primary family) | MZ NB2 |
|---|---:|---:|---:|
| GC fraction | 2.07 | 2.25 | 1.95 |
| TGAGCCA density | 1.52 | 1.52 | 1.60 |
| TATTTAA density | 1.33 | 1.43 | 1.34 |
| Condition accessibility | 1.28 | 1.50 | 1.40 |
| Gene-start density | 1.12 | 1.22 | 1.34 |

GC is unlogged; densities and accessibility use log1p before standardization. Do not mix effect sizes from different families without labeling them. Standardized coefficients are conditional associations, not demonstrated causal effects or unique feature importance; predictors are correlated. Formal coefficient p-values were not used. Fold ranges are not confidence intervals.

### The apparent TATT contrast across scales

The user finds this especially interesting and wants it retained for the PI email:

- Earlier promoter-level RF: TATTAAA/TATTTAA centered around −40 to −20 bp favor an isolated promoter, while higher MZ core accessibility favors clustered promoters.
- Current regional regression: greater **TATTTAA density anywhere within a large genomic window** is associated with a higher **number of clusters in that window**.
- These are different observations, outcomes and motif definitions. A region can contain more of both isolated and clustered promoters. The regional association does not mean that a particular promoter containing a TATT motif is more likely to be clustered, and it does not establish protein occupancy or a mechanism.
- Earlier follow-up did not establish that BcRF1-mediated Tn5 protection explains the core-accessibility association. The core pattern also persisted without the exact TATT motifs; ATAC dips alone cannot identify a bound protein.

### Chromosome ends

Adjusted end-associated count enrichment persists qualitatively under absolute and relative distance. In primary NB models the near-end contrast relative to median distance is approximately 1.37×/1.32× for MC/MZ with absolute distance, and 1.22×/1.17× with relative distance. The reference points differ, so those magnitudes are not a like-for-like mechanistic comparison. Switching distance definitions changes primary count deviance by less than 2% in every specification. No model omitting end distance was fitted, so its separate incremental predictive contribution has not been measured. Reference-end distance is not proof of a telomere mechanism.

## User's interpretation and next discussion

The user views the Spearman score as encouraging, recognizes that sequence context carries much information, is interested in the TATTTAA cross-scale contrast, and wants individual sequence features highlighted in the future email. They prioritize finding broadly cluster-rich neighborhoods over distinctions such as 29 versus 30 clusters in a hypothetical count range of 1–100.

Mentoring qualification: that precision preference is reasonable, but a wide marginal count range does not by itself explain or excuse prediction error. The model predicts expected counts; missing features, model misspecification, detection effects and residual variation can all affect calibration. Systematic overprediction and insufficient zero-count probability remain substantive limitations.

The PI encouraged other approaches. The user asks whether these results are strong enough as they stand and what additional approaches would advance "why clusters occur where they do." Current assessment: sufficiently developed for a substantive PI update and an exploratory predictive/association result; not a demonstrated causal mechanism. Future approaches are for discussion only. Particularly useful questions include whether these features predict general promoter abundance versus excess clustering after accounting for regional initiation density; whether motif information survives composition/context controls; and whether motif placement within accessible subregions explains differences lost by window averages. No follow-up implementation has begun.
