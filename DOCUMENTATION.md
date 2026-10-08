# Project documentation: predicting loan amount for accepted LendingClub applications

Data Visualization class project, Part A. This file records what was done in each step, why, and what was found, so a teammate can pick up the work without asking. The notebook `notebooks/01_data_prep.ipynb` holds the code and its outputs; this file is the summary.

## Status board

| Step | What | Owner | State | Where |
|---|---|---|---|---|
| 1 | Dataset, motivation, complexity | dhruvinn25 | Done | notebook section 1 |
| 2 | Load dataset | dhruvinn25 | Done | notebook section 2 |
| 3 | Pre-cleaning insights and visuals | dhruvinn25 | Done | notebook section 3 |
| 4 | Data cleaning | dhruvinn25 | Not started | notebook section 4 |
| 5 | Pre vs post cleaning visuals | dhruvinn25 | Not started | notebook section 5 |
| 6 | ML direction | Teammate | Not started | |
| 7 | Feature selection, feature-to-target, correlation heatmap | Teammate | Not started | |
| 8 | Decision table | Teammate | Not started | |
| 9 | Tableau dashboard | Teammate | Not started | |

## How to reproduce

1. Download `accepted_2007_to_2018Q4.csv.gz` from <https://www.kaggle.com/datasets/wordsforthewise/lending-club> (Kaggle login needed).
2. Put it in `data/raw/`. Data files are not in git: they are too large for GitHub's 100 MB limit.
3. `pip install -r requirements.txt` (pandas, pyarrow, numpy, matplotlib, seaborn, jupyter).
4. Open `notebooks/01_data_prep.ipynb` and run all cells from top to bottom.

Environment the notebook was last run in: Windows 11, 16 GB RAM, Anaconda Python 3.14, pandas 3.0.3, pyarrow 23.0.1, matplotlib 3.11.0, seaborn 0.13.2.

## Files

| Path | What it is | In git? |
|---|---|---|
| `notebooks/01_data_prep.ipynb` | Steps 1 to 5, executed, with outputs | Yes |
| `DOCUMENTATION.md` | This file | Yes |
| `outputs/figures/` | Charts saved by the notebook, for slides and the report | Yes |
| `data/raw/` | The Kaggle file | No |

---

## Step log

### Step 1: dataset, motivation and complexity

**What was done**

- Wrote section 1 of the notebook in eight parts: the question and who cares, source, size and features, why this dataset, why loan amount as the target, data complexity, problem complexity, and a complexity scorecard.
- Every number in the write-up comes from a cell further down the notebook, and the write-up names that cell.
- Grouped the 151 columns into seven kinds of information and charted the counts (`outputs/figures/01_feature_groups.png`).

**Why**

- **Dataset:** real, large (2.26M loans, 151 columns), mixed types, twelve years of history and untidy enough to need real cleaning.
- **Accepted file only:** the rejected-applications file in the same Kaggle dataset has 9 columns (amount requested, application date, loan title, risk score, DTI, zip code, state, employment length, policy code). No income and no credit history, so it cannot explain loan size. Checked by reading that file's header.
- **Target `loan_amnt`:** a dollar amount, so Part B is a regression task. It is complete for every loan.
- **Order inside the notebook:** the write-up sits at the top, but its evidence is produced by the load in section 2. Nothing about specific columns was written before the output it depends on had been seen.

**What we found**

| Group | Columns | Median % missing |
|---|---|---|
| Credit bureau file | 69 | 3.1 |
| Loan terms and listing | 19 | 0.0 |
| Post-issue performance | 17 | 0.0 |
| Joint / second applicant | 16 | 95.2 |
| Hardship plan | 15 | 99.5 |
| Borrower profile | 8 | 0.0 |
| Debt settlement | 7 | 98.5 |

- Only 8 columns describe the borrower directly; almost half the file is the credit report.
- 55 columns describe what happened after the loan was issued, or apply to a small share of loans.

**Outside facts checked**

