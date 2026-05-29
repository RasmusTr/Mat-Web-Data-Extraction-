### Data Persistence Functions for Progress Tracking and Dataset Storage

```python
def load_processed_names(base_folder="DataMatWeb"):
    """
    Loads already processed names from file.
    Returns a set (for fast lookup).
    """
    # Create the base directory if it does not exist yet
    os.makedirs(base_folder, exist_ok=True)
    filepath = os.path.join(base_folder, "processed_names.txt")

    # If the file tracking processed materials doesn't exist, return an empty set
    if not os.path.exists(filepath):
        return set()

    # Read the file and return a set of unique, stripped material names
    with open(filepath, "r", encoding="utf-8") as f:
        return set(line.strip() for line in f if line.strip())

def save_processed_names(processed_names, base_folder="DataMatWeb"):
    """
    Saves the current list of processed names.
    """
    filepath = os.path.join(base_folder, "processed_names.txt")

    # Write the sorted list of processed names back to the tracking text file
    with open(filepath, "w", encoding="utf-8") as f:
        for name in sorted(processed_names):
            f.write(name + "\n")

def save_wide_all(wide_all, base_folder="DataMatWeb", filename="wide_all.csv"):
    """
    Saves wide_all as a CSV in the specified folder.
    Only saves if wide_all exists and is not empty.
    """

    # Guard clause to skip saving if the DataFrame is uninitialized or empty
    if wide_all is None or wide_all.empty:
        return

    # Ensure the target directory exists before writing the file
    os.makedirs(base_folder, exist_ok=True)

    # Resolve the full destination path and export the DataFrame with its index
    filepath = os.path.join(base_folder, filename)
    wide_all.to_csv(filepath, index=True)

def load_wide_all(base_folder="DataMatWeb", filename="wide_all.csv"):
    """
    Loads wide_all.csv if present.
    Returns DataFrame or None if the file does not exist.
    """

    path = os.path.join(base_folder, filename)

    # Return None early if no matching dataset file is found
    if not os.path.exists(path):
        print("Found no existing wide_all.csv.")
        return None

    print("Load current wide_all.csv ...")

    # Read the data from the CSV file
    df = pd.read_csv(path)

    # If "Material" exists as a column -> set it as the index again
    if "Material" in df.columns:
        df = df.set_index("Material")

    return df

```

### HTML Parsing and Material Link Extraction Functions

```python
def contains_any_term(text, term_list, case_sensitive=False):
    """
    Checks if at least one string from term_list occurs in the provided text.

    Returns True if any term is found, otherwise False.
    """
    # Normalize text and terms to lowercase if case sensitivity is disabled
    if not case_sensitive:
        text = text.lower()
        term_list = [term.lower() for term in term_list]

    # Iterate through each exclusion term and return True on the first match
    for term in term_list:
        if term in text:
            return True

    return False

def extract_materials_from_html_file(filepath, base_url="[https://www.matweb.com](https://www.matweb.com)", exclude_terms=None):
    from bs4 import BeautifulSoup
    from urllib.parse import urljoin

    # Initialize empty list if no exclusion terms are provided
    if exclude_terms is None:
        exclude_terms = []

    # Read the raw HTML file content using UTF-8 encoding
    with open(filepath, "r", encoding="utf-8") as file:
        html = file.read()

    # Parse the raw HTML string using BeautifulSoup
    soup = BeautifulSoup(html, "html.parser")

    materials = []

    # Select all anchor tags containing specific MatWeb datasheet GUID URLs
    for a in soup.select("a[href*='DataSheet.aspx?MatGUID=']"):
        href = a["href"].strip()
        full_url = urljoin(base_url, href)
        name = a.get_text(strip=True)

        # Skip material entry if its name contains any forbidden terms
        if contains_any_term(name, exclude_terms):
            continue

        # Append valid material entry details to the initial collection
        materials.append({
            "name": name,
            "url": full_url
        })

    # Remove duplicates based on URL to ensure list uniqueness
    seen = set()
    unique_materials = []
    for m in materials:
        if m["url"] not in seen:
            unique_materials.append(m)
            seen.add(m["url"])

    return unique_materials

```

