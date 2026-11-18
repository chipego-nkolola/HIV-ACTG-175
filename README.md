# Age, Treatment, and Immune Recovery in HIV: An Analysis of ACTG 175

This project analyzes data from AIDS Clinical Trials Group Study 175 (ACTG 175), a randomized trial of HIV-1-infected adults. It asks three things:
- Do age and treatment regimen affect how much a patient's CD4 count recovers in the first 20 weeks of therapy?
- Does age change how well each treatment works?
- Can early CD4 response help predict clinical failure?

## Research questions

**Primary question.** After controlling for baseline CD4 (`cd40`), how much do age and treatment regimen (`trt`) each contribute to 20-week CD4 change (`cd420 - cd40`)? Does age change the effect of treatment?

**Sub-question A: baseline status.** Do older patients enter the trial with lower baseline CD4 or a lower CD4:CD8 ratio (`cd40 / cd80`)?
*Method:* compare across age brackets (<30, 30–39, 40–49, 50+) with a one-way ANOVA, and treat age as continuous in a simple regression.

**Sub-question B: treatment interaction.** Compared with ZDV alone, does the CD4 gain from ZDV + ddI, ZDV + Zal, or ddI alone differ by age?
*Method:* test the `age × trt` interaction in an OLS model that includes `cd40` as a covariate. The model uses HC3 robust standard errors.

**Sub-question C: clinical failure.** Do age and treatment predict clinical failure (`cid`)? Does early CD4 change add predictive value beyond them?
*Method:* logistic regression evaluated with ROC-AUC on a held-out, stratified test set.

## Key findings

Results are from the run with `random_state = 42`. See the Findings section of the notebook for estimates, confidence intervals, and caveats.

| Question | Result |
|---|---|
| A: baseline status | Age has at most a weak link to baseline immune status. Mean `cd40` is 344–361 cells/mm³ in every age bracket (ANOVA p ≈ 0.11). The narrow range is consistent with the trial's 200–500 CD4 enrollment criterion. |
| Primary: drivers of CD4 change | Baseline CD4 explains the most variance (unique ΔR² ≈ 0.077), followed by treatment arm (ΔR² ≈ 0.041). Age on its own adds almost nothing (ΔR² ≈ 0.001). At mean age, ZDV + ddI gains about +70 cells/mm³ over ZDV alone. |
| B: age × treatment | The interaction is significant overall (joint Wald p ≈ 0.013). It comes mainly from ZDV + ddI, whose advantage over ZDV grows by about 18 cells per decade of age. This is exploratory: one of three terms is significant, the confidence interval is wide, and only 123 patients are aged 50+. |
| Predicting CD4 change | Held-out R² is ≈ 0.10 for `cd4_change`. Predicting `cd420` gives R² ≈ 0.36, but only because `cd420` tracks `cd40`. |
| C: clinical failure | Test ROC-AUC is 0.61 with age + treatment, 0.68 after adding baseline CD4, and 0.76 after adding 20-week CD4 change. |

## Dataset

**AIDS Clinical Trials Group Study 175**, UCI Machine Learning Repository (dataset ID 890).
<https://archive.ics.uci.edu/dataset/890/aids+clinical+trials+group+study+175>

> "The AIDS Clinical Trials Group Study 175 Dataset contains healthcare statistics and categorical information about patients who have been diagnosed with AIDS. This dataset was initially published in 1996. The prediction task is to predict whether or not each patient died within a certain window of time or not."

The dataset has 2,139 patients, 23 features, and 1 target (`cid`), with no missing values.

### Data source and loading

