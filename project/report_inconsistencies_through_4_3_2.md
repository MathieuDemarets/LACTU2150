# Report inconsistencies through section 4.3.2

## Scope and verification

Reviewed the complete [HTML report](project_MD.html), including folded code,
tables, footnotes, all 40 static figures, seven interactive plots, and the paged
ANOVA table. Findings below concern sections **1 through 4.3.2, inclusive**.
Later sections are outside the findings scope.

The HTML is the reviewed deliverable. The [R Markdown source](project_MD.Rmd)
and saved modelling data were used to investigate its calculations. Selected
models and five-fold evaluations were reproduced in R; the enriched model's
AIC reproduced exactly at **99,575.66**. This review did not modify or re-render
either report file.

The source received concurrent edits during this review. Those edits were left
untouched; findings describe the reviewed HTML, not a claim that every issue
remains in the latest, independently edited source.

This list distinguishes actual contradictions from methodological limitations:

- **Error:** an incorrect statement, calculation, interpretation, or
  implementation mismatch.
- **Method:** a validation or inference problem; not necessarily proof that the
  selected model is wrong.
- **Evidence:** an interpretation that the report does not establish.
- **Presentation / Incomplete:** a rendering, terminology, or unfinished-content
  issue.

### Highest-priority findings

| IDs | Issue |
|---|---|
| I02 | Claimed stratification collapses to a single stratum. |
| I05, I07, I14, I33 | Reversed exposure inequality, incorrect severity identity, incorrect WoE interpretation, and reversed hypothesis-test interpretation. |
| I35-I38 | Population is not density; geographic identifiers do not consistently match 1997; missing matches are hidden; imputation is refitted on new data. |
| I42, I45 | Orthogonal-polynomial frequency analysis drops every zero-claim policy; the Poisson tree ignores exposure. |
| I49-I53 | Supervised preprocessing precedes CV; the prediction-error derivation is invalid; CV is penalized twice; a chi-squared test is applied to out-of-fold deviance. |
| I47-I48, I56-I57 | Promised interactions are not the fitted interactions; feasible candidate models are excluded from CV and the leaderboard. |
| I61, I66 | GAM baseline instructions conflict, and section 4.3.2 is still a placeholder. |

## 1. Executive summary and portfolio overview

### I01 - Empty executive summary