- LendingClub closed its investor (Notes) platform on 31 December 2020 ([Finextra](https://www.finextra.com/pressarticle/84439/lendingclub-sunsets-notes-platform), [Banking Dive](https://bankingdive.com/news/lendingclub-peer-to-peer-loans/586841)). The company still lends; only the peer-to-peer investor side closed.
- The maximum loan size was raised to \$35,000 in 2011 ([PR Newswire](https://www.prnewswire.com/news-releases/lending-club-increases-maximum-personal-loan-size-to-35000-116217554.html)) and to \$40,000 in March 2016 ([deBanked](https://debanked.com/2016/03/lending-club-loan-size-cap-raised-to-40000-should-investors-be-worried/)). The data shows the same two steps.
- `loan_amnt` is the amount the borrower applied for, lowered if LendingClub's credit department reduced it. This wording comes from a copy of LendingClub's data dictionary (`LCDataDictionary.xlsx`); the original on LendingClub's site could not be opened, so treat the exact wording as likely but not confirmed at source.

**Decisions later steps must respect**

- **Leakage:** `funded_amnt`, `funded_amnt_inv`, `installment` and every post-issue field are never model features. They restate the target or only exist after the loan is issued.
- **Two candidate feature sets, to be compared in step 7:**
  - Set A, application-time only: income, employment, home ownership, DTI, credit history, purpose, state.
  - Set B: Set A plus LendingClub's pricing (term, grade, sub-grade, interest rate). LendingClub sets these knowing the amount, so they are partly circular.
- **Cleaning does not drop columns for leakage.** Step 4 tags each column with its role; the drop decision belongs to step 7.

### Step 2: load the dataset

**What was done**

- Loaded the full file with `pd.read_csv(DATA_PATH, low_memory=False)` and timed it.
- Inspected it before saying anything about it: `df.info(verbose=True, show_counts=True)`, the first three rows across all 151 columns, `df.describe()` for the numeric columns and for the text columns.
- Checked the rows that are not loans, duplicate ids, the issue-date range and volume by year.

**Why**

- `low_memory=False` makes pandas type each column from the whole column, which avoids mixed-type warnings.
- `verbose=True, show_counts=True` is needed because with this many columns and rows `df.info()` otherwise prints only a summary and hides the non-null counts.

**What we found**

| Measure | Value |
|---|---|
| File on disk | 393 MB (gzip) |
| pandas load time, CPU | 64 to 80 s across runs |
| Shape | 2,260,701 rows x 151 columns |
| In memory | 3.29 GB |
| Column types | 113 numeric (`float64`), 38 text |
| Rows that are not loans | 33 |
| Loans | 2,260,668 |
| Duplicate loan ids | 0 |
| Issue dates | Jun 2007 to Dec 2018 (139 months) |

- **33 summary rows.** They hold a sentence in `id` ("Total amount funded in policy code 1: ...") and nothing else: the totals lines of the quarterly files that were stacked to make this one.
- **Volume by year** grows from 603 loans (2007) to 495,242 (2018). 2015 to 2018 hold 79% of all loans.
- **Target:** \$500 to \$40,000, median \$12,900, mean \$15,047.
- **Largest loan by year:** \$25,000 through 2010, \$35,000 from 2011, \$40,000 from 2016.
- **Snapshot date:** the most common `last_pymnt_d` is Mar-2019, so the file was taken around March 2019.
- **Empty and constant columns:** `member_id` has no values; `policy_code` is always 1; `hardship_type` has one value.
- **Text that needs parsing:** `term`, `emp_length`, `issue_d`, `earliest_cr_line`.
- **Implausible extremes:** `annual_inc` up to 110,000,000, `dti` from -1 to 999, `revol_util` up to 892%.

**Open**

- **GPU timing not done.** The plan was one Colab run timing the same load with cuDF, for a CPU-versus-GPU comparison. Colab needs a Google login, so this is still to do. The CPU number to compare against is 64 to 80 s (it varies from run to run on the same laptop).

**Commit:** `Steps 1-2: dataset introduction, load and first profile`

### Step 3: pre-cleaning insights and visuals

**What was done**

- Looked at the raw data from seven angles, changing nothing. Each one ends with the cleaning action it leads to.
- Saved six charts to `outputs/figures/` (names start with `03_`).
- Built the complexity scorecard (notebook 3.8) from the measured values, which completes section 1.8.

**What we found, and what it means for cleaning**

| # | Looked at | Finding | Cleaning action |
|---|---|---|---|
| 3.1 | Volume over time | 603 loans in 2007, 495,242 in 2018 | Do not delete early rows for lacking later fields |
| 3.2 | Missing values by column | 31.8% of all cells are empty. 58 of 151 columns are more than 30% empty; the other 93 are at most 13.1% empty. Nothing lies between 13% and 38% | Drop columns above 30% |
| 3.3 | Missing values by issue year | Gaps follow the calendar: `tot_cur_bal` and its block start in 2012, `open_acc_6m` and its block of 13 in December 2015, joint fields in late 2015, second-applicant fields in 2017 | Drop the late blocks; they cannot be filled for earlier loans |
| 3.4 | Extreme values | `annual_inc` up to 110,000,000, `dti` up to 999, `revol_util` up to 892% | Cap four columns at the 99.9th percentile |
| 3.4 | Where the broken values come from | All 1,667 zero incomes, all 2,561 DTIs above 100 and all 1,711 missing DTIs are joint applications | Use the joint income and DTI for joint applications |
| 3.5 | Target | 69.9% of loans are an exact multiple of \$1,000, 34.2% of \$5,000; \$10,000 alone is 8.3% of loans. Maximum \$25,000, then \$35,000 from Feb 2011, then \$40,000 from Mar 2016 | None: the target is complete and valid. Keep issue year as a feature |
| 3.6 | Text columns | `term` is `" 36 months"`, `emp_length` is `"10+ years"`, dates are `"Dec-2015"`, `emp_title` has 512,694 distinct values, `home_ownership` has three rare labels (1,232 loans) | Parse to numbers and dates; tidy the labels |
| 3.7 | Empty, constant, redundant | `member_id` is empty; `policy_code`, `hardship_type`, `deferral_term`, `hardship_length` hold one value; `fico_range_high` is always `fico_range_low` + 4 or 5 | Drop them |
| 3.7 | Leakage | `funded_amnt` equals `loan_amnt` in 99.91% of loans; correlations with the target: `funded_amnt` 1.000, `funded_amnt_inv` 0.999, `installment` 0.946. For comparison `annual_inc` 0.197, `fico_range_low` 0.111, `int_rate` 0.098 | Tag, do not drop |

**Why these choices**

- **30% threshold:** the columns fall into two separate sets (at most 13% empty, or at least 38% empty), so any threshold between those gives the same result. The choice is not sensitive.
- **Drop, not fill, the mostly-empty columns:** filling a third or more of a column means inventing data. The blocks added in 2015 are 100% empty for the 38% of loans issued before then.
- **"Months since" columns:** an empty value means the event never happened (for example never delinquent). No number can stand for "never", and the count columns (`delinq_2yrs`, `pub_rec`) carry the same information.
- **Cap, not delete, extremes:** each row is a real loan with a valid target.

**Things the teammate should know for steps 6-9**

- The target is heaped on round numbers and bounded by a cap that moved twice. Both are properties of the problem, not errors.
- The 2015 to 2018 loans are 79% of the file, so overall averages mostly describe those years.

**Commit:** `Step 3: pre-cleaning insights`
