# Project documentation: predicting loan amount for accepted LendingClub applications

Data Visualization class project, Part A. This file records what was done in each step, why, and what was found, so a teammate can pick up the work without asking. The notebook `notebooks/01_data_prep.ipynb` holds the code and its outputs; this file is the summary.

## Status board

| Step | What | Owner | State | Where |
|---|---|---|---|---|
| 1 | Dataset, motivation, complexity | dhruvinn25 | Done | notebook section 1 |
| 2 | Load dataset | dhruvinn25 | Done | notebook section 2 |
| 3 | Pre-cleaning insights and visuals | dhruvinn25 | Done | notebook section 3 |
| 4 | Data cleaning | dhruvinn25 | Done | notebook section 4 |
| 5 | Pre vs post cleaning visuals | dhruvinn25 | Done | notebook section 5 |
| 6 | ML direction | Teammate | Not started | |
| 7 | Feature selection, feature-to-target, correlation heatmap | Teammate | Not started | |
| 8 | Decision table | Teammate | Not started | |
| 9 | Tableau dashboard | Teammate | Not started | |

## How to reproduce

1. Download `accepted_2007_to_2018Q4.csv.gz` from <https://www.kaggle.com/datasets/wordsforthewise/lending-club> (Kaggle login needed).
2. Put it in `data/raw/`. Data files are not in git: they are too large for GitHub's 100 MB limit.
3. `pip install -r requirements.txt` (pandas, pyarrow, numpy, matplotlib, seaborn, jupyter).
4. Open `notebooks/01_data_prep.ipynb` and run all cells from top to bottom.

The whole notebook takes about three minutes on a 16 GB laptop (the load alone is 64 to 84 s). It finishes with `data/processed/loans_clean.parquet` (2,260,668 rows x 90 columns, 275 MB) on disk.

Environment the notebook was last run in: Windows 11, 16 GB RAM, Anaconda Python 3.14, pandas 3.0.3, pyarrow 23.0.1, matplotlib 3.11.0, seaborn 0.13.2. It has not been run on other versions.

## Hand-off notes for steps 6-9

**Start from** `data/processed/loans_clean.parquet` (`pd.read_parquet`) and `outputs/column_tags.csv`.

| | Raw file | Cleaned file |
|---|---|---|
| Rows | 2,260,701 | 2,260,668 (one per loan) |
| Columns | 151 | 90 |
| Column types | 113 numeric, 38 text | 72 numeric, 13 category, 3 date, 2 free text |
| Cells missing | 31.8% | 0.001% (two post-issue date columns) |
| In memory | 3.29 GB | 1.44 GB |

**Pick features with the `role` column of `column_tags.csv`:**

- `application` (61 columns): known when the borrower applies. Feature set A.
- `lc_pricing` (6): set by LendingClub knowing the amount. Feature set B only, and partly circular.
- `leakage` (3) and `post_issue` (18): never model features.
- `target`: `loan_amnt`. `id`: the loan id.

**Know these before using the table:**

- `annual_inc` and `dti` hold the **joint** figures for joint applications and the individual figures otherwise. `application_type` says which.
- Loans issued before mid-2012 (about 3% of rows) have a **median placeholder** in every credit-bureau column that did not exist yet. `issue_year` identifies them.
- `emp_length` is in years and **10 means ten or more**. The 6.5% with no value got the median (6).
- `annual_inc`, `revol_util` and `revol_bal` are **capped** at their 99.9th percentile. `dti` is not capped.
- `emp_title` still has 411,810 distinct values and needs grouping before use. `zip_code` is a three-digit prefix stored as text.
- The **target is untouched**: identical to the raw file loan by loan. It is heaped on round numbers and bounded by a cap that moved in February 2011 and March 2016.
- `last_pymnt_d` and `last_credit_pull_d` have a few empty cells on purpose.

**Still open**

- GPU timing of the load on Colab (see step 2).
- How the parquet file reaches teammates: re-run the notebook, or share the 275 MB file on a drive.
- Roles were assigned from column names; check them against LendingClub's data dictionary before modelling.

## Files

| Path | What it is | In git? |
|---|---|---|
| `notebooks/01_data_prep.ipynb` | Steps 1 to 5, executed, with outputs | Yes |
| `DOCUMENTATION.md` | This file | Yes |
| `outputs/figures/` | Charts saved by the notebook, for slides and the report | Yes |
| `outputs/column_tags.csv` | Every cleaned column with its role, group and type | Yes |
| `data/raw/` | The Kaggle file | No |
| `data/processed/loans_clean.parquet` | The cleaned table, written by notebook section 4 (275 MB) | No: re-create it by running the notebook |

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
| pandas load time, CPU | 64 to 84 s across runs |
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