**Incomplete.** [Section 1](project_MD.html#executive-summary) has a title and
spacing but no summary. It therefore does not summarize the objectives, results,
selected frequency model, or limitations already available by section 4.3.2.

**Correction:** write a short summary, explicitly identifying the unfinished
parts of the analysis.

### I02 - The split and CV are not actually stratified as claimed

**Error / Method.** [Section 2.2.1](project_MD.html#records-rows) says the holdout
split preserves the claim distribution; [section 4.1.2](project_MD.html#performance-evaluation)
similarly describes claim-balanced folds. However, the numeric target is
quantile-binned by the splitting functions. With this zero-heavy dataset,
the quantile boundaries collapse and both procedures produce **one stratum**:

- Initial split: `[0,5]`, containing all **163,231** policies.
- Five-fold CV: `[0,5]`, containing all **130,585** training policies.

The splits are usable random splits, but the stated stratification is not
implemented. Use an explicit factor such as zero/one/multiple claims, or suitably
pooled claim-count classes, and show the resulting distributions.

### I03 - Exposure values do not identify claim occurrence dates

**Error.** [Section 2.2.2.1](project_MD.html#all-variables) infers from 365 distinct
`EXP` values that claims occurred on every possible day of a year. `EXP` records
the duration insured, not the dates on which claims happened.

**Correction:** describe the daily granularity of coverage durations only.

### I04 - The explanation for the different AMOUNT and AVG means is wrong

**Error.** [Section 2.2.2.1](project_MD.html#all-variables) attributes the difference
between **208.8667** and **1698.6839** to a need to adjust both centers for
exposure. These are different estimands:

- `mean(AMOUNT)` is cost per policy, including zero-claim policies.
- `mean(AVG, na.rm = TRUE)` is the unweighted mean of policy-level average costs
  among claimants.
- Portfolio cost per claim is instead `sum(AMOUNT) / sum(NCLAIMS)` =
  **1693.14**.

Exposure annualizes frequency and policy cost; it does not explain away these
different denominators or justify exposure-adjusting severity in the same way.

### I05 - The exposure inequality is reversed

**Error.** [Section 2.2.2.2](project_MD.html#deep-dive-into-exposure-exp) prints

`sum(NCLAIMS) / sum(EXP) <= sum(NCLAIMS) / n`.

For exposures in `(0,1]`, the relation is **greater than or equal to**, not less
than or equal to. The numerical output, **13.862% versus 12.336%**, and the
surrounding prose already contradict the displayed inequality.

### I06 - Counts, annual rates, probabilities, and policy costs are conflated

**Error / Terminology.** [Sections 2.2.2.2-2.2.2.3](project_MD.html#deep-dive-into-exposure-exp)
describe modelling a count as modelling frequency, refer to frequency as the
likelihood of a claim, and call `lambda(x) * mu(x)` an expected total policy cost
without specifying exposure.

**Correction:** distinguish:

- Annual rate `lambda_i`, in claims per policy-year.
- Expected policy count `EXP_i * lambda_i`.
- Probability of at least one Poisson claim `1 - exp(-EXP_i * lambda_i)`.
- Annual expected cost `lambda_i * mu_i`.
- Expected cost for the observed policy duration
  `EXP_i * lambda_i * mu_i`.

The displayed **13.862%** is a rate expressed as a percentage, not a 13.862%
probability of at least one claim.

### I07 - Portfolio severity is incorrectly equated to an individual AVG_i

**Error.** [Section 2.2.2.3](project_MD.html#deep-dive-into-target-variables-nclaims-avg-amount)
states `sum(AMOUNT_i) / sum(NCLAIMS_i) = AVG_i`.

The left side is a single portfolio-level, claim-weighted average. The right side
varies by policy and equals `AMOUNT_i / NCLAIMS_i` for policies with claims.

**Correction:** use a separate symbol for portfolio severity, or write
`sum(NCLAIMS_i * AVG_i) / sum(NCLAIMS_i)` over positive-claim policies.

### I08 - The printed decomposition is not numerically an equality

**Presentation.** [Section 2.2.2.3](project_MD.html#deep-dive-into-target-variables-nclaims-avg-amount)
prints `234.71 = 0.14 * 1693.14`, but the rounded right side is **237.04**.

**Correction:** print more digits for the frequency, approximately **0.138624**,
or use an approximation sign and explain rounding.

### I09 - A high zero proportion is treated as evidence of a zero problem

**Evidence.** [Section 2.2.2.3](project_MD.html#deep-dive-into-target-variables-nclaims-avg-amount)
frames **88.84%** zero-claim policies as an imbalance to tackle. A low-rate
Poisson process naturally generates many zeros. Using the observed exposures,
even the intercept-only Poisson model predicts about **88.45%** zero policies.

**Correction:** distinguish ordinary low-frequency count data from excess zeros.
Check the fitted conditional zero distribution and dispersion before motivating
zero-inflated or hurdle models. The raw zero proportion is not enough.

### I10 - Hurdle models are incorrectly said to necessarily break calibration

**Error.** The same [section](project_MD.html#deep-dive-into-target-variables-nclaims-avg-amount)
groups hurdle models with synthetic oversampling and says these techniques
break observed/modelled frequency alignment. Naive oversampling can change the
population distribution; a properly fitted hurdle model does not inherently
prevent aggregate calibration.

**Correction:** distinguish the absence of an automatic Poisson score-equation
calibration identity from an inability to calibrate a model.

### I11 - Claim types are inferred from a histogram without supporting data

**Evidence.** [Section 2.2.2.3](project_MD.html#deep-dive-into-target-variables-nclaims-avg-amount)
identifies peaks as small dents, larger repairs, and major damages. The plot is
of **policy-level log average costs**, not individual claim types or individual
claim costs. Multiple claims on a policy are already averaged.

**Correction:** describe cost scales and possible interpretations, not established
claim categories. The conversions from log amounts to euros are not themselves
the problem.

## 2. Transformation and binning

### I12 - Ordinal and unordered factors are confused

**Error.** [Section 3.1.1](project_MD.html#categorical-variables) says BM could be
considered *unordered* because higher values mean higher risk but equal jumps are
not known. A known ordering is precisely what makes a variable **ordinal**;
ordered categories do not require equal spacing.

**Correction:** distinguish ordinal factors from treating a score as a numerical
linear predictor. Binning BM can remain a valid separate modelling choice.

### I13 - The GLM linearity assumption is stated on the wrong scale

**Error.** [Section 3.1.2](project_MD.html#numerical-variables-binning) says a unit
increase in a numerical predictor has the same linear effect on the target.
For the Poisson log-link model, it is the **log conditional mean/rate** that is
linear. A unit increase has a constant multiplicative effect, not a constant
additive change in expected claims.

**Correction:** qualify the scale, and avoid implying that GLMs cannot contain
nonlinear transformations or polynomial terms.

### I14 - WoE signs are explained using raw good/bad counts

**Error.** [Section 3.1.2.2](project_MD.html#weight-of-evidence-woe-and-information-value-iv-calculation)
says equal numbers of good and bad outcomes give WoE zero, and more good outcomes
imply positive WoE. The implemented formula compares **class-normalized
proportions**, `good_j / total_good` and `bad_j / total_bad`.

With approximately 89% zero-claim policies, a bin can contain many more good than
bad outcomes and still have negative WoE. Zero WoE means the bin has the
portfolio's good/bad odds, not equal good/bad counts.

### I15 - The finite-WoE safeguard is incomplete

**Error.** [Section 3.1.2.1](project_MD.html#baseline-quantile-binning) says ensuring
at least one claim in every bin prevents undefined WoE. The function only checks
positive-claim policies within bins. A bin with no zero-claim policies still
gives `log(0)`, and its IV can be infinite.

**Correction:** require both outcome classes in each bin, or use an explicitly
documented smoothing rule. This is a defect in the stated guarantee, not a claim
that the displayed dataset currently contains such a bin.

### I16 - The binning objective is not the frequency objective described

**Method.** [Section 3.1.2](project_MD.html#numerical-variables-binning) uses only
`NCLAIMS == 0` versus `NCLAIMS >= 1`. It ignores exposure and treats one and five
claims identically. Its IV therefore measures discrimination of claim
**occurrence over the observed duration**, not annual count frequency.

**Correction:** call this a binary surrogate and assess its implications, or use
exposure-aware count/deviance-based binning. Differences in duration can otherwise
influence the risk bins.

### I17 - The stated elbow-selection rule is not the actual selection rule

**Method / Consistency.** [Section 3.1.2](project_MD.html#numerical-variables-binning)
says the algorithm's elbow determines the final boundaries. In practice the
vertical markers identify **3 age**, **4 BM**, **3 car-age**, and **6 population**
bins, while the initial retained choices are **7, 3, 5, and 5**, respectively.

**Correction:** describe the elbow as a candidate-generating heuristic followed
by manual judgement, not an automatic optimal-bin procedure. Evaluate those
judgements inside resampling when claiming predictive performance.

### I18 - Reference-bin descriptions do not match their boundaries

**Error.** [Section 3.1.2.4](project_MD.html#bonus-malus-binning-bm) and the
[frequency-model introduction](project_MD.html#frequency-modeling) describe
`[0,1]` as BM zero. It contains **both 0 and 1**: **49,341** and **21,557**
training policies, respectively.

The reference car-age bin `[0,1]` is also described as “less than 1 year old,”
although it includes age 1. Use the actual closed/open intervals consistently;
otherwise the reference-policy interpretation is wrong.

### I19 - The power discussion cites a binning that is not shown

**Presentation.** [Section 3.1.2.6](project_MD.html#power-binning-power) refers to
two risk pockets at `bins=12`, but the displayed comparisons are **6, 8, 13,
and 23** bins. The narrow `(50,51]` pocket appears in the 13-bin comparison.

**Correction:** change 12 to 13 and distinguish that narrow pocket from the
wider retained `(47,51]` bin.

### I20 - The power explanations lack reliable, period-appropriate evidence

**Evidence.** [Section 3.1.2.6](project_MD.html#power-binning-power) calls 44-47 kW
a “minimum legal and practical power” for highway-legal cars and 50-51 kW a
mechanical “sweet spot.” The footnote explicitly attributes these interpretations
to an AI search, without a verifiable source. “Modern” vehicle-market explanations
also need justification for a **1997** portfolio.

**Correction:** substantiate the legal and engineering claims for the relevant
period or label them speculative hypotheses. The observed association does not
identify vehicle type or urban usage by itself.

## 3. Graphical and statistical exploration

### I21 - The normal-approximation derivation misidentifies the random quantity

**Error.** [Section 3.2.1](project_MD.html#graphical-approach) initially writes
`lambda ~ Normal(lambda_hat, ...)`, assigning a sampling distribution to the
fixed parameter. It also introduces `NCLAIMS_hat_i ~ Normal(...)` without defining
that estimator.

**Correction:** start with the sum of independent Poisson counts and obtain the
large-sample distribution of `lambda_hat`. Individual policy means are small
here, so a normal approximation for each individual count is not justified;
the aggregate approximation can nevertheless be good.

### I22 - The stacked exposure chart subtracts counts from policy-years

**Error.** [Section 3.2.1](project_MD.html#graphical-approach), in `plot_claim_freq`,
sets `sum_no_claims = sum_exp - sum_claims` and stacks it with claims. These
quantities have different units. The resulting grey segment is neither a count
of no-claim policies nor no-claim exposure.

**Correction:** show exposure and claim counts separately, or count policies by
claim status. Do not label exposure minus claims as “No Claims.”

### I23 - The plotted intervals are not the stated pairwise significance tests

**Method.** [Section 3.2.1](project_MD.html#graphical-approach) and footnote 12 use
single-bin confidence bands to test differences from other bins or the
portfolio mean. Overlap/non-overlap of separate 95% or 99% intervals is not the
corresponding pairwise test. The estimated portfolio mean also contains the bin,
so the two estimates are dependent.

The footnote's “95% certainty” wording is additionally not a frequentist
interpretation of a confidence interval.

**Correction:** use a contrast or an appropriate likelihood-ratio test for the
specified difference; describe intervals as confidence procedures, not posterior
probabilities.

### I24 - The claim that Gamma mean variance has no closed form is false

**Error.** [Section 3.2.1](project_MD.html#graphical-approach) says no closed-form
variance exists for average claim cost under a Gamma assumption. Gamma variance
is explicit: for independent equal-distribution claims,
`Var(mean of m claims) = phi * mu^2 / m`.

Estimating dispersion and allowing heterogeneous policy means require care, but
do not remove the closed-form variance relation. Explain that model fitting is
being used to estimate means and uncertainty, rather than because no formula
exists.

### I25 - The total-cost offset is inconsistently specified

**Error.** The prose in [section 3.2.1](project_MD.html#graphical-approach) says
`AMOUNT` uses an offset of `NCLAIMS`, while the working comparison correctly
uses `offset(log(NCLAIMS))`. The commented severity GAM in
[section 4.3.1](project_MD.html#smoothing-example-for-spatial-effect) again uses
the incorrect `offset(NCLAIMS)`.

**Correction:** consistently use the **log claim count** for a log-link
total-cost model. An offset of the raw count multiplies the mean by
`exp(NCLAIMS)`, not by `NCLAIMS`.

### I26 - Wider BM severity intervals are interpreted as bigger crashes

**Evidence / Error.** [Section 3.2.1](project_MD.html#graphical-approach) says
increasing variability at higher BM levels implies bigger crashes. The ribbons
are uncertainty intervals for estimated means, not estimates of within-group
crash-size variability. They widen substantially because there are fewer claims.

**Correction:** separate the upward point-estimate pattern from uncertainty and
test any severity contrast before asserting that high-BM drivers have bigger
claims.

### I27 - The coverage-frequency interpretation is not monotone

**Error.** [Section 3.2.1.1](project_MD.html#categorical-variables-1) says frequency
is higher for those choosing less coverage. The chart goes approximately
**TPL 0.145, PO 0.127, FO 0.134**. TPL is highest, but PO is lower than FO.

**Correction:** state that TPL differs from the omnium categories, without
claiming a monotone ordering across all three coverage levels.

### I28 - The initial coverage-merging suggestion contradicts the selected test level

**Consistency.** [Section 3.2.1.1](project_MD.html#categorical-variables-1) says the
omnium levels differ at 5% but could be merged on frequency alone.
[Section 3.2.2.2](project_MD.html#level-wise-anova-tests) confirms
**p = 0.0469242**, which rejects that merge under the report's **5%** rule.

**Correction:** say a frequency-only merge is not supported at 5%, although it
would not be rejected at 1%. A later multivariable merge can be valid, but that
is a different conditional comparison.

### I29 - “Clear” coverage-severity differences are not statistically established

**Evidence.** [Section 3.2.1.1](project_MD.html#categorical-variables-1) justifies
keeping PO and FO separate because severity shows a clear difference. The
severity bands are broad and overlap strongly. Reproducing the claim-weighted
Gamma model gives an approximate PO-versus-FO Wald **p = 0.240**.

**Correction:** describe a difference in point estimates, not an established
severity distinction. Keeping levels for business reasons is possible, but
should be identified as such.

### I30 - Univariate non-significance is treated as proof of irrelevance

**Method.** [Sections 3.2.1.1](project_MD.html#categorical-variables-1) and
[3.2.2.1](project_MD.html#variable-wise-anova-tests) exclude `USE` from subsequent
models because its marginal test is not significant and it is imbalanced.

Non-significance is not proof of no predictive information, and imbalance alone
does not make a predictor unusable. A predictor can become informative after
adjustment or through interactions.

**Correction:** call this a preliminary screening decision and assess conditional
or out-of-sample relevance before a categorical exclusion. The report's own
`FLEET` example shows that a rare level can be informative.

### I31 - The fleet variability description names the wrong category

**Error.** [Section 3.2.1.1](project_MD.html#categorical-variables-1) refers to high
volatility of the **non-fleet** category immediately after explaining that it
dominates the portfolio. The broad bands belong to the rare **fleet** category.

**Correction:** replace “non-fleet” with “fleet.”

### I32 - The likelihood-ratio null hypothesis is incorrectly written

**Error.** [Section 3.2.2.1](project_MD.html#variable-wise-anova-tests) writes
`H0: D0 - D1 = 0`. Under the null, the **additional population coefficients are
zero**; the observed deviance reduction is generally positive because the larger
model can fit sampling noise.

**Correction:** formulate the null in terms of coefficients or equal constrained
effects. The deviance difference is approximately chi-squared **under** that null;
it is not required to equal zero.

### I33 - The coverage-test interpretation reverses the hypothesis

**Error.** [Section 3.2.2.2](project_MD.html#level-wise-anova-tests) says
**p = 0.0469242** means rejecting the hypothesis that the deviances “were
different.” That reverses the test.

**Correction:** at 5%, reject the equal-effect/merged-level model and retain the
distinction; at 1%, fail to reject it. Non-rejection at 1% is not proof that the
effects are equal.

## 4. Geographic enrichment

### I34 - Postal geography and the NIS administrative hierarchy are conflated

**Error.** [Section 3.3.1](project_MD.html#geospatial-features) treats postal codes
as part of a nested NIS hierarchy, omits the municipality level from its list,
and describes roughly 600 postal codes as a Belgian total. Postal codes are a
separate system; the conversion table's ambiguous 1040/1050/1804 mappings already
show that postal and municipal boundaries are not strictly nested.

The **583** codes are those in this portfolio, not a national count. Calling the
region the “highest level of granularity” also reverses fine versus coarse
resolution.

**Correction:** distinguish postal areas, municipalities/REFNIS, statistical
sectors, and higher administrative levels, and separate sample counts from
national counts.

### I35 - POPULATION is a count, not population density

**Error.** [Section 3.3.1.2](project_MD.html#population-density), its surrounding
discussion, and [section 4.2.3](project_MD.html#enriched-features) repeatedly call
the enrichment population density. The join imports `Totaal`, the **total
municipal population**, and never divides by area.

**Correction:** rename the interpretation to municipal population size, or obtain
consistent municipality areas and calculate inhabitants per square kilometre.
The current results do not isolate density or traffic congestion.

### I36 - Geographic identifiers do not consistently match the 1997 population data

**Error / Method.** [Section 3.3.1.1](project_MD.html#dataset-enrichment) combines
the postal/municipal conversion with the 1997 population table as though their
identifiers were aligned.

For example, the [conversion table](../data/postcode_refnis_conversion.csv)
uses code **46030** for Beveren-Kruibeke-Zwijndrecht, also present in the
[municipality table](../data/refnis_matching.csv), whereas the
[1997 population table](../data/pop_1997.csv) contains Beveren under **46003**.
The mismatch is not confined to that example: **11,163 training policies** have
no population match, including all **1,806 La Louviere policies**.

**Correction:** harmonize code vintages and boundaries, or use a
period-appropriate crosswalk before joining. Mean imputation does not repair
historically incompatible identifiers.

### I37 - The missing-population count is measured only after it has been erased

**Error / Reporting.** [Section 3.3.1.1](project_MD.html#dataset-enrichment) says
there are **0** missing population values and then says they will be imputed.
The code has already replaced missing values before counting them.

Before that replacement there are **11,163 missing matches**, about **8.55%**
of training policies. They all receive **56,488.20** inhabitants, an artificial
shared value that also affects the bins and exploratory conclusions.

**Correction:** show missing counts before and after imputation, explain their
causes, and disclose the policy-weighted imputation method.

### I38 - The imputation rule changes when new data are processed

**Method / Consistency.** [Section 4.1.1](project_MD.html#data-preparation-workflow)
recomputes `mean(POPULATION, na.rm = TRUE)` inside `data_enrichment(input_data)`.
Thus processing the holdout fits a different imputation value instead of applying
a training-fitted transformation:

- Training: **56,488.20**, with **11,163** missing matches.
- Holdout: **56,840.29**, with **2,754** missing matches.

Both happen to fall in the same current population bin, so this does not by itself
change that binned predictor's assignments. It does change the continuous feature
intended for GAMs and contradicts a fixed preparation pipeline.

**Correction:** estimate imputation on the training/analysis sample and reuse the
stored value on validation or holdout data.

### I39 - Regional comparisons are overgeneralized into an urban/rural result

**Evidence.** [Section 3.3.1.3](project_MD.html#regions) compares Brussels with
all other regions, then calls this confirmation of an urban/rural hypothesis.
“Others” includes major urban municipalities, so it is not a rural group.
The prose itself also acknowledges that the severity difference is not
statistically significant, but subsequently calls the hypothesis confirmed.

**Correction:** retain the descriptive Brussels-versus-rest result, distinguish
estimated severity ratios from established differences, and use an actual
urban/rural or density measure before generalizing.

### I40 - Expected claim cost is labelled as an expected number of claims

**Terminology / Units.** The tables in [sections 3.3.1.3](project_MD.html#regions)
and [3.3.1.4](project_MD.html#municipalities) name
`frequency * severity` as `expected_claims` and caption the result “Expected
claims.” Values such as Brussels **274.07** and Charleroi **746.93** are
**euros per policy-year**, not claim counts.

**Correction:** use `expected_annual_claim_cost` or `pure_premium`, and state the
currency and exposure basis.

## 5. Polynomial exploration and interaction discovery

### I41 - The raw quadratic AGEC conclusion contradicts its heatmap

**Error.** [Section 3.3.2](project_MD.html#polynomial-features) says AGEC is not
significantly correlated with claim counts at either degree one or degree two.
The raw-polynomial heatmap shows degree-two **p = 0.0403**, significant under
the report's 5% threshold. This was reproduced as **0.04032823**.

**Correction:** distinguish the non-significant linear term from the nominally
significant quadratic term. Apply any multiplicity adjustment explicitly rather
than silently switching the threshold.

### I42 - The orthogonal frequency analysis drops every zero-claim policy

**Error / Method.** [Section 3.3.2](project_MD.html#polynomial-features) uses
`no_na <- na.omit(data_enriched)`. Because `AVG` is missing for zero-claim
policies, this reduces **130,585 policies to 14,570**, with **zero zero-claim
policies remaining**.

The `OrthClaims` columns therefore analyse counts conditional on having a claim,
not portfolio claim frequency. Comparing them with the raw-frequency heatmap
confounds a change of basis with a change of population.

**Correction:** select complete cases only for the predictors needed in each
analysis; retain zero-claim policies for frequency.

### I43 - Correlation tests are presented as incremental Poisson/Gamma term tests

**Method.** [Section 3.3.2](project_MD.html#polynomial-features) uses Pearson
correlation p-values to infer the contribution of polynomial terms to the
frequency/severity models. Frequency correlations ignore exposure; severity
correlations do not use claim-count weights. Orthogonality under an unweighted
sample inner product also does not automatically make terms orthogonal under a
Poisson/Gamma likelihood.

**Correction:** treat the heatmaps as limited descriptive screens. Test incremental
terms with the intended exposure-offset Poisson model or claim-weighted Gamma
model on a consistent sample. Smaller p-values alone do not measure stronger
predictive relationships.

### I44 - The heatmap color key contradicts its thresholds

**Presentation.** The [section 3.3.2](project_MD.html#polynomial-features) titles
say “Green = Significant p<0.05,” but the code colors:

- `p < 0.01`: green;
- `0.01 <= p < 0.05`: white;
- larger p-values: red.

Thus significant results such as **0.0403**, **0.0172**, **0.0343**, and **0.0252**
are white, not green.

**Correction:** explain all three colors or use a two-color 5% scheme.

### I45 - The interaction-discovery Poisson tree ignores exposure

**Error / Method.** [Section 3.3.3](project_MD.html#interaction-features) fits
`rpart(NCLAIMS ~ ..., method = "poisson")` without exposure. Its root rate is
approximately **0.123 claims per policy**, rather than **0.138624 per
policy-year**.

**Correction:** use the Poisson-tree exposure/count response, for example
`cbind(EXP, NCLAIMS)`, and regenerate the splits and importance ranking before
using them to select annual-frequency interactions.

### I46 - Requesting tree cross-validation does not stabilize the selected tree

**Method.** [Section 3.3.3](project_MD.html#interaction-features) says 10-fold CV
mitigates tree instability. The code sets `xval = 10` but fixes `maxdepth = 3`,
sets `cp = 0`, and never uses the CV error to prune/select a tree or measures
split stability.

**Correction:** use the CV results in a stated selection rule and assess
stability separately. A depth limit alone does not establish “without
overfitting.”

### I47 - The listed tree-based interactions do not match the fitted interactions

**Error.** [Section 3.3.3](project_MD.html#interaction-features) says interactions
use the three most important variables. The ranking is **BM, AGEPH, REGION**,
but one proposed pair is **REGION x POWER**.

[Section 4.2.4](project_MD.html#interaction-effects) repeats REGION x POWER,
while the actual `interaction_features` and printed formula use
**REGION x AGEPH** instead.

**Correction:** choose the intended pair and align the importance rationale,
feature list, formula, and interpretation.

### I48 - The promised expert interaction is never fitted

**Error / Incomplete.** [Section 4.2.4](project_MD.html#interaction-effects) says
the expert-motivated **AGEPH x POWER** interaction will be considered. It is
absent from `interaction_features` and from both fitted interaction formulas.

**Correction:** actually evaluate it, or remove the claim that it has been
considered. Its performance cannot be inferred from the interactions that were
fitted.

## 6. Cross-validation and performance metrics

### I49 - Supervised preparation is performed before the CV folds

**Method.** [Section 4.1.1](project_MD.html#data-preparation-workflow) prepares the
entire training dataset once. Its WoE boundaries, chosen bin counts, ANOVA
merges, screening decisions, and interaction discovery have already used the
targets from policies later treated as validation observations.

The recipes inside `fit_resamples` therefore refit model coefficients, not the
complete target-dependent preparation/selection process.

**Correction:** learn supervised transformations and selections within each
analysis fold, ideally with nested evaluation for tuning. Alternatively, state
that the reported CV is conditional on globally selected features and can be
optimistic. The separate holdout has not thereby been used for target training.

### I50 - The prediction-error derivation mixes incompatible loss scales

**Error.** [Section 4.1.2](project_MD.html#performance-evaluation) begins with a
squared-error covariance identity, then replaces squared error with Poisson
deviance and imports a `2(p+1)/n` penalty without establishing that change of
loss or scale.

It also calls `p` the number of parameters while adding another intercept through
`p+1`; the implementation instead uses `k = rank`, including the intercept.

**Correction:** choose and define a loss consistently. For a Poisson
likelihood-based training criterion, use the appropriate deviance/AIC
relationship and `k` consistently. Do not present it as a derivation of mean
squared prediction error.

### I51 - CV deviance is penalized a second time

**Error / Method.** [Section 4.2](project_MD.html#frequency-glm-workflow) assigns
`training_deviance <- CV_Deviance` and reports
`CV_Prediction_Error = CV_Deviance + 2*k`.

Out-of-fold deviance already evaluates unseen observations; `2*k` is an
AIC-style correction for **in-sample** fit, not a required optimism correction
to that CV estimate.

**Correction:** report out-of-fold deviance directly. If an additional
complexity preference is intentional, label it a custom selection criterion and
justify it instead of calling it the estimated prediction error.

### I52 - CV log loss and CV deviance are not independent confirmations

**Error / Interpretation.** [Sections 4.1.2](project_MD.html#performance-evaluation)
and [4.2.5](project_MD.html#selection-of-the-best-model) present these as
complementary evidence of different prediction properties.

For the same observations and Poisson predictions they obey exactly:

`CV_Deviance = 2*n*CV_LogLoss + 2*logLik(saturated model)`.

The second term is constant across the candidates, so their rankings must agree.
AIC and training deviance plus `2*k` are similarly equivalent up to a constant
for these models.

**Correction:** explain the equivalence rather than treating agreement as
independent corroboration.

### I53 - The “ANOVA test based on CV Deviance” has no stated chi-squared basis

**Error / Method.** [Section 4.2.1](project_MD.html#categorical-variables-only)
takes an out-of-fold deviance difference and sends it to `pchisq` using the
full-fit parameter-count difference.

A standard likelihood-ratio chi-squared result concerns nested maximum-likelihood
fits to the **same training observations**, not predictions from several
overlapping training folds.

**Correction:** retain the ordinary same-data nested-model test and compare CV
scores with an appropriate resampling/uncertainty procedure. Do not label the
out-of-fold calculation an ANOVA likelihood-ratio test.

### I54 - Poisson significance and uncertainty are used without dispersion validation

**Method / Evidence.** From [section 3.2.2](project_MD.html#statistical-approach)
through [section 4.2](project_MD.html#frequency-glm-workflow), confidence bands,
Wald tests, and chi-squared deviance tests rely on the Poisson variance model,
but no dispersion/residual check is reported.

The reproduced enriched model has Pearson dispersion about **1.1603**, not 1.
This does not prove that every conclusion changes, but marginal decisions such
as the **SEX p = 0.0266** and coverage merge deserve sensitivity checks.

**Correction:** report diagnostics and investigate the variance assumption;
if needed, use an appropriate alternative or uncertainty treatment and matching
model-comparison tests.

## 7. GLM interpretation and model selection

### I55 - The omnium multiplier is rendered as NA

**Error.** [Section 4.2.1](project_MD.html#categorical-variables-only) prints an
omnium multiplier and interval as **NA / NA / NA**. It indexes
`COVERAGE_BINOMN`, but the fitted model contains separate `COVERAGEPO` and
`COVERAGEFO` coefficients.

**Correction:** report the two actual multipliers. Reproduced values are:

| Coverage versus TPL | Multiplier | Approximate 95% CI |
|---|---:|---:|
| PO | 0.8748 | 0.8438-0.9070 |
| FO | 0.9085 | 0.8665-0.9524 |

A merged omnium coefficient exists only after fitting a merged-coverage model.

### I56 - The stated reasons for skipping candidate-model CV are not borne out

**Error / Method.** [Section 4.2.3](project_MD.html#enriched-features) says
municipality levels are too small and would be absent from CV folds;
[section 4.2.4](project_MD.html#interaction-effects) similarly skips CV for
interactions.

Reproducing the stated five folds gives:

- Smallest municipality: **1,649 policies and 222 claims**.
- Minimum municipality count in any analysis fold: **1,290**.
- Minimum municipality count in any assessment fold: **305**.
- All five municipality fits and all five interaction fits predict successfully,
  with **zero missing predictions**.
- Model ranks remain **30** and **41**, respectively, in every fold.

**Correction:** distinguish structural aliasing from absent levels, and evaluate
these candidates with the common workflow. Their exclusion also contradicts the
claim that the workflow is used for all frequency models.

### I57 - The full interaction model is rejected without the promised comparison

**Consistency / Method.** [Section 4.1.2](project_MD.html#performance-evaluation)
says significant complexity layers become the new benchmark. Yet
[section 4.2.4](project_MD.html#interaction-effects) finds the full interaction
layer significant and then discards it largely for interpretability. It tests
only the two BM/FUEL indicators afterwards, rather than pruning and evaluating
the significant BM/REGION interaction.

Reproduced comparisons, using the report's existing prepared features:

| Candidate | AIC | BIC | Five-fold deviance |
|---|---:|---:|---:|
| Enriched | 99,575.66 | 99,829.94 | 69,552.74 |
| Full interaction model | 99,546.67 | 99,947.64 | 69,545.28 |

The interaction likelihood-ratio test gives approximately **p = 3.76e-7**.
These results do **not** establish that the interaction model is the final best
choice: BIC favors the enriched model, preprocessing remains conditional, and
the CV differences need uncertainty assessment. They do show that the interaction
candidate cannot be dismissed as uniformly worse or impossible to validate.

**Correction:** include these candidates in the leaderboard, assess a
hierarchy-respecting pruned interaction model, and state how predictive fit,
complexity, and interpretability are traded off.

### I58 - An interaction changes the meaning of the Brussels main coefficient

**Error / Interpretation.** [Section 4.2.4](project_MD.html#interaction-effects)
says Brussels no longer has a significant effect because it is included in
interactions and confounded with other variables.

With interactions, the main Brussels coefficient tests its effect **at the
reference BM and age categories**, not whether region has any overall effect.
The BM/Brussels interaction is itself significant.

**Correction:** interpret conditional effects or test region and its interactions
jointly; do not infer disappearance of a regional effect from that one main-term
p-value.

### I59 - The Gand alias explanation is off by a factor of ten

**Error.** [Section 4.2.3](project_MD.html#enriched-features) describes the
`(2.05e+05,4.53e+05]` population bin as **20,500-45,300** people. It means
approximately **205,000-453,000**.

**Correction:** fix the numbers. Gand's joined population is **225,469**, which
is consistent with the actual bin. The aliasing demonstration itself is not
invalidated by this prose error.

### I60 - Tiny CV improvements are treated as demonstrated better generalization

**Method / Evidence.** [Section 4.2.5](project_MD.html#selection-of-the-best-model)
says the final merges generalize better because CV deviance/log loss improve.
The unmerged enriched model has deviance **69,552.74** versus **69,548.24** after
all merges, and mean log loss improves by only about **0.0000172**.

**Correction:** say the point estimates are slightly better on this split.
Report fold-wise paired differences, repeated-resampling stability, or other
appropriate uncertainty before treating the ordering as a robust improvement.

## 8. GAM workflow through section 4.3.2

### I61 - Two incompatible GAM baselines are specified

**Consistency.** [Sections 4.2.5](project_MD.html#selection-of-the-best-model) and
[4.3](project_MD.html#frequency-gam-workflow) say the final simplified, enriched
GLM will be the GAM base.
[Section 4.3.2](project_MD.html#smooth-terms-for-continuous-variables), like the
earlier approach outline, instead says to use the **categorical-only GLM** as the
baseline and add enrichment and smooths.

**Correction:** specify one modelling sequence, including whether final coverage
merges and enriched predictors are retained when binned numerical terms are
replaced by smooths.

### I62 - GAM and GLM significance numbers do not test GAM superiority

**Method.** [Section 4.3.1](project_MD.html#smoothing-example-for-spatial-effect)
compares the GAM's joint spatial smooth p-value with the GLM's separate LONG and
LAT p-values and calls the former a far more significant predictor.

These are different hypotheses and different degrees of freedom. A joint
two-variable linear test is missing, and a smooth's significance against no
spatial effect is not a test of improvement over the linear spatial model.

**Correction:** compare compatible models/hypotheses and out-of-sample scores.
The contour illustration can still demonstrate a nonlinear fitted pattern.

### I63 - Degrees of longitude and latitude do not justify physical isotropy

**Method.** [Section 4.3.1](project_MD.html#smoothing-example-for-spatial-effect)
justifies an isotropic `s(LONG, LAT)` because both coordinates use the same units.
At Belgian latitudes, one longitude degree spans only about **63-64%** of the
distance of one latitude degree.

**Correction:** use a suitable projected metric coordinate system if isotropic
physical distance is intended, or explicitly justify smoothing in angular
coordinates. A tensor product is not automatically required or forbidden simply
because the input unit labels match.

### I64 - The promised manual backfitting demonstration is absent

**Consistency / Incomplete.** [Section 2.3](project_MD.html#approach) promises to
show GAM logic through **manual backfitting** for latitude and longitude.
[Section 4.3.1](project_MD.html#smoothing-example-for-spatial-effect) instead makes
one `bam` call with a joint bivariate smooth. No manual alternating fit,
partial-residual update, or backfitting iteration is shown.

**Correction:** change the outline to describe the actual spatial-smoothing
illustration, or add the promised demonstration.

### I65 - The commented severity-GAM conclusion is not supported by shown results

**Evidence.** The folded code in
[section 4.3.1](project_MD.html#smoothing-example-for-spatial-effect) asserts that
severity spatial effects are non-significant, but both severity fits are
commented out and no test statistic or p-value is supplied.

**Correction:** remove the unsubstantiated conclusion, identify it as an
unreported prior exploration, or show the relevant fitted result. The
total-cost version would also need the offset correction in I25 before reuse.

### I66 - Section 4.3.2 contains an instruction, not the described analysis

**Incomplete.** [Section 4.3.2](project_MD.html#smooth-terms-for-continuous-variables)
contains only “Use the categorical-only frequency GLM as the baseline, then add
enriched features and smooth terms for continuous variables.”

There is no candidate formula, list of retained smooths, fit, smooth-term table,
or result. This is a visible drafting placeholder within the requested review
boundary.

**Correction:** implement and document this step, or explicitly mark the report
as unfinished at this point.

## 9. Rendering and editorial inconsistencies

### I67 - One mathematical footnote is empty and another reference is unresolved

**Presentation.** Footnote 4, referenced from
[section 2.2.2.2](project_MD.html#deep-dive-into-exposure-exp), says the standard
error can be estimated as **`$ $`** in the HTML. The expression is present in the
current source, so the reviewed HTML does not contain that source content.

[Section 3.2.1](project_MD.html#graphical-approach) also displays the literal
unresolved marker **`[^demonstration]`**.

**Correction:** repair/remove the undefined reference, use valid math
delimiters, re-render, and check the actual HTML output.

### I68 - The merged-age and merged-power figures are each printed twice

**Presentation.** [Section 3.2.2.2](project_MD.html#level-wise-anova-tests) contains
two identical plots for merged AGEPH and two identical plots for merged POWER.
The caller uses `plot(plot_freq_sev(...))` even though `plot_freq_sev` already
prints its plot.

**Correction:** print at just one level.

### I69 - Some figures are clipped or compressed enough to hide their content

**Presentation.**

- In [section 3.3.1.3](project_MD.html#regions), the long region names compress
  the frequency/exposure panels into very thin strips and overlap axis labels.
- In [section 4.2.4](project_MD.html#interaction-effects), the large interaction
  coefficient chart has overlapping labels and a clipped title.
- In [section 4.2.5](project_MD.html#selection-of-the-best-model), several
  leaderboard value labels are clipped at panel boundaries, including the
  categorical and intercept-only scores.

**Correction:** shorten display labels, allocate suitable panel heights and
widths, and reserve enough plot space for annotations. Validate the rendered
figures, not just the plotting code.

### I70 - Formatting and naming drift obscure the intended comparisons

**Presentation.**

- [Section 3.3.2](project_MD.html#polynomial-features) renders its “Note that”
  items as one paragraph with inline hyphens rather than a proper bullet list.
- [Section 4.2.5](project_MD.html#selection-of-the-best-model) alternates between
  `POWER_binned`, the fitted `POWER_binned_merged`, and the nonexistent name
  `POPULATION_levels`.
- Omnium naming changes from `OMN` to `OM`; harmless if local and intentional,
  but it contributes to the stale coefficient reference in I55.
- Correct obvious editing remnants such as “thereforeuse,” “constribution,”
  “withouf,” “correllations,” “geospation,” “LONG in LAT,” and “becayse.”

**Correction:** make variable names and comparison labels agree with the
actual fitted objects, and complete an editorial/rendering pass after the
substantive fixes.

## Suggested correction order

1. Repair and disclose geographic matching; decide whether population size or
   actual density is intended.
2. Correct the frequency/severity identities, WoE explanation, and hypothesis
   interpretations.
3. Retain zero-claim policies in frequency analyses and include exposure in the
   interaction-discovery tree.
4. Rebuild the resampling design, learn supervised preparation within folds, and
   remove the invalid CV penalty and CV chi-squared calculation.
5. Align and evaluate the intended interactions and all feasible candidate
   models, then revisit the leaderboard and conclusions.
6. Resolve the GAM baseline and complete section 4.3.2.
7. Re-render and check formulas, footnotes, duplicate plots, and clipped labels.

The reproduced numbers above verify specific issues in the current snapshot.
They are not replacement final-model results: correcting enrichment,
preprocessing, or candidate selection requires rerunning the affected analysis.