### Web Scraping with Browser Attachment and Dynamic Content Waiting

```python
import pandas as pd
from selenium import webdriver
from selenium.webdriver.edge.options import Options
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.common.by import By
from selenium.webdriver.support import expected_conditions as EC


def get_html_from_url(url):
    # Configure Selenium Edge options to attach to a running browser instance
    opts = Options()
    opts.add_experimental_option("debuggerAddress", "127.0.0.1:9222")

    # Connect to the existing Edge browser window via the debugging port
    driver = webdriver.Edge(options=opts)  # ATTACH, no new browser instance
    driver.get(url)

    # Explicitly wait until the anti-bot or challenge page title disappears
    WebDriverWait(driver, 90).until_not(
        EC.title_contains("Just a moment")
    )

    # Warte bis irgendein relevanter Properties-Header auftaucht
    # Build a case-insensitive XPath expression targeting specific material property sections
    xpath_any_props = (
        "//*[contains(translate(., 'ABCDEFGHIJKLMNOPQRSTUVWXYZ', 'abcdefghijklmnopqrstuvwxyz'), 'physical properties')"
        " or contains(translate(., 'ABCDEFGHIJKLMNOPQRSTUVWXYZ', 'abcdefghijklmnopqrstuvwxyz'), 'mechanical properties')"
        " or contains(translate(., 'ABCDEFGHIJKLMNOPQRSTUVWXYZ', 'abcdefghijklmnopqrstuvwxyz'), 'electrical properties')"
        " or contains(translate(., 'ABCDEFGHIJKLMNOPQRSTUVWXYZ', 'abcdefghijklmnopqrstuvwxyz'), 'thermal properties')]"
    )

    # Block execution until at least one target property element is located on the page
    WebDriverWait(driver, 60).until(
        EC.presence_of_element_located((By.XPATH, xpath_any_props))
    )

    # Retrieve the dynamically rendered raw HTML content from the browser
    html = driver.page_source
    
    # Parse all table elements found in the HTML source code into a list of pandas DataFrames
    tables = pd.read_html(html)
```

### Target Dataframe Extraction and Identification Functions

```python
import re

def get_right_dataframe_2(tables):
    # Initialize the variable to hold the matched DataFrame
    selected_df = None

    # Define the ordered list of material property headers to scan for in priority order
    priority_headers = [
        "Physical Properties",
        "Mechanical Properties",
        "Electrical Properties",
        "Thermal Properties",
    ]

    # Loop through each property header to search the tables sequentially by priority
    for header in priority_headers:
        # Compile a case-insensitive regular expression pattern with word boundaries
        pattern = re.compile(rf"\b{re.escape(header)}\b", re.IGNORECASE)

        # Enumerate through all extracted DataFrames in the tables list
        for i, t in enumerate(tables):
            # Guard clause to skip the table if column index 0 does not exist
            if 0 not in t.columns:
                continue

            # Convert the entire first column to strings for uniform text pattern matching
            col0 = t[0].astype(str)
            
            # If any cell in column 0 matches the regex pattern, select this table
            if col0.str.contains(pattern, na=False).any():
                print(f"Gefundene Tabelle: {i} (Header: {header})")
                # Slice and copy only the first two columns (properties and metric values)
                selected_df = t.iloc[:, :2].copy()
                #print("Shape:", selected_df.shape)
                return selected_df

    print("Found no fitting table.")
    return None

def get_right_dataframe_3(tables):
    # Initialize the variable to hold the matched DataFrame
    selected_df = None

    # Define the same ordered list of property headers for structured extraction
    priority_headers = [
        "Physical Properties",
        "Mechanical Properties",
        "Electrical Properties",
        "Thermal Properties",
    ]

    # Loop through each property header to search the tables sequentially by priority
    for header in priority_headers:
        # Compile a case-insensitive regular expression pattern with word boundaries
        pattern = re.compile(rf"\b{re.escape(header)}\b", re.IGNORECASE)

        # Enumerate through all extracted DataFrames in the tables list
        for i, t in enumerate(tables):
            # Guard clause to skip the table if column index 0 does not exist
            if 0 not in t.columns:
                continue

            # Convert the entire first column to strings for uniform text pattern matching
            col0 = t[0].astype(str)
            
            # If any cell in column 0 matches the regex pattern, select this table
            if col0.str.contains(pattern, na=False).any():
                print(f"Gefundene Tabelle: {i} (Header: {header})")
                # Slice and copy only the first two columns (properties and metric values)
                selected_df = t.iloc[:, :2].copy()
                #print("Shape:", selected_df.shape)
                return selected_df

    print("Found no fitting table.")
    return None
    print("Number of Tables:", len(tables))

    # NICHT driver.quit() !!
    # Return both the list of DataFrames and the raw HTML string, leaving the browser open
    return tables, html
```

