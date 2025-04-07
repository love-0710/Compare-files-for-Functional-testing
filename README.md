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












#Html

<pre>
    <code>



import os
import pandas as pd
import matplotlib.pyplot as plt
from jinja2 import Environment, FileSystemLoader

# Setup paths
before_folder = "before"
after_folder = "after"

# Output data
match_report = []
mismatch_report = []
extra_rows_info = []
summary_stats = {"match": 0, "mismatch": 0}

# Output HTML template folder
template_folder = "templates"
os.makedirs(template_folder, exist_ok=True)

# Write basic HTML Jinja2 template
with open(os.path.join(template_folder, "report_template.html"), "w") as f:
    f.write("""
<!DOCTYPE html>
<html>
<head>
    <title>CSV Comparison Report</title>
    <style>
        body { font-family: Arial, sans-serif; padding: 20px; }
        h1, h2 { color: #333; }
        table { border-collapse: collapse; width: 100%; margin-bottom: 30px; }
        th, td { border: 1px solid #ccc; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
        .pie-chart { max-width: 400px; margin-bottom: 30px; }
    </style>
</head>
<body>
    <h1>CSV Comparison Report</h1>
    <h2>Summary</h2>
    <p>Total Files Compared: {{ total }}</p>
    <p>Matched Files: {{ match }}</p>
    <p>Mismatched Files: {{ mismatch }}</p>

    <div class="pie-chart">
        <img src="pie_chart.png" alt="Pie Chart">
    </div>

    <h2>Matched Files</h2>
    <table>
        <tr><th>File Name</th><th>Status</th><th>Total Rows in BEFORE</th><th>Missing Rows in AFTER</th></tr>
        {% for row in matched %}
        <tr><td>{{ row[0] }}</td><td>{{ row[1] }}</td><td>{{ row[2] }}</td><td>{{ row[3] }}</td></tr>
        {% endfor %}
    </table>

    <h2>Mismatched Files</h2>
    <table>
        <tr><th>File Name</th><th>Row Number</th><th>Column Name</th><th>Mismatch Reason</th></tr>
        {% for row in mismatched %}
        <tr><td>{{ row[0] }}</td><td>{{ row[1] }}</td><td>{{ row[2] }}</td><td>{{ row[3] }}</td></tr>
        {% endfor %}
    </table>
</body>
</html>
""")

def compare_csv(file_name):
    global summary_stats
    path_before = os.path.join(before_folder, file_name)
    path_after = os.path.join(after_folder, file_name)

    try:
        df_before = pd.read_csv(path_before, dtype=str).fillna("NaN")
        df_after = pd.read_csv(path_after, dtype=str).fillna("NaN")
    except Exception as e:
        mismatch_report.append([file_name, "N/A", "N/A", f"Read error: {e}"])
        summary_stats["mismatch"] += 1
        return

    cols_before = list(df_before.columns)
    cols_after = list(df_after.columns)

    if cols_before != cols_after:
        extra_cols = set(cols_after) - set(cols_before)
        if extra_cols:
            for col in extra_cols:
                mismatch_report.append([file_name, "N/A", col, "Extra column in AFTER"])
        else:
            mismatch_report.append([file_name, "N/A", "N/A", "Column mismatch or order mismatch"])
        summary_stats["mismatch"] += 1
        return

    if len(df_after) > len(df_before):
        mismatch_report.append([file_name, "N/A", "N/A", "AFTER has more rows than BEFORE"])
        summary_stats["mismatch"] += 1
        return

    has_mismatch = False
    for idx in range(len(df_after)):
        row_before = df_before.iloc[idx]
        row_after = df_after.iloc[idx]
        for col in cols_before:
            val_before = row_before[col]
            val_after = row_after[col]

            if val_before == "NaN" and val_after != "NaN":
                mismatch_report.append([file_name, idx, col, "Before is NULL, After is not"])
                has_mismatch = True
            elif val_before != "NaN" and val_after == "NaN":
                mismatch_report.append([file_name, idx, col, "After is NULL, Before is not"])
                has_mismatch = True
            elif val_before != val_after:
                mismatch_report.append([file_name, idx, col, f"Mismatch | Before: {val_before} | After: {val_after}"])
                has_mismatch = True

    extra_rows = list(range(len(df_after), len(df_before)))
    if extra_rows:
        extra_rows_info.append(f"File: {file_name}\nExtra row numbers in BEFORE not in AFTER: {extra_rows}\n" + "*"*50 + "\n")

    if has_mismatch:
        summary_stats["mismatch"] += 1
    else:
        match_report.append([file_name, "Match", len(df_before), extra_rows if extra_rows else "None"])
        summary_stats["match"] += 1

# Compare common files
before_files = set(os.listdir(before_folder))
after_files = set(os.listdir(after_folder))
common_files = sorted(before_files.intersection(after_files))

for file in common_files:
    if file.endswith(".csv"):
        compare_csv(file)

# Save reports
pd.DataFrame(match_report, columns=["File Name", "Status", "Total Rows in BEFORE", "Missing Rows in AFTER"]).to_csv("match_report.csv", index=False)
pd.DataFrame(mismatch_report, columns=["File Name", "Row Number", "Column Name", "Mismatch Reason"]).to_csv("mismatch_report.csv", index=False)
with open("extra_rows_in_before.txt", "w") as f:
    f.writelines(extra_rows_info)

# Generate pie chart
labels = ["Matched", "Mismatched"]
sizes = [summary_stats["match"], summary_stats["mismatch"]]
colors = ["green", "red"]
plt.figure(figsize=(4,4))
plt.pie(sizes, labels=labels, autopct='%1.1f%%', colors=colors)
plt.title("Comparison Summary")
plt.savefig("pie_chart.png")
plt.close()

# Generate HTML report
env = Environment(loader=FileSystemLoader(template_folder))
template = env.get_template("report_template.html")
html_out = template.render(
    total=summary_stats["match"] + summary_stats["mismatch"],
    match=summary_stats["match"],
    mismatch=summary_stats["mismatch"],
    matched=match_report,
    mismatched=mismatch_report
)

with open("comparison_report.html", "w", encoding="utf-8") as f:
    f.write(html_out)

print("✅ All reports generated:")
print("- match_report.csv")
print("- mismatch_report.csv")
print("- extra_rows_in_before.txt")
print("- pie_chart.png")
print("- comparison_report.html")



        
    </code>
</pre>
