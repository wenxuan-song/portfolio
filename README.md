# Logic Walkthrough: Semantic Similarity Matching Pipeline

## Overview
This pipeline processes research data to find semantically similar examples for each opinion/answer choice. It uses sentence embeddings to match text from specific pairids with similar examples from a larger dataset.

---

## Required Files & Directory Structure

### Input Files Needed:

1. **Source Data Directory**: `./pre-data/origin data/`
   - Contains original CSV files: `b2a.csv`, `b15f.csv`, `b27c.csv`, `b45a.csv`, `r2a.csv`, `r12f.csv`, `r12f.csv`, `r16c.csv`, `r50b.csv`, `p23a.csv`, `p40b.csv`
   - Each CSV has columns: `pairid`, `report_type`, text columns (like `text3b_protocol_access`), and opinion columns (like `r2a` or `b27c___1`, `b27c___2`, etc.)

2. **Question Info File**: `quesiton_info_updated.csv`
   - Contains metadata about each question file
   - Has columns: `choices_index`, `choices_text`
   - Used to extract opinion numbers (1, 2, 3, etc.) for each file

---

## Step-by-Step Logic Flow

### **PHASE 1: Data Filtering (Cells 0-2)**

#### Cell 0: Setup
- **Purpose**: Define configuration
- **What it does**:
  - Sets up paths and directories
  - Defines which pairids to exclude: `[11, 45, 61, 71, 100]`
  - Defines filtering rules based on filename prefix

#### Cell 1: Create "Final Data" (Main Dataset)
- **Input**: `./pre-data/origin data/*.csv`
- **Output**: `./pre-data/final data/*.csv`
- **Logic**:
  1. Reads each CSV file from origin data
  2. **Step 1**: Removes rows where `pairid` is in exclusion list (11, 45, 61, 71, 100)
  3. **Step 2**: Filters by `report_type` based on filename:
     - Files starting with `r` (e.g., `r2a.csv`) → Keep only `report_type = 2` (results)
     - Files starting with `p` (e.g., `p23a.csv`) → Keep only `report_type = 1` (protocol)
     - Files starting with `b` (e.g., `b2a.csv`) → Keep all rows (both protocol and results)
  4. Saves filtered data to `final data` directory

**Example**:
- `r2a.csv` originally has 200 rows (100 protocol + 100 results)
- After removing pairids 11, 45, 61, 71, 100: ~190 rows
- After filtering for results only: ~95 rows (only `report_type = 2`)

#### Cell 2: Create "Pairid Filtered Data" (Query Dataset)
- **Input**: `./pre-data/origin data/*.csv`
- **Output**: `./pre-data/pairid filtered data/*.csv`
- **Logic**:
  1. Reads each CSV file from origin data
  2. **Step 1**: Keeps ONLY rows where `pairid` is in `[11, 45, 61, 71, 100]` (opposite of Cell 1)
  3. **Step 2**: Applies same `report_type` filtering as Cell 1
  4. Saves to `pairid filtered data` directory

**Purpose**: This creates a small dataset of specific pairids that we want to find examples for.

---

### **PHASE 2: Semantic Matching Setup (Cells 3-5)**

#### Cell 3: Load Embedding Model
- **Purpose**: Initialize the semantic similarity model
- **What it does**:
  - Loads `sentence-transformers/all-MiniLM-L6-v2` model
  - This model converts text into numerical vectors (embeddings)
  - Sets up directories for results

#### Cell 4: Display Opinion Numbers
- **Input**: `quesiton_info_updated.csv`
- **Purpose**: Preview what opinion numbers exist for each file
- **Logic**:
  - Reads question info file
  - For each file (b2a, r2a, etc.), extracts opinion numbers from `choices_text`
  - Example: `choices_text = "['1, Yes', '2, No', '3, Not reported']"` → extracts `[1, 2, 3]`

#### Cell 5: Create File Opinions Dictionary
- **Input**: `quesiton_info_updated.csv`
- **Output**: `file_opinions` dictionary
- **Purpose**: Store opinion numbers for each file
- **Example**:
  ```python
  file_opinions = {
      'r2a': [1, 2],
      'b2a': [1, 2, 3],
      'b27c': [1, 2, 3, 4, 5, 6, 7, 8],
      ...
  }
  ```

---

### **PHASE 3: Semantic Matching (Cells 6-8)**

#### Cell 6: Define Helper Functions
- **Purpose**: Create reusable functions for the matching process
- **Functions**:
  1. `get_file_opinions_dict()` - Loads opinion numbers for each file
  2. `find_text_column()` - Finds the text column in a dataframe
  3. `check_multilabel()` - Determines if question is multilabel (b27c___1, b27c___2) or single-choice (r2a)
  4. `find_best_match_for_opinion()` - Finds best matching example for a specific opinion number
  5. `process_single_query()` - Processes one query (one pairid + report_type combination)

#### Cell 7: Main Processing Loop
- **Input Files**:
  - `./pre-data/pairid filtered data/*.csv` (query files)
  - `./pre-data/final data/*.csv` (search dataset)