The data loads through the [`ucimlrepo`](https://github.com/uci-ml-repo/ucimlrepo) package. During development, that repository was git-cloned over HTTPS for reference. It is not included in this repo; install it with pip instead. The notebook loads the data with the snippet provided on the UCI dataset page:

```python
from ucimlrepo import fetch_ucirepo

aids = fetch_ucirepo(id=890)
X = aids.data.features   # 23 feature columns
y = aids.data.targets    # cid
```

Loading needs an internet connection the first time.

### Key variables

| Variable | Description |
|---|---|
| `cd40` | Baseline CD4+ T-cell count (cells/mm³) at trial entry. Enrollment required screening CD4 of 200–500. Because CD4 varies between the screening and randomization visits, some baseline values fall outside that range (17.6% here); all are kept. |
| `cd420` | CD4+ count about 20 weeks after starting therapy. It is an early indicator of treatment response. |
| `cd80` | Baseline CD8+ T-cell count (cells/mm³). It is often elevated in HIV because of chronic immune activation. |
| `cd4_change` | *Derived:* `cd420 - cd40`. A positive value means immune reconstitution on therapy. |
| `cd4_cd8_ratio` | *Derived:* `cd40 / cd80`. A ratio below 1.0 marks immune dysregulation; healthy adults are around 1.5–2.5. |
| `trt` | Treatment arm: 0 = ZDV only (reference), 1 = ZDV + ddI, 2 = ZDV + Zal, 3 = ddI only |
| `age` | Age in years at baseline (range 12–70) |
| `cid` | Target. 1 = clinical failure event, 0 = censored. The event rate is about 24%. |

**Biological context.**
- **CD4+ (helper) T-cells** coordinate the adaptive immune response. HIV infects and destroys them, which makes CD4 count the primary marker of disease stage.
- **CD8+ (cytotoxic) T-cells** kill virus-infected cells. Their counts often rise during HIV infection.

### Modeling notes

- **`time` is excluded from the `cid` models.** It is days until the event or censoring, so it would leak the outcome.
- **`treat` is excluded from the `cid` models.** It is a yes/no for "anything other than ZDV only", so it duplicates `trt`.
- **`cd4_change` uses the week-20 CD4 count.** 13 of the 521 failures occur before day 140, so the full `cid` model uses information from slightly after those few events.
- **Age is centered at its mean (35.2 years)** in the interaction model. As a result, the treatment coefficients describe a patient of average age.

## Visualizations

1. **Baseline status by age:** boxplots of `cd40` and the CD4:CD8 ratio by age bracket, with sample sizes annotated.
2. **Age vs. CD4 change by arm:**
   - A faceted `lmplot`, one panel per treatment arm. It is *not* adjusted for `cd40`.
   - A forest plot of the `age × trt` interaction coefficients with 95% CIs. It *is* adjusted for `cd40`.
3. **Model diagnostics:** test-set actual vs. predicted, residuals vs. predicted, and a Q-Q plot of residuals.
4. **Clinical failure:** ROC curves for the three nested logistic models, a confusion matrix, and an adjusted odds-ratio forest plot.

## Project structure

```
HIV_personal_project/
├── age_treatment_immune_recovery_HIV.ipynb    # full analysis
├── requirements.txt                           # Python dependencies
├── LICENSE
└── README.md
```

## How to run

1. Clone this repo and create a Python 3.12 environment. The project was developed in Anaconda `base`.
2. Install the dependencies:
   ```bash
   pip install -r requirements.txt
   ```
3. Open the notebook in Jupyter or VS Code.
4. Run all cells top to bottom. The data downloads automatically from UCI, so an internet connection is needed.

### Libraries

| Library | Version used |
|---|---|
| pandas | 2.2.2 |
| NumPy | 1.26.4 |
| Matplotlib | 3.8.4 |
| Seaborn | 0.13.2 |
| statsmodels | 0.14.2 |
| scikit-learn | 1.4.2 |
| ucimlrepo | 0.0.7 |

## References

- Hammer SM, et al. *A trial comparing nucleoside monotherapy with combination therapy in HIV-infected adults with CD4 cell counts from 200 to 500 per cubic millimeter.* New England Journal of Medicine, 1996; 335:1081–1090.
- UCI Machine Learning Repository: [AIDS Clinical Trials Group Study 175](https://archive.ics.uci.edu/dataset/890/aids+clinical+trials+group+study+175)
- [`ucimlrepo` on GitHub](https://github.com/uci-ml-repo/ucimlrepo)
