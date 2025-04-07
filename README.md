# Compare-files-for-Functional-testing


<pre>
<code>


import os
import pandas as pd

# Input folders
before_folder = 'before'
after_folder = 'after'

# Output containers
match_report = []
mismatch_report = []
extra_rows_info = []

def compare_csv_files(file_name):
    file_before = os.path.join(before_folder, file_name)
    file_after = os.path.join(after_folder, file_name)

    try:
        df_before = pd.read_csv(file_before, dtype=str).fillna('NaN')
        df_after = pd.read_csv(file_after, dtype=str).fillna('NaN')
    except Exception as e:
        mismatch_report.append([file_name, "N/A", "N/A", f"File read error: {e}"])
        return

    # Check columns match exactly
    cols_before = list(df_before.columns)
    cols_after = list(df_after.columns)

    if cols_before != cols_after:
        extra_in_after = set(cols_after) - set(cols_before)
        if extra_in_after:
            for col in extra_in_after:
                mismatch_report.append([file_name, "N/A", col, "Extra column in AFTER"])
        else:
            mismatch_report.append([file_name, "N/A", "N/A", "Column order/name mismatch"])
        return

    # Check row count
    if len(df_after) > len(df_before):
        mismatch_report.append([file_name, "N/A", "N/A", "AFTER has more rows than BEFORE"])
        return

    # Compare row by row
    has_mismatch = False
    for idx in range(len(df_after)):
        row_before = df_before.iloc[idx]
        row_after = df_after.iloc[idx]
        for col in cols_before:
            val_before = row_before[col]
            val_after = row_after[col]

            if val_before == 'NaN' and val_after == 'NaN':
                continue
            elif val_before == 'NaN' and val_after != 'NaN':
                mismatch_report.append([file_name, idx, col, "Before is NULL, After is not"])
                has_mismatch = True
            elif val_before != 'NaN' and val_after == 'NaN':
                mismatch_report.append([file_name, idx, col, "After is NULL, Before is not"])
                has_mismatch = True
            elif val_before != val_after:
                mismatch_report.append([file_name, idx, col, f"Value mismatch | Before: {val_before} | After: {val_after}"])
                has_mismatch = True

    # Handle extra rows in BEFORE
    extra_rows = list(range(len(df_after), len(df_before)))
    if extra_rows:
        extra_rows_info.append(f"File: {file_name}\nExtra row numbers in BEFORE not present in AFTER: {extra_rows}\n" + "*"*40 + "\n")

    if not has_mismatch:
        match_report.append([file_name, "Match", len(df_before), extra_rows if extra_rows else "None"])

# MAIN SCRIPT

before_files = set(os.listdir(before_folder))
after_files = set(os.listdir(after_folder))
common_files = sorted(before_files.intersection(after_files))

for fname in common_files:
    if fname.endswith(".csv"):
        compare_csv_files(fname)

# Save outputs
pd.DataFrame(match_report, columns=["File Name", "Status", "Total Rows in BEFORE", "Missing Rows in AFTER"]).to_csv("match_report.csv", index=False)
pd.DataFrame(mismatch_report, columns=["File Name", "Row Number", "Column Name", "Mismatch Reason"]).to_csv("mismatch_report.csv", index=False)

with open("extra_rows_in_before.txt", "w") as f:
    f.writelines(extra_rows_info)

print("✅ Comparison completed.")
print("Files generated:\n - match_report.csv\n - mismatch_report.csv\n - extra_rows_in_before.txt")

    
    
</code>
    
</pre>