### Material Data Transformation and Wide-Format Restructuring Functions

```python
### Material Data Transformation and Wide-Format Restructuring Functions

```python
import pandas as pd
import re
from bs4 import BeautifulSoup
from urllib.parse import urljoin

def extract_to_wide_lists(df_2cols: pd.DataFrame, material_name: str = "X") -> pd.DataFrame:
    # Slice the first two columns to guarantee uniform processing inputs
    df_full = df_2cols.iloc[:, :2].copy()
    df_full.columns = ["prop_raw", "metric_raw"]

    # Regular expression pattern to recognize and match floating point or scientific notation numbers
    num_re = re.compile(r"[-+]?(?:\d+(?:\.\d*)?|\.\d+)(?:[eE][-+]?\d+)?")

    # =========================================================
    # NEW (1): Truncate everything from "Descriptive Properties" onwards (inclusive)
    # =========================================================
    # Generate a boolean mask identifying rows starting descriptive properties sections
    desc_mask0 = df_full["prop_raw"].astype(str).str.strip().str.lower().eq("descriptive properties")
    if desc_mask0.any():
        first_desc_idx = desc_mask0.idxmax()
        # Truncate everything from the first descriptive property index onwards
        df_full = df_full.loc[:first_desc_idx - 1]

    # ---------------------------
    # Extract Component Elements
    # ---------------------------
    def parse_element_symbol(text: str) -> str:
        s = str(text).strip()
        # Search for explicit element shortcodes following a comma delimiter
        m = re.search(r",\s*([A-Za-z]{1,3})\b", s)
        return m.group(1) if m else s

    def parse_percent_bounds(metric: str):
        s = str(metric).strip()
        # Standardize equality comparison tokens for easier evaluation
        s = s.replace("≤", "<=").replace("≥", ">=")

        nums = [float(x) for x in num_re.findall(s)]
        if not nums:
            return None

        # Return predefined boundary pairs for less-than-or-equal conditions
        if "<=" in s:
            return (0.0, nums[0])
        # Return predefined boundary pairs for greater-than-or-equal conditions
        if ">=" in s:
            return (nums[0], 100.0)

        matches = list(num_re.finditer(s))
        if len(matches) >= 2:
            between = s[matches[0].end():matches[1].start()]
            # Extract high-low bounds if values specify a hyphenated range
            if "-" in between:
                a, b = float(matches[0].group()), float(matches[1].group())
                return (min(a, b), max(a, b))

        x = nums[0]
        return (x, x)

    def project_to_sum100(bounds, init_vals):
        vals = init_vals[:]
        mins = [b[0] for b in bounds]
        maxs = [b[1] for b in bounds]

        # Constrain initial values to fit inside their structural boundaries
        for i in range(len(vals)):
            vals[i] = max(mins[i], min(maxs[i], vals[i]))

        # Run up to 200 optimization iterations to adjust weights evenly to 100%
        for _ in range(200):
            total = sum(vals)
            delta = 100.0 - total
            if abs(delta) < 1e-9:
                break

            if delta > 0:
                slack = [maxs[i] - vals[i] for i in range(len(vals))]
                slack_total = sum(s for s in slack if s > 1e-12)
                if slack_total <= 1e-12:
                    break
                for i in range(len(vals)):
                    if slack[i] > 1e-12:
                        add = delta * (slack[i] / slack_total)
                        vals[i] = min(maxs[i], vals[i] + add)
            else:
                slack = [vals[i] - mins[i] for i in range(len(vals))]
                slack_total = sum(s for s in slack if s > 1e-12)
                if slack_total <= 1e-12:
                    break
                for i in range(len(vals)):
                    if slack[i] > 1e-12:
                        sub = (-delta) * (slack[i] / slack_total)
                        vals[i] = max(mins[i], vals[i] - sub)

        return vals

    def extract_component_elements(df_in: pd.DataFrame):
        start_mask = df_in["prop_raw"].astype(str).str.strip().str.lower().eq("component elements properties")
        if not start_mask.any():
            return [], []

        start_idx = start_mask.idxmax()
        pos = df_in.index.get_loc(start_idx)

        elements, bounds = [], []

        # Iterate vertically through data cells underneath the localized element header
        for j in range(pos + 1, len(df_in)):
            prop = df_in.iloc[j]["prop_raw"]
            met  = df_in.iloc[j]["metric_raw"]

            if pd.isna(prop) or pd.isna(met):
                continue

            prop_s = str(prop).strip()
            met_s  = str(met).strip()

            # Stop: Next properties block (header row)
            # Break if encountering an alternate block header that concludes elements
            if met_s == "Metric" and prop_s.lower().endswith("properties") and prop_s.lower() != "component elements properties":
                break

            # Skip pure header or separator rows
            # Skip rows mirroring technical table descriptions
            if met_s == "Metric":
                continue

            b = parse_percent_bounds(met_s)
            if b is None:
                continue

            elements.append(parse_element_symbol(prop_s))
            bounds.append(b)

        if not elements:
            return [], []

        # Generate middle-of-the-road values from boundary pairs as starting weights
        init = [0.5 * (lo + hi) for (lo, hi) in bounds]
        vals = project_to_sum100(bounds, init)

        # Round and adjust strictly to sum up to exactly 100%
        # Apply strict 4-decimal place rounding and adjust residuals on the primary weight element
        vals = [round(v, 4) for v in vals]
        rest = round(100.0 - sum(vals), 4)
        if abs(rest) > 1e-9:
            k = max(range(len(vals)), key=lambda i: vals[i])
            lo, hi = bounds[k]
            vals[k] = round(max(lo, min(hi, vals[k] + rest)), 4)

            last = round(100.0 - sum(vals), 4)
            if abs(last) > 1e-9:
                k = max(range(len(vals)), key=lambda i: vals[i])
                vals[k] = round(vals[k] + last, 4)

        return elements, vals

    elements_list, perc_list = extract_component_elements(df_full)

    # =========================================================
    # NEW (2): Filter elements list to strictly keep valid element symbols
    # =========================================================
    # Eliminate non-standard chemical notations from identified text collections
    elem_sym_re = re.compile(r"^[A-Z][a-z]{0,2}$")
    filtered = [(e, p) for e, p in zip(elements_list, perc_list) if elem_sym_re.match(str(e).strip())]
    elements_list = [e for e, _ in filtered]
    perc_list     = [p for _, p in filtered]

    # =========================================================
    # NEW (3): Fallback – if no elements were found,
    #          e.g., "Molybdenum, Mo" → ["Mo"] with 100%
    # =========================================================
    # Trigger fallback parsing logic if the primary elements engine yielded no fields
    if not elements_list:
        first_prop_series = df_full["prop_raw"].dropna().astype(str)
        first_prop = first_prop_series.iloc[0] if not first_prop_series.empty else ""
        m = re.search(r",\s*([A-Z][a-z]{0,2})\b", first_prop)
        if m:
            elements_list = [m.group(1)]
            perc_list = [100.0]

    # ---------------------------
    # Remaining Properties (WITHOUT Optical-Cut)
    # ---------------------------
    df = df_full.copy()
    df["prop"] = df["prop_raw"].ffill()

    # ---------------------------
    # Remove Component-Elements block from the remaining properties
    # ---------------------------
    prop_lower = df["prop_raw"].astype(str).str.strip().str.lower()

    start_mask = prop_lower.eq("component elements properties")
    if start_mask.any():
        start_idx = start_mask.idxmax()
        start_pos = df.index.get_loc(start_idx)

        end_pos = len(df)  # Fallback: Until end of DataFrame
        for j in range(start_pos + 1, len(df)):
            pr = str(df.iloc[j]["prop_raw"]).strip()
            mr = str(df.iloc[j]["metric_raw"]).strip()
            if mr == "Metric" and pr.lower().endswith("properties") and pr.lower() != "component elements properties":
                end_pos = j
                break

        drop_idx = df.index[start_pos:end_pos]
        df = df.drop(index=drop_idx)

        df["prop"] = df["prop_raw"].ffill()

    # ---------------------------
    # Drop everything under "Descriptive Properties" (including the row itself)
    # (Kept as a secondary safeguard)
    # ---------------------------
    desc_mask = df["prop"].astype(str).str.strip().str.lower().eq("descriptive properties")
    if desc_mask.any():
        first_desc_idx = desc_mask.idxmax()
        df = df.loc[:first_desc_idx - 1]

    drop_props = {
        "Physical Properties", "Chemical Properties", "Mechanical Properties",
        "Electrical Properties", "Optical Properties",
        "Metric"
    }

    # Filter out empty metrics and metadata header fields from properties
    df = df[df["metric_raw"].notna() & df["prop"].notna()]
    df = df[df["metric_raw"].astype(str).str.strip().ne("Metric")]
    df = df[~df["prop"].isin(drop_props)]

    def parse_metric(text: str):
        s = str(text).strip()

        temp_val, temp_unit = None, None
        # Split string if temperature modifiers appear adjacent to values
        if "@Temperature" in s:
            left, right = s.split("@Temperature", 1)
            s = left.strip()

            nums = num_re.findall(right)
            if nums:
                temp_val = float(nums[1] if len(nums) >= 2 else nums[0])

                last_match = None
                for mm in num_re.finditer(right):
                    last_match = mm
                rest = right[last_match.end():].strip() if last_match else ""
                temp_unit = rest.split()[0] if rest else None

        matches = list(num_re.finditer(s))
        if not matches:
            return None, None, temp_val, temp_unit

        if len(matches) >= 2:
            between = s[matches[0].end():matches[1].start()]
            # Average out maximum and minimum parameters if written as a range
            if "-" in between:
                a = float(matches[0].group())
                b = float(matches[1].group())
                value = (a + b) / 2.0
                unit = s[matches[1].end():].strip() or None
                return value, unit, temp_val, temp_unit

        m0 = matches[0]
        value = float(m0.group())
        unit = s[m0.end():].strip() or None
        return value, unit, temp_val, temp_unit

    store = {}

    # Pack individual metrics sequentially into the core storage dictionary
    for _, r in df.iterrows():
        prop = str(r["prop"]).strip()
        value, unit, tval, tunit = parse_metric(r["metric_raw"])

        if prop not in store:
            store[prop] = {"values": [], "temps": [], "unit": unit, "tunit": tunit}

        if store[prop]["unit"] is None and unit is not None:
            store[prop]["unit"] = unit
        if store[prop]["tunit"] is None and tunit is not None:
            store[prop]["tunit"] = tunit

        if value is not None:
            store[prop]["values"].append(value)
            store[prop]["temps"].append(tval)

    def collapse(lst):
        if len(lst) == 0:
            return None
        if len(lst) == 1:
            return lst[0]
        return lst

    out = {}
    out["Elements"] = elements_list if elements_list else None
    out["Percentage (%)"] = perc_list if perc_list else None

    # Dynamically structure dictionary items for data presentation
    for prop, d in store.items():
        unit = d["unit"]
        tunit = d["tunit"]
        vals = d["values"]
        temps = d["temps"]

        val_col = f"{prop} ({unit})" if unit else prop
        has_any_temp = any(t is not None for t in temps)

        if not has_any_temp:
            out[val_col] = collapse(vals)
            continue

        vals_with_temp, temps_with_temp, vals_rt = [], [], []
        for v, t in zip(vals, temps):
            if t is None:
                vals_rt.append(v)
            else:
                vals_with_temp.append(v)
                temps_with_temp.append(t)

        out[val_col] = collapse(vals_with_temp)

        temp_col = f"Temperature {prop} ({tunit})" if tunit else f"Temperature {prop}"
        out[temp_col] = collapse(temps_with_temp)

        if len(vals_rt) > 0:
            rt_col = f"{prop} ({unit}) RT" if unit else f"{prop} RT"
            out[rt_col] = collapse(vals_rt)

    # Instantiate and label the newly generated single-row DataFrame
    wide = pd.DataFrame([out], index=[material_name])
    wide.index.name = "Material"
    return wide


