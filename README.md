# Countries with the Highest National Debt

This project fetches the latest data on national debt from the *World Population Review* website and visualizes it using interactive Plotly bar charts. It shows the total debt, debt as a percentage of GDP, and debt per capita for each country. The charts also display ranked labels in the format `rank X, value` for clarity.

---

## How the Code Works

The code is organized in several sections, each handling a specific task. Here’s a detailed explanation of each part.

### [1] Importing Required Libraries

```python
import requests
from bs4 import BeautifulSoup
import pandas as pd
import plotly.graph_objects as go
import re
```
We import `requests` to fetch the webpage content, `BeautifulSoup` to parse HTML, `pandas` to handle dataframes, `plotly.graph_objects` to create interactive bar charts, and `re` for regular expression parsing.

### [2] Fetching and Parsing the Webpage

```python
url = "https://worldpopulationreview.com/country-rankings/countries-by-national-debt"
html = requests.get(url).text
soup = BeautifulSoup(html, "html.parser")

table = soup.find("table")
rows = table.find_all("tr")[1:]
```
We get the HTML content of the webpage and parse it. Then we locate the table containing the debt data and extract all rows except the header.

### [3] Preparing Multipliers and Data Storage

```python
DEBT_MULT = {"T": 1e12, "B": 1e9, "M": 1e6}
records = []
```
Debt values are reported in trillions (T), billions (B), or millions (M). The `DEBT_MULT` dictionary converts these to USD. `records` will store each country's data as a dictionary.

### [4] Extracting Data for Each Country

```python
for row in rows:
    cols = row.find_all("td")
    if len(cols) < 5:
        continue

    country = cols[1].text.strip()
    if country.upper() == "TOTAL":
        continue

    debt_text = cols[2].text.strip()
    gdp_pct_text = cols[3].text.strip()
    per_capita_text = cols[4].text.strip()
```
We loop through each row and extract the country name, total debt, debt as a percentage of GDP, and debt per capita. Rows labeled "TOTAL" or incomplete rows are skipped.

### [5] Converting Debt Text to Numeric Values

```python
m = re.match(r"\$(\d+(\.\d+)?)([TBM])", debt_text)
if not m:
    continue
debt_val = float(m.group(1))
debt_unit = m.group(3)
debt_usd = int(debt_val * DEBT_MULT[debt_unit])
```
We parse the debt string to extract the numeric value and unit, then multiply by the corresponding factor to get the amount in USD.

### [6] Converting Debt % of GDP

```python
debt_pct_gdp = float(gdp_pct_text.replace("%", "")) if "%" in gdp_pct_text else None
```
We remove the `%` symbol and convert the value to a floating-point number. Missing values are stored as `None`.

### [7] Converting Debt Per Capita
```python
per_capita = None
pc_text = per_capita_text.replace("$", "").replace(",", "").strip()
pc_match = re.match(r"(\d+(\.\d+)?)\s*(Mn|Bn)?", pc_text)
if pc_match:
    pc_val = float(pc_match.group(1))
    pc_unit = pc_match.group(3)
    if pc_unit == "Mn":
        per_capita = int(pc_val * 1_000_000)
    elif pc_unit == "Bn":
        per_capita = int(pc_val * 1_000_000_000)
    else:
        per_capita = int(pc_val)
```
Debt per capita can be in millions or billions. We parse the value and convert it into an integer.

### [8] Storing Each Country's Data
```python
records.append({
    "Country": country,
    "Debt Label": debt_text,
    "Debt USD": debt_usd,
    "Debt % GDP": debt_pct_gdp,
    "Debt Per Capita": per_capita
})
df = pd.DataFrame(records)
```
We store each country's data in a dictionary and then convert the list of dictionaries into a Pandas DataFrame.

