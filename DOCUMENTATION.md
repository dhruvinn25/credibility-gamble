# Project documentation: predicting loan amount for accepted LendingClub applications

Data Visualization class project, Part A. This file records what was done in each step, why, and what was found, so a teammate can pick up the work without asking. The notebook `notebooks/01_data_prep.ipynb` holds the code and its outputs; this file is the summary.

## Status board

| Step | What | Owner | State | Where |
|---|---|---|---|---|
| 1 | Dataset, motivation, complexity | dhruvinn25 | Done | notebook section 1 |
| 2 | Load dataset | dhruvinn25 | Done | notebook section 2 |
| 3 | Pre-cleaning insights and visuals | dhruvinn25 | Not started | notebook section 3 |
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
| pandas load time, CPU | 64 s |
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

- **GPU timing not done.** The plan was one Colab run timing the same load with cuDF, for a CPU-versus-GPU comparison. Colab needs a Google login, so this is still to do. The CPU number to compare against is 64 s.

**Commit:** `Steps 1-2: dataset introduction, load and first profile`