- **GPU timing not done.** The plan was one Colab run timing the same load with cuDF, for a CPU-versus-GPU comparison. Colab needs a Google login, so this is still to do. The CPU number to compare against is 64 to 84 s (it varies from run to run on the same laptop).

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
| 3.4 | Extreme values | `annual_inc` up to 110,000,000, `dti` up to 999, `revol_util` up to 892% | Cap income, utilisation and revolving balance at the 99.9th percentile; DTI is fixed by the next row |
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

### Step 4: data cleaning

**What was done**

Nine actions, each one traced to an observation in step 3. Rows and columns are logged after every action.

| # | Action | Rows | Columns | Detail |
|---|---|---|---|---|
| 0 | Raw file | 2,260,701 | 151 | |
| 1 | Drop summary rows | 2,260,668 | 151 | 33 rows removed; no duplicate ids; `id` converted from text to integer |
| 2 | Joint income and DTI for joint loans | 2,260,668 | 151 | 120,710 joint applications now carry the joint figures |
| 3 | Drop empty and constant columns | 2,260,668 | 146 | `member_id`, `policy_code`, `hardship_type`, `deferral_term`, `hardship_length` |
| 4 | Drop columns over 30% missing | 2,260,668 | 92 | 54 columns |
| 5 | Drop redundant columns | 2,260,668 | 89 | `url`, `title`, `fico_range_high` |
| 6 | Parse text; derive 2 columns | 2,260,668 | 90 | `issue_year` and `credit_history_years` added, `earliest_cr_line` removed |
| 7 | Fill missing values | 2,260,668 | 90 | 2,998,961 cells in 54 columns |
| 8 | Cap 3 columns at the 99.9th percentile | 2,260,668 | 90 | 6,771 values capped |
| 9 | Tag columns; store labels as categories | 2,260,668 | 90 | roles written to `outputs/column_tags.csv` |

The cleaned table is written to `data/processed/loans_clean.parquet`.

**Detail and reasons, action by action**

- **1. Summary rows.** They are totals lines, not loans.
- **2. Joint applications.** LendingClub assessed joint applications on the two borrowers together. The individual fields are then sometimes 0, 999 or empty. Replacing them with the joint value fixed all 1,667 zero incomes, all 2,561 DTIs above 100 and all 1,711 missing DTIs. One negative DTI on an individual application was set to missing and then filled with the median in action 7. After this step `annual_inc` and `dti` mean "the figure the application was assessed on".
- **3 and 4. Columns dropped.** The 54 dropped for missing values are: 19 from the credit bureau file (the 2015-onward block and the mostly-empty "months since" columns), all 16 joint and second-applicant columns, 11 hardship and 6 settlement detail columns, `desc`, and `next_pymnt_d`. The most-missing column that is kept is `mths_since_recent_inq` at 13.1%.
- **5. Redundant columns.** `url` is the id inside a web address, `title` repeats `purpose` as free text, `fico_range_high` is `fico_range_low` plus 4 or 5.
- **6. Parsing.**
  - `term`: `" 36 months"` to 36.
  - `emp_length`: `"< 1 year"` to 0, up to `"10+ years"` to 10. So 10 means ten or more.
  - `issue_d`, `last_pymnt_d`, `last_credit_pull_d`: text to dates.
  - `issue_year`: new, from `issue_d`.
  - `credit_history_years`: new, the time from `earliest_cr_line` to `issue_d`. Range 0.5 to 83.3 years, median 14.8.
  - `emp_title`: trimmed and lower-cased, 512,694 distinct values down to 411,810.
  - `home_ownership`: `ANY` and `NONE` merged into `OTHER` (1,232 loans).
- **7. Missing values.** Numbers get the column median (not the mean, because the columns are skewed). Text gets the label `unknown`. Dates are left empty. The largest fills: `mths_since_recent_inq` 13.1% of loans, `emp_title` 7.4%, `num_tl_120dpd_2m` 6.8%, `emp_length` 6.5% (median 6 years), `mo_sin_old_il_acct` 6.2%, then about 30 credit-bureau columns at 2 to 3%.
- **8. Capping.**

  | Column | Max before | Cap | Loans capped |
  |---|---|---|---|
  | `annual_inc` | 110,000,000 | 611,024 | 2,261 |
  | `revol_util` | 892.3 | 102.1 | 2,249 |
  | `revol_bal` | 2,904,836 | 265,894 | 2,261 |

  `dti` is **not** capped. After action 2 it runs from 0 to 69.5, and those top values are real joint DTIs. A first version of the notebook did cap it at the 99.9th percentile (40.7); that was removed because it cut into genuine values.
