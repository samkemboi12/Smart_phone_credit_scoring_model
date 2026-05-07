# Credit Scoring Model Roadmap 2

## Objective

Build a practical and reliable credit scoring model for smartphone loans that classifies applicants into:

- `Low Risk`
- `Medium Risk`
- `High Risk`

The model must work for:

- existing historical loans for training
- new applicants at approval stage for prediction

The main goal is to keep the workflow simple, organized, and team-friendly so that most effort goes into training a good model.

## Team Roles

To keep the work clear and fast, the project should be split into focused roles.

### 1. Data Loading and SQL Structuring Lead

Responsibility:

- load raw files into SQL
- create the base SQL layers
- ensure tables are stable and easy for the rest of the team to use

Main tasks:

- load quarterly credit CSVs into raw SQL tables
- load sales and customer workbook sheets into raw SQL tables
- load NPS into raw SQL tables for reference only
- standardize basic column names
- preserve raw data without changing meaning
- create the cleaned modeling tables required by the next team members

Expected output:

- `raw` layer
- `clean` layer
- `model_input` layer

### 2. Data Cleaning Lead

Responsibility:

- clean and validate the usable data
- remove duplicates correctly
- resolve messy values and missing values

Main tasks:

- remove blank rows and padded Excel rows
- enforce `loan_id` join consistency
- standardize date formats
- standardize numeric fields
- clean category values such as seller, gender, sale type, and product names
- identify usable vs non-usable columns
- ensure one clean origination record per loan for the approval model

Expected output:

- cleaned credit table
- cleaned sales/customer table
- column usability report

### 3. Feature Engineering Lead

Responsibility:

- create model-ready variables
- prepare the two final datasets

Main tasks:

- derive origination features for approval scoring
- derive outcome label from future loan behavior
- build grouped categorical variables
- create affordability and income proxy features
- add missingness flags where useful
- build the two final datasets

Expected output:

- `approval_model_dataset`
- `behavioral_model_dataset`
- feature dictionary

### 4. Modeling and Training Lead

Responsibility:

- train and compare candidate models
- choose the most stable and useful model

Main tasks:

- split train, validation, and test by time
- train baseline models
- compare performance
- tune the best model
- convert probabilities into `Low`, `Medium`, and `High` risk bands

Expected output:

- final selected model
- model performance report
- score band thresholds

### 5. Validation and Business Logic Lead

Responsibility:

- check model logic
- ensure input variables make sense for new applicants
- review model fairness and practical use

Main tasks:

- confirm no target leakage
- confirm approval model uses only origination variables
- validate bad definition
- review special statuses like `Return`, `FPD`, `FMD`, and `Write Off`
- review model outputs with business team

Expected output:

- model validation checklist
- approved target logic
- approved risk band interpretation

### 6. Project Coordinator

Responsibility:

- track handoffs across the team
- keep the project moving with minimal complexity

Main tasks:

- manage sequencing
- track blockers
- confirm when each layer is ready
- keep documentation updated

## Recommended Simple Data Architecture

To avoid overcomplicating the process, use just three SQL layers.

### Layer 1: `raw`

Purpose:

- load files exactly as they come

Tables:

- `raw.credit_snapshot`
- `raw.sales_details`
- `raw.customer_dob`
- `raw.customer_gender`
- `raw.customer_income`
- `raw.nps_responses`

Rules:

- no business logic
- no deduplication
- keep source file and load timestamp

### Layer 2: `clean`

Purpose:

- clean and standardize all sources
- remove obvious noise
- make tables joinable and usable

Tables:

- `clean.credit_snapshot`
- `clean.sales_customer_master`
- `clean.nps_responses`

Rules:

- standardize names and formats
- keep only usable rows
- clean categorical values
- cast dates and numerics
- remove duplicates at source-table level where needed

### Layer 3: `model_input`

Purpose:

- prepare the final datasets for modeling

Tables:

- `model_input.approval_model_dataset`
- `model_input.behavioral_model_dataset`

Rules:

- all usable columns brought together
- fully joined and feature-ready
- no unnecessary extra transformation outside this layer except model-specific encoding

This is the simplest structure that still keeps the work organized.

## Two Final Datasets

We should prepare two complete datasets from SQL so that the modeling team can move straight into training.

### 1. Approval Model Dataset

Purpose:

- predict whether a client is likely to default before or at loan approval

Grain:

- one row per `loan_id`

Important rule:

- use only variables known at origination

Main contents:

- cleaned sales variables
- cleaned customer variables
- origination contract variables
- engineered origination features
- final target label created from future performance

This is the main dataset for the credit scoring model.

### 2. Behavioral Model Dataset

Purpose:

- monitor existing active loans after disbursement

Grain:

- one row per `loan_id` per observation date

Main contents:

- repayment behavior
- delinquency measures
- loan progression metrics
- current arrears state

This dataset is useful later, but the approval model remains the main priority.

## Key Logic On Historical Credit Data

This remains very important.

The credit data is historical and cumulative across quarters, so the same loan appears many times.

That means:

- January loans can also appear in March, June, September, and December
- those are not duplicates to delete blindly
- they are repeated observations of the same loan through time

Correct treatment:

- use the earliest valid origination row as the feature record
- use later snapshots only to determine whether the loan became bad
- keep one final approval-model row per `loan_id`

For the approval model:

- `1 loan_id = 1 row`

For the behavioral model:

- `1 loan_id + snapshot_date = 1 row`

This is how we avoid duplicate customers while still using historical performance properly.

## Recommended Columns For The Approval Model

The approval model should only use columns available for new applicants.

### Keep if usable

#### Credit and contract variables

- `loan_id`
- `sale_date`
- `deposit`
- `weekly_rate`
- `credit_expiry`
- `credit_check_done`
- `advance`
- `initial_pay` if confirmed as origination-time

#### Sales variables

- `sale_id`
- `sale_type`
- `seller`
- `seller_type`
- `return_policy_compliance`
- `cash_price`
- `loan_price`
- `loan_term`
- `product_name`
- `model`
- `business_model`

#### Customer variables

- `date_of_birth`
- derived applicant age
- `gender`
- `citizenship` if usable

#### Income variables

- `received`
- `duration`
- `persons_received_from_total`
- `banks_received`
- `paybills_received_others`

#### Engineered variables

- `deposit_ratio`
- `weekly_rate_to_price`
- `price_delta`
- `device_tier`
- `sales_channel_group`
- `income_band`
- `age_band`
- missingness flags

### Do not use in the approval model

- `days_past_due`
- `arrears`
- `balance`
- `closing_balance`
- `total_paid`
- `total_due_today`
- `payment_amount`
- `adjustment_amount`
- `prepayment_amount`
- `account_status_l1`
- `account_status_l2`
- `delinquency_bucket`
- `current_flag`
- any NPS variables

These fields are useful for outcome creation or behavioral scoring, but not for scoring a brand-new client.

## Target Definition

For the approval model, define a bad loan using future repayment outcome.

Recommended definition:

- `Bad = 1` if the loan ever reaches `30+ days past due` within `180 days` of origination
- `Bad = 1` if it enters severe states like `FPD`, `FMD`, or confirmed default/write-off status in that same window
- `Bad = 0` otherwise

This target is simple, business-friendly, and suitable for training.

## What Each Person Should Deliver

### Data Loading and SQL Structuring Lead

Deliver:

- raw SQL tables loaded
- clean layer table skeletons
- source-to-table mapping document

### Data Cleaning Lead

Deliver:

- cleaned `credit_snapshot`
- cleaned `sales_customer_master`
- row filtering logic
- duplicate handling logic
- missingness summary

### Feature Engineering Lead

Deliver:

- approval model dataset
- behavioral model dataset
- feature list with definitions
- target creation SQL logic

### Modeling and Training Lead

Deliver:

- train/validation/test split
- benchmark model results
- selected final model
- risk band thresholds

### Validation and Business Logic Lead

Deliver:

- leakage check
- variable applicability check for new members
- target approval note
- status-mapping note for ambiguous outcomes

## Recommended Project Sequence

To keep it simple, follow this order:

1. Load all raw data into SQL.
2. Build the `clean` layer.
3. Merge all usable origination variables into one approval-model dataset.
4. Build the future-performance target from historical credit snapshots.
5. Build the behavioral dataset separately.
6. Train the approval model first.
7. Validate the selected model.
8. Convert model scores into `Low`, `Medium`, and `High` risk bands.

## Final Recommendation

The cleanest and least complicated setup is:

- one SQL layer to load raw data
- one SQL layer to clean and standardize
- one SQL layer to prepare the two final modeling datasets

And the team should work in these roles:

- loading and SQL structuring
- cleaning
- feature engineering
- modeling and training
- validation and business logic
- coordination

This keeps everyone focused, reduces confusion, and helps the team move quickly toward the real goal:

- a working and reliable credit scoring model
