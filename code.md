### 

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