- **9. Roles.**

  | Role | Columns | Meaning |
  |---|---|---|
  | `application` | 61 | Known when the borrower applies. Feature set A |
  | `post_issue` | 18 | Only exists after the loan is issued. Never a feature |
  | `lc_pricing` | 6 | `term`, `int_rate`, `grade`, `sub_grade`, `initial_list_status`, `disbursement_method`. Set by LendingClub knowing the amount. Feature set B only |
  | `leakage` | 3 | `funded_amnt`, `funded_amnt_inv`, `installment`. Never a feature |
  | `id` | 1 | |
  | `target` | 1 | `loan_amnt` |

**Checks built into the notebook**

- Every raw column is in exactly one group and every cleaned column has exactly one role. The notebook stops if not.
- `emp_length` parsing stops if it meets a label that is not in the mapping.
- The row count after cleaning equals the raw count minus the 33 summary rows.
- The only columns with empty cells after cleaning are `last_pymnt_d` (2,427) and `last_credit_pull_d` (72), left empty on purpose.

**Limits of this cleaning (open for step 7)**

- **Medians are placeholders for early loans.** About 3% of loans (issued before mid-2012) have a median in every credit-bureau column that was not recorded yet.
- **`mths_since_recent_inq`:** an empty value means no recent inquiry, so the median (5 months) understates how clean those borrowers are.
- **`emp_length`:** the 6.5% with no value probably gave no employer. They get the median, which hides that. `emp_title == "unknown"` still marks most of them.
- **Roles are assigned from column names and what the columns hold.** They were not checked field by field against LendingClub's data dictionary; worth a look before modelling.
- **Only three columns are capped.** Other skewed columns (for example `tot_cur_bal`, `tot_coll_amt`) are as recorded.

**Commit:** `Step 4: cleaning pipeline`

### Step 5: before and after cleaning

**What was done**

- Reloaded the saved parquet file and drew every "after" from it, not from the table in memory, so the charts show what a teammate will load.
- Compared the table before and after on size, types, missing values and extreme values, and drew the cleaning funnel.
- Saved three charts to `outputs/figures/` (names start with `05_`).
- Checked that the target did not change.

**What we found**

| Measure | Before | After |
|---|---|---|
| Rows | 2,260,701 | 2,260,668 |
| Columns | 151 | 90 |
| Numeric columns | 113 | 72 |
| Date columns | 0 | 3 |
| Category columns | 0 | 13 |
| Free-text columns | 38 | 2 |
| Columns with missing values | 113 | 2 |
| Cells missing | 31.8% | 0.001% |
| In memory | 3.29 GB | 1.44 GB |
| File on disk | 393 MB (csv.gz) | 275 MB (parquet) |

- **Missing values** (`05_missing_before_after.png`): complete columns go from 38 to 88. Before, 58 columns were more than 30% empty; after, none is.

  | Share missing | Columns before | Columns after |
  |---|---|---|
  | none | 38 | 88 |
  | up to 15% | 55 | 2 |
  | 15% to 50% | 14 | 0 |
  | 50% to 90% | 6 | 0 |
  | over 90% | 38 | 0 |

- **Extreme values** (`05_outliers_before_after.png`): before, each box plot is squashed flat by a few extreme points. After, the box, median and whiskers are readable.

  | Column | Max before | Max after | Median before | Median after |
  |---|---|---|---|---|
  | `annual_inc` | 110,000,000 | 611,024 | 65,000 | 68,500 |
  | `dti` | 999 | 69.5 | 17.8 | 17.7 |
  | `revol_util` | 892.3 | 102.1 | 50.3 | 50.3 |
  | `revol_bal` | 2,904,836 | 265,894 | 11,324 | 11,324 |

  The medians barely move: the cleaning changed the extremes and left the bulk of the data alone. The `annual_inc` median rises because joint applications now carry the combined income of both borrowers.
- **Cleaning funnel** (`05_cleaning_funnel.png`): columns go 151, 146, 92, 89, 90. The big drop is the mostly-empty columns. Rows change once, by 33.
- **Target:** `loan_amnt` is identical before and after, loan by loan.

**Verification**

- The notebook was run from top to bottom in one pass with no errors or warnings, and it is committed with those outputs.
- A separate check in a fresh Python process, reading the saved parquet file and comparing it with the raw data, confirmed:
  - 2,260,668 rows and 90 columns, unique ids, the same loans in the same order as the raw file;
  - the target identical to the raw file;
  - empty cells only in the two post-issue date columns;
  - `term`, `issue_d` and `issue_year` agree with the raw text;
  - `annual_inc` and `dti` equal the joint-or-individual value, median-filled, with income capped and DTI not;
  - `column_tags.csv` lists exactly the cleaned columns, each with a role.

**Commit:** `Step 5: before/after cleaning visuals`