def append_material_to_wide(
    wide_df: pd.DataFrame | None,
    df_2cols: pd.DataFrame,
    material_name: str
) -> pd.DataFrame:
    """
    Takes an existing wide-format DataFrame (or None) and appends a new material as a row.
    - Columns are automatically expanded (union of all attributes).
    - Identical property names are mapped to the same columns.
    """
    new_row = extract_to_wide_lists(df_2cols, material_name=material_name)

    # Initialize the wide dataset layout if it is currently missing or unpopulated
    if wide_df is None or wide_df.empty:
        return new_row

    # Create an aligned columns structure spanning all historical and incoming fields
    combined_cols = wide_df.columns.union(new_row.columns)
    wide_df = wide_df.reindex(columns=combined_cols)
    new_row = new_row.reindex(columns=combined_cols)

    # Overwrite the material metrics line directly if the identifier index match exists
    if material_name in wide_df.index:
        wide_df.loc[material_name, :] = new_row.iloc[0]
        return wide_df

    # Concat the new material row onto the existing matrix vertically
    return pd.concat([wide_df, new_row], axis=0)


def _pair_temperature_columns_left(columns):
    """
    Sorts columns so that each property column has its corresponding temperature column
    placed directly on its LEFT flank, if available.
    """
    cols = list(columns)
    colset = set(cols)

    out = []
    used = set()

    def base_prop(val_col: str) -> str:
        return val_col.split(" (", 1)[0].strip()

    # Map across standard categories, trying to find corresponding paired temperature vectors
    for c in cols:
        if c in used:
            continue
        if c.startswith("Temperature "):
            continue

        bp = base_prop(c)
        candidates = [tc for tc in cols if tc.startswith(f"Temperature {bp}")]
        tc = candidates[0] if candidates else None

        # Insert paired temperature columns directly onto the left flank of the property column
        if tc and tc in colset and tc not in used:
            out.append(tc)
            used.add(tc)

        out.append(c)
        used.add(c)

    # Collect and clean up left-over columns that didn't process inside primary logic loops
    for c in cols:
        if c not in used:
            out.append(c)
            used.add(c)

    return out


