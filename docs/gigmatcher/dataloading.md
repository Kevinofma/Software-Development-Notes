# Overview of How Plinker Data is Prepped for Training

The entire process for how PLINDER data is prepped and transformed into the dataset used for training is managed by scripts in the `gigmatcher/scripts/data_prep/` directory

### 1. **Create Data Splits (`make_plinder_split_parquet.py`)**
This is the first step, which defines the train, validation, and test sets.

1. **Query PLINDER**: It queries the full PLINDER database to get a list of all protein-ligand systems.

2. **Filter Entries**: It filters this list, removing entries with fewer than a minimum number of interacting residues (e.g., 2).

3. **Get Clusters**: It loads PLINDER's pre-computed cluster dataframes (`val_cluster_df`, `test_cluster_df`) to assign a cluster label to each system. This ensures that structurally similar proteins are kept in the same split.

4. **Remove Benchmarks**: It has options to identify systems related to common benchmarks (like `remove_matcher_benchmark`) and move them from the training set to the validation set to prevent data leakage and ensure fair evaluation.

5. **Output**: It saves separate `_train.pq`, `_val.pq`, and `_test.pq` parquet files (and a `.json` file) that list the `system_ids` and cluster information for each split.

### 2. **Create Raw Data Directories (`make_plinder_data_dir.py`)**
This script transforms the parquet files (lists of systems) into the raw file structure needed for processing.

1. **Read Splits**: It reads one of the parquet files from Step 1 (e.g., `plinder_splits_train_pq.gz`).

2. **Create Folders**: For each `system_id` in the file, it creates a subdirectory (e.g., `/path/to/plinder_train_raw/<system_id>/`).

3. **Copy/Convert Files**:
    - It copies the system's receptor PDB file from the PLINDER database into its new folder as `<system_id>_protein.pdb`.
    - It finds all associated ligand `.sdf` files for that system, combines them, and converts them into a single `<system_id>_ligand.mol2` file using `obabel`.

### 3. **Preprocess Raw Data (`preprocess_plinder_data.py`)**
This is the main transformation step. It takes the raw `_protein.pdb` and `_ligand.mol2` files and generates the final training data.

1. **Combine Protein & Ligand**: It first combines the `<system_id>_protein.pdb` and `<system_id>_ligand.mol2` files into a single `_protein_ligand.pdb` file.

2. **Load & Idealize**: The script loads this combined PDB into a Protein object. It then idealizes the protein's sidechains by calculating their chi angles and rebuilding them from standard bond lengths/angles. This ensures a clean, standardized geometry.

3. **Run Reduce**: It runs the Reduce tool on the idealized protein-ligand structure. This adds and optimizes hydrogen atoms. It uses a custom connectivity database (`_reduce_conect.db`) generated from the `.mol2` file to correctly handle the ligand.

4. **Extract GIGs**:
    - **All Components**: It scans the final, reduced protein and extracts all possible GIG components (all "NH", "CO", "OH", "COO", etc. interacting pairs) and saves them as `<system_id>_gig-comps.npz`. This file represents the entire "GIG library" available in that one protein.
    - **Ligand GIG**: It separately analyzes the protein-ligand interface and extracts the specific GIG representing the functional site (i.e., the components interacting with the ligand). This is saved as `<system_id>_ligand-gig.npz`.

5. **Save Processed Protein**: Finally, it saves the processed protein (with idealized sidechains and added hydrogens) as `<system_id>_protein.npz`, which stores its atomic coordinates, masks, and sequence information in NumPy arrays.

### 4. **(Optional) Pre-generate GIGs (`pregenerate_gigs.py`)**
This step is for efficiency. Instead of sampling random GIGs from `_gig-comps.npz` during training, this script does it ahead of time.

1. **Load Data**: It loads the `_gig-comps.npz` (all components) and `_protein.npz` files.

2. **Sample GIGs**: It generates many (e.g., 250) random GIGs for the protein by sampling from its full component set, using various filters like max distance and component type biases (e.g., centrality, amino acid type).