### [9] Creating Rank Labels for Charts
```python
def create_rank_labels(df, value_col, value_format="raw"):
    df_sorted = df.sort_values(value_col, ascending=False).reset_index(drop=True)
    df_sorted["Rank"] = df_sorted.index + 1

    def format_value(row):
        if value_format == "debt":
            return f"{row['Debt Label']}"
        elif value_format == "pct":
            return f"{row[value_col]}%"
        elif value_format == "percap":
            val = row[value_col]
            if val >= 1e9:
                return f"${val/1e9:.2f} B"
            elif val >= 1e6:
                return f"${val/1e6:.2f} M"
            else:
                return f"${val}"
        else:
            return str(row[value_col])

    df_sorted["Plot Label"] = df_sorted.apply(
        lambda x: f"rank {x['Rank']}, {format_value(x)}", axis=1
    )
    return df_sorted.sort_values(value_col)
```
This helper function sorts the DataFrame by a selected column, assigns a rank to each country, and generates a label in the format `rank X, value`. It handles debt in USD, percent of GDP, or per capita amounts.

### [10] National Debt Chart
```python
df1_plot = create_rank_labels(df, "Debt USD", "debt")
fig1 = go.Figure(go.Bar(
    x=df1_plot["Debt USD"],
    y=df1_plot["Country"],
    orientation="h",
    text=df1_plot["Plot Label"],
    textposition="outside"
))
fig1.update_layout(
    title="National Debt by Country",
    xaxis_title="National Debt (USD)",
    height=max(600, len(df1_plot)*28),
    margin=dict(l=220, r=320),
    xaxis=dict(range=[0, df1_plot["Debt USD"].max()*1.3]),
    template="plotly_white"
)
fig1.show()
```
We create a horizontal bar chart for total debt, dynamically adjusting height and margins. Labels use the `rank X, value` format.

### [11] Debt as Percentage of GDP Chart
```python
df2 = df.dropna(subset=["Debt % GDP"])
df2_plot = create_rank_labels(df2, "Debt % GDP", "pct")
fig2 = go.Figure(go.Bar(
    x=df2_plot["Debt % GDP"],
    y=df2_plot["Country"],
    orientation="h",
    text=df2_plot["Plot Label"],
    textposition="outside"
))
fig2.update_layout(
    title="Debt as Percentage of GDP",
    xaxis_title="Debt (% of GDP)",
    height=max(600, len(df2_plot)*28),
    margin=dict(l=220, r=200),
    xaxis=dict(range=[0, df2_plot["Debt % GDP"].max()*1.2]),
    template="plotly_white"
)
fig2.show()
```
This chart shows debt relative to GDP. Countries without GDP percentage data are excluded.

### [12] Debt Per Capita Chart
```python
df3 = df.dropna(subset=["Debt Per Capita"])
df3_plot = create_rank_labels(df3, "Debt Per Capita", "percap")
fig3 = go.Figure(go.Bar(
    x=df3_plot["Debt Per Capita"],
    y=df3_plot["Country"],
    orientation="h",
    text=df3_plot["Plot Label"],
    textposition="outside"
))
fig3.update_layout(
    title="Debt Per Capita by Country",
    xaxis_title="Debt Per Capita (USD)",
    height=max(600, len(df3_plot)*28),
    margin=dict(l=220, r=240),
    xaxis=dict(range=[0, df3_plot["Debt Per Capita"].max()*1.2]),
    template="plotly_white"
)
fig3.show()
```
This chart displays how much each person would owe if the national debt were evenly divided.

---
## Summary
This project demonstrates how to scrape live data from a website, clean and convert text into numeric formats, and produce interactive visualizations in Python. Using rank labels makes it clear which countries have the highest national debt in absolute terms, relative to GDP, and per capita.

*(simply view the jupyter notebook file above, enjoy!)*

### National Debt Chart
<img width="1610" height="5340" alt="newplot" src="https://github.com/user-attachments/assets/a2176285-05ee-447f-b40a-ae46a4523f1e" />

### Debt as Percentage of GDP Chart
<img width="1610" height="5340" alt="newplot (1)" src="https://github.com/user-attachments/assets/d559f282-39a1-4a3a-995a-03329382242f" />

### Debt Per Capita Chart
<img width="1610" height="5340" alt="newplot (2)" src="https://github.com/user-attachments/assets/d2a1dae9-bada-4446-8ee8-5fbe4da3d4eb" />