def move_single_values_to_rt(wide_df: pd.DataFrame) -> pd.DataFrame:
    """
    Shifts values into '<ValueCol> RT' if:
      - The property is temperature-governed (Temperature column exists AND contains non-NaN values somewhere)
      - The Temperature cell for this specific row is NaN
      - The primary Value cell is not NaN
      - The target RT column is empty (or does not exist yet -> will be created)
    """
    df = wide_df.copy()

    def base_from_value_col(col: str) -> str:
        s = str(col).strip()
        if s.endswith(" RT"):
            s = s[:-3].rstrip()
        return s.split(" (", 1)[0].strip()

    def base_from_temp_col(col: str) -> str:
        s = str(col)
        if s.startswith("Temperature "):
            s = s[len("Temperature "):]
        return s.split(" (", 1)[0].strip()

    temp_cols = [c for c in df.columns if str(c).startswith("Temperature ")]
    base_to_tempcol = {base_from_temp_col(c): c for c in temp_cols}

    # Identify fields categorized by valid, non-null temperature measurements
    temp_governed_bases = {
        b for b, tc in base_to_tempcol.items()
        if df[tc].notna().any()
    }

    # Scan and filter target column structures inside the active processing environment
    for val_col in [c for c in df.columns if not str(c).startswith("Temperature ") and not str(c).endswith(" RT")]:
        base = base_from_value_col(val_col)
        if base not in temp_governed_bases:
            continue

        temp_col = base_to_tempcol.get(base)
        if temp_col is None:
            continue

        rt_col = f"{val_col} RT"
        if rt_col not in df.columns:
            df[rt_col] = pd.NA

        # Isolate instances lacking ambient measurements, reassigning data points to room temp columns
        mask = df[val_col].notna() & df[temp_col].isna() & df[rt_col].isna()
        if mask.any():
            df.loc[mask, rt_col] = df.loc[mask, val_col]
            df.loc[mask, val_col] = pd.NA

    return df