- **Purpose**: Find best matching examples for each opinion
- **Logic Flow**:

  **For each pairid file (e.g., `r2a.csv`):**
  
  1. **Load Data**:
     - Read pairid filtered file (small query dataset)
     - Read corresponding final data file (large search dataset)
  
  2. **Identify Columns**:
     - Find text column (e.g., `text3b_protocol_access`)
     - Check if multilabel (has columns like `b27c___1`) or single-choice (has column like `r2a`)
     - Get opinion numbers for this file (e.g., `[1, 2]` for r2a)
  
  3. **Create Embeddings**:
     - Convert all texts in final data to embeddings (for efficiency)
     - This creates numerical vectors representing the meaning of each text
  
  4. **For each row in pairid filtered data** (one query):
     - Get the text from this row (e.g., text from pairid 11, report_type 2)
     - Convert this text to an embedding
     - Calculate cosine similarity with all texts in final data
     - **For each opinion number** (e.g., 1, 2):
       - Filter final data to only rows with this opinion
       - Find the row with highest similarity score
       - Store the match result
  
  5. **Store Results**:
     - Each result contains:
       - Query info: pairid, report_type, text
       - Matched info: matched pairid, matched text, similarity score
       - Opinion number that was matched

#### Cell 8: Save Results
- **Output**: `semantic results/examples_by_opinion_per_pairid.csv`
- **Purpose**: Save all matches to CSV and display summary
- **What it saves**:
  - For each query (pairid + report_type), for each opinion number, the best matching example

---

### **PHASE 4: Format Results (Cell 9)**

#### Cell 9: Create Formatted Text Files
- **Input**: `semantic results/examples_by_opinion_per_pairid.csv`
- **Output**: `semantic results/formatted_examples_*.txt` (one file per CSV)
- **Purpose**: Create human-readable formatted files
- **Format**:
  ```
  File: r2a.csv
  ================================================================================
  
  Pairid 11, Report Type 2:
  --------------------------------------------------------------------------------
  Example 1:
  Relevant Text: "...text content..."
  Answer: [1]
  
  Example 2:
  Relevant Text: "...text content..."
  Answer: [2]
  ```

---

## Key Concepts

### 1. **Pairid Filtering**
- **Final Data**: Contains most pairids EXCEPT 11, 45, 61, 71, 100 (used as search dataset)
- **Pairid Filtered Data**: Contains ONLY pairids 11, 45, 61, 71, 100 (used as queries)

### 2. **Report Type Filtering**
- `report_type = 1`: Protocol section of research papers
- `report_type = 2`: Results section of research papers
- Files prefixed with `r`/`p` filter to one type; `b` files keep both

### 3. **Multilabel vs Single-Choice**
- **Single-Choice** (e.g., `r2a.csv`): Has one column `r2a` with values 1, 2, 3...
- **Multilabel** (e.g., `b27c.csv`): Has multiple columns `b27c___1`, `b27c___2`, etc., each can be 0 or 1

### 4. **Semantic Similarity**
- Uses sentence embeddings to convert text to numerical vectors
- Cosine similarity measures how similar two texts are (0-1 scale, higher = more similar)
- Finds the most semantically similar example for each opinion number

---

## Data Flow Summary

```
Origin Data (all pairids)
    ↓
    ├─→ Final Data (exclude pairids 11,45,61,71,100) → Used as SEARCH dataset
    └─→ Pairid Filtered Data (only pairids 11,45,61,71,100) → Used as QUERY dataset
         ↓
         For each query text:
             ↓
             Convert to embedding
             ↓
             Compare with all texts in Final Data
             ↓
             For each opinion number:
                 Find best match with that opinion
                 ↓
                 Store result
         ↓
Results CSV → Formatted Text Files
```

---

## Example Walkthrough

**Scenario**: Finding examples for `r2a.csv`, pairid 11, report_type 2

1. **Load Data**:
   - Query: Read `pairid filtered data/r2a.csv`, find row with pairid=11, report_type=2
   - Search: Read `final data/r2a.csv` (has ~95 rows)

2. **Get Opinion Numbers**:
   - From `file_opinions['r2a']` → `[1, 2]`

3. **Process Query**:
   - Query text: "The data will be made available upon request..."
   - Convert to embedding vector

4. **Find Matches**:
   - For opinion 1: Find row in final data where `r2a = 1` with highest similarity
   - For opinion 2: Find row in final data where `r2a = 2` with highest similarity

5. **Store Results**:
   - Opinion 1: Matched pairid 17, similarity 0.5872
   - Opinion 2: Matched pairid 63, similarity 0.8344

---

## File Dependencies

```
Required:
├── pre-data/
│   ├── origin data/
│   │   ├── b2a.csv
│   │   ├── r2a.csv
│   │   └── ... (all CSV files)
│   ├── final data/ (created by Cell 1)
│   └── pairid filtered data/ (created by Cell 2)
├── quesiton_info_updated.csv
└── filter.ipynb

Generated:
└── semantic results/
    ├── examples_by_opinion_per_pairid.csv (created by Cell 8)
    └── formatted_examples_*.txt (created by Cell 9)
```