3. **Output**: It saves an array of the indices for these pre-sampled GIGs into a `<system_id>_gig-comp-indices.npy` file.

At the end of this pipeline, a single PLINDER system is transformed into a set of `.npz` and `.npy` files that contain the processed protein scaffold, its full component library, its specific ligand-binding GIG, and (optionally) a list of pre-sampled GIGs for training.

## Understanding `make_plpinder_split_parquet.py`

The `main` function's job is to query the entire PLINDER database, clean and filter all the protein-ligand systems, and then carefully divide them into **train**, **validation**, and **test** sets. 

Its main goal is to prevent **data leakage**, which means ensuring that proteins in the training set are not structurally similar to proteins in the validation or test sets. It does this by using pre-computed "clusters" (groups of similar proteins).

1. Gets the start time to track how long the main functions runs for
```python
def main(
    outfile_prefix: str, 
    use_gz: bool, 
    min_interacting_res: int, 
    remove_matcher_benchmark: bool, 
    remove_rfd2_benchmark: bool
) -> None:
    t0 = time.time()

    ### Rest of the code...

    print(f'Saved final datasets to {outfiles} in {time.time() - t0:.3f} sec')

    ### More code...

```

2. Load all data by calling `load_full_plinder_df()`. This function queries the PLINDER database and loads all protein-ligand systems into a giant pandas DataFrame. It filters this list, keeping only systems that have at least `min_interacting_res` (e.g., 2) residues interacting with the ligand.
```python
# Read full dataset with specific columns
    cols_of_interest = [
        "system_id", 
        "entry_pdb_id", 
        "ligand_ccd_code", 
        "ligand_plip_type", 
        "ligand_interacting_residues", 
        "ligand_interactions"
    ]
    full_df = load_full_plinder_df(cols_of_interest, min_interacting_res)
```

    !!! info "load_full_plinder_df()"
        ```python
        def load_full_plinder_df(cols: list[str], min_interacting_res: int, drop_duplicates: bool = True) -> pd.DataFrame:
            # Read full dataset with specific columns
            full_df = query_index(columns=cols, splits=["*"])

            # Filter out entries with < min_interacting_res interacting residues
            entry_mask = [len(row[1]['ligand_interacting_residues']) >= min_interacting_res for row in full_df.iterrows()]
            if drop_duplicates:
                return full_df[entry_mask].drop_duplicates("system_id")
            else:
                return full_df[entry_mask]
        ```
        This function's purpose is to query the main PLINDER database and then clean that data based on two criteria: the number of interacting residues and whether to remove duplicates.

        -   `drop_duplicates: bool = True`: A boolean (True/False) flag. If it's `True`, the function will remove duplicate entries. It's set to `True` by default.

        #### 1. First the function reads the database and makes a `dataframe`

        ```python
        full_df = query_index(columns=cols, splits=["*"])
        ```

        -   `query_index` function is imported from PLINDER
        -   `columns=cols`: It tells the query to only fetch the columns specified in the cols list that was passed into the function.

        -   `splits=["*"]`: This is a wildcard, telling the query to fetch data from all splits (train, val, test, etc.) in the database.

        #### 2. Second the function filters entries

        ```python
        entry_mask = [len(row[1]['ligand_interacting_residues']) >= min_interacting_res for row in full_df.iterrows()]
        ```

        -   This line builds a "mask," which is a list of `True` and `False` values
        -   It loops through every row in the dataframe and it gets the value from the `ligand_interacting_residues` column and calculates its length (i.e., how many interacting residues there are).
        -   It checks if that number is greater than or equal to the `min_interacting_res` value and the entry_mask will mark which rows to keep(`True`) and which to discard (`False`).

        #### 3. Return the cleaned data

        ```python
        if drop_duplicates:
            return full_df[entry_mask].drop_duplicates("system_id")
        else:
            return full_df[entry_mask]
        ```

        -   returns the cleaned data depending on the `drop_duplicates` flag