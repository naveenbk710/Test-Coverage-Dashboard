import os
import subprocess
import json
import shutil
from pathlib import Path
import pandas as pd
import matplotlib.pyplot as plt
import streamlit as st

REPO_URL = "https://github.com/pallets/flask.git"
CLONE_DIR = "sample_repo"
COVERAGE_JSON = "coverage.json"
COVERAGE_HTML_DIR = "htmlcov"

def clone_repo(repo_url, clone_dir):
    if os.path.exists(clone_dir):
        shutil.rmtree(clone_dir)
    subprocess.run(["git", "clone", repo_url, clone_dir], check=True)

def create_sample_tests(clone_dir):
    tests_dir = Path(clone_dir) / "tests"
    tests_dir.mkdir(exist_ok=True)
    sample_test = tests_dir / "test_sample.py"
    sample_test.write_text('''
from flask import Flask

def test_basic():
    app = Flask(__name__)
    assert app is not None
''')

def run_coverage(clone_dir):
    commands = [
        ["coverage", "run", "-m", "pytest"],
        ["coverage", "json", "-o", COVERAGE_JSON],
        ["coverage", "html", "-d", COVERAGE_HTML_DIR],
    ]
    for cmd in commands:
        subprocess.run(cmd, cwd=clone_dir, check=True)

def parse_coverage(clone_dir):
    coverage_path = Path(clone_dir) / COVERAGE_JSON
    with open(coverage_path, "r") as f:
        data = json.load(f)
    files = data.get("files", {})
    records = []
    for file_path, stats in files.items():
        summary = stats.get("summary", {})
        records.append({
            "File": file_path,
            "Coverage %": summary.get("percent_covered", 0),
            "Missing Lines": summary.get("missing_lines", 0),
            "Covered Lines": summary.get("covered_lines", 0),
        })
    df = pd.DataFrame(records)
    return df.sort_values(by="Coverage %")

def plot_coverage(df):
    fig, ax = plt.subplots(figsize=(12, 6))
    colors = ["red" if x < 80 else "green" for x in df["Coverage %"]]
    ax.barh(df["File"], df["Coverage %"], color=colors)
    ax.set_xlabel("Coverage Percentage")
    ax.set_ylabel("Files")
    ax.set_title("Test Coverage by File")
    ax.axvline(80, color="orange", linestyle="--", label="Threshold (80%)")
    ax.legend()
    plt.tight_layout()
    return fig

def run_dashboard(df, clone_dir):
    st.title("📊 Test Coverage Dashboard")
    total_files = len(df)
    avg_coverage = df["Coverage %"].mean()
    uncovered_files = df[df["Coverage %"] < 80]

    col1, col2, col3 = st.columns(3)
    col1.metric("Total Files", total_files)
    col2.metric("Average Coverage", f"{avg_coverage:.2f}%")
    col3.metric("Files Below Threshold", len(uncovered_files))

    st.subheader("Coverage Table")
    st.dataframe(df)

    st.subheader("Coverage Visualization")
    st.pyplot(plot_coverage(df))

    st.subheader("⚠️ Low Coverage Files")
    st.dataframe(uncovered_files)

    html_report = Path(clone_dir) / COVERAGE_HTML_DIR / "index.html"
    if html_report.exists():
        st.markdown(f"[Detailed HTML Report](file://{html_report.resolve()})")

def main():
    st.sidebar.title("Settings")
    repo_url = st.sidebar.text_input("GitHub Repository URL", REPO_URL)

    if st.sidebar.button("Run Coverage Analysis"):
        with st.spinner("Cloning repository..."):
            clone_repo(repo_url, CLONE_DIR)
        with st.spinner("Creating tests..."):
            create_sample_tests(CLONE_DIR)
        with st.spinner("Running coverage..."):
            run_coverage(CLONE_DIR)
        with st.spinner("Parsing coverage data..."):
            df = parse_coverage(CLONE_DIR)
        run_dashboard(df, CLONE_DIR)
    else:
        st.info("Enter repository URL and click Run Coverage Analysis.")

if __name__ == "__main__":
    main()# Test-Coverage-Dashboard