```

### Web Scraping Loop and Material Data Orchestration Pipeline

```python
import pandas as pd
import os
import re
import time
import random

# Exclude list contains words that filter out unwanted material types
exclude_list = ["overview", "coating", "powder", "weld", "Billet", "Grain", "Electron", "T-111® Tantalum Alloy", "T-222® Tantalum Alloy", "(UNS R07005)", "(UNS R07620)", "(UNS R07941)", "Materion", "MoLa", "Non-Sag", "Elmet Technologies"]
materials = extract_materials_from_html_file("html.txt", exclude_terms=exclude_list)
df = pd.DataFrame(materials)
print("Number of Materials:", len(df))

names = []
urls = []
wide_all = load_wide_all()
processed_names = load_processed_names()

# Process each found material from the extracted DataFrame
for i in range(len(df)):
    name = df.iloc[i, 0]
    url  = df.iloc[i, 1]

    # Skip processing if the material name is already inside processed_names
    if name in processed_names:
        print(name, "already scanned. Scan in new name.")
        continue
    
    print("Load link", (i+1), name)

    Table, Html = get_html_from_url(url) 

    if not Table:
        print("Found no tables – skip :", name)
        time.sleep(random.randint(20, 60))
        continue

    # Optional: Anti-bot throttling delay following the HTTP request
    time.sleep(random.randint(20, 60))

    selected_df_2 = get_right_dataframe_2(Table)
    selected_df_3 = get_right_dataframe_3(Table)
    
    if selected_df_2 is None or selected_df_2.empty:
        print("Found no tables – skip :", name)
        continue

    # ------------------------------------------------------------------------
    # Save data to raw file locations inside Data_MatWeb
    # ------------------------------------------------------------------------
    base_folder = "Data_MatWeb"
    html_folder = os.path.join(base_folder, "HTML")
    csv_folder = os.path.join(base_folder, "PropertiesCSV")
    
    # Establish subdirectories if they are currently missing
    os.makedirs(html_folder, exist_ok=True)
    os.makedirs(csv_folder, exist_ok=True)
    
    # Sanitize the material string to prevent unsafe file-path tokens
    safe_name = re.sub(r'[<>:"/\\|?*]', "_", name)

    # Persist the raw fetched HTML string onto disk
    html_filepath = os.path.join(html_folder, f"{safe_name}.txt")
    with open(html_filepath, "w", encoding="utf-8") as f:
        f.write(Html)

    # Export structured property vectors as tab-separated values
    csv_filepath = os.path.join(csv_folder, f"{safe_name}.txt")
    selected_df_3.to_csv(csv_filepath, sep="\t", index=False)
    # ------------------------------------------------------------------------
    
    # Transform raw elements and bind metrics onto the master matrix
    wide_all = append_material_to_wide(wide_all, selected_df_2, material_name=name)

    # Re-cache processed string identifiers to prevent duplicate processing runs
    processed_names.add(name)
    save_processed_names(processed_names)

    # Backup progress by exporting the wide DataFrame state
    save_wide_all(wide_all)

# Execute final post-processing restructuring routines once loops conclude
if wide_all is not None and not wide_all.empty:
    # Organize columns by putting matching temperature matrices on the left
    wide_all = wide_all.reindex(columns=_pair_temperature_columns_left(wide_all.columns))
    
    # Push missing temperature items directly into designated ambient RT columns
    wide_all = move_single_values_to_rt(wide_all)

    # Override terminal truncation policies to verify dataset metrics cleanly
    pd.set_option("display.max_rows", None)
    pd.set_option("display.max_columns", None)
    pd.set_option("display.width", None)
    pd.set_option("display.max_colwidth", None)

    display(wide_all)
else:
    print("Processed no further materials.")
