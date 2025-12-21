# Countries with the Highest National Debt

This project fetches the latest data on national debt from the *World Population Review* website and visualizes it using interactive plots. It shows the total debt, debt as a percentage of GDP, and debt per capita for each country.  It highlights `user-selected countries` for comparison, prints `comparative table`, generates `narrative analysis`, and provides `interactive bar charts`, and `choropleth maps` for a comprehensive understanding of debt profiles across all countries. 

*(as for suggestion, quickly scan this md file down until the very end. To fully grasp what this project is all about)*

---
## Required Libraries

The project relies on several Python libraries. `requests` is used to send HTTP requests and retrieve HTML content from the web. `BeautifulSoup` is employed to parse the HTML content, allowing the extraction of table data. `pandas` is used for data manipulation, sorting, filtering, and preparing it for visualization. `plotly.graph_objects` provides tools for creating detailed and interactive bar charts and tables, while `plotly.express` simplifies choropleth map creation. `re`, the regular expressions library, is used extensively for extracting numbers from strings that include currency symbols, percentages, and units such as T, B, or M. It is crucial that all these libraries are installed before executing the code, as missing packages will cause immediate errors.

```python
import requests
from bs4 import BeautifulSoup
import pandas as pd
import plotly.graph_objects as go
import plotly.express as px
import re
```

## User Inputs

The code allows the user to define two countries to highlight for comparison. The main country is stored in `USER_COUNTRY`, with its display color defined in `USER_COLOR`. The secondary country is stored in `HIGHLIGHT_COUNTRY` and is assigned `HIGHLIGHT_COLOR`. These inputs determine which countries will be emphasized in all charts, tables, and maps. It is critical that the country names exactly match the entries in the source website and the Plotly location database, including capitalization and spacing, because any mismatch can result in missing data or failed visualizations.

```python
USER_COUNTRY = "Philippines"
USER_COLOR = "teal"
HIGHLIGHT_COUNTRY = "Japan"
HIGHLIGHT_COLOR = "orange"
```

## Fetching National Debt Data

The project retrieves data from the World Population Review website. The HTML of the page is requested using `requests.get()`, and the text is parsed using `BeautifulSoup` with the HTML parser. The code locates the first `<table>` element and extracts all rows using `find_all("tr")`. The first row, which is the header, is skipped, leaving only the rows representing individual countries.

```python
url = "https://worldpopulationreview.com/country-rankings/countries-by-national-debt"
html = requests.get(url).text
soup = BeautifulSoup(html, "html.parser")
table = soup.find("table")
rows = table.find_all("tr")[1:]
```

## Parsing and Cleaning Data

To convert debt values to raw numbers, a dictionary `DEBT_MULT` is defined that maps units `T`, `B`, and `M` to their corresponding numerical multipliers. An empty list `records` is initialized to store the parsed data. Each row is iterated over, and the code checks for a valid number of `<td>` columns. The country name is extracted from the second column, skipping any rows labeled "TOTAL". The national debt label, debt percentage of GDP, and debt per capita are extracted from subsequent columns.

```python
DEBT_MULT = {"T": 1e12, "B": 1e9, "M": 1e6}
records = []

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

National debt in USD is parsed using a regular expression that extracts the numeric part and the unit. The numeric part is converted to a float and multiplied by the corresponding factor from `DEBT_MULT` to produce a raw integer value representing the total debt in USD. Any row that does not match the expected pattern is skipped. Debt as a percentage of GDP is extracted by removing the `%` symbol and converting the string to a float. If the value is missing, it is set to `None`. Debt per capita is parsed similarly, handling optional units `Mn` or `Bn` and converting them to integer USD values. Rows with unrecognized formats are skipped. Finally, each country’s data is stored as a dictionary and appended to `records`.

```python
m = re.match(r"\$(\d+(\.\d+)?)([TBM])", debt_text)
if not m:
    continue
debt_val = float(m.group(1))
debt_unit = m.group(3)
debt_usd = int(debt_val * DEBT_MULT[debt_unit])

debt_pct_gdp = float(gdp_pct_text.replace("%", "")) if "%" in gdp_pct_text else None

pc_text = per_capita_text.replace("$", "").replace(",", "").strip()
pc_match = re.match(r"(\d+(\.\d+)?)\s*(Mn|Bn)?", pc_text)
per_capita = None
if pc_match:
    pc_val = float(pc_match.group(1))
    pc_unit = pc_match.group(3)
    if pc_unit == "Mn":
        per_capita = int(pc_val * 1_000_000)
    elif pc_unit == "Bn":
        per_capita = int(pc_val * 1_000_000_000)
    else:
        per_capita = int(pc_val)

records.append({
    "Country": country,
    "Debt Label": debt_text,
    "Debt USD": debt_usd,
    "Debt % GDP": debt_pct_gdp,
    "Debt Per Capita": per_capita
})
```

The list `records` is converted into a pandas DataFrame, which serves as the main data structure for further analysis.

```python
df = pd.DataFrame(records)
```

## Helper Functions

The code defines several helper functions that simplify ranking, labeling, and coloring for the visualizations.

The `create_rank_labels` function takes a DataFrame, a column to sort by (`value_col`), and an optional `value_format` parameter that specifies how the value should be displayed. The function first sorts the DataFrame in descending order by the specified metric and assigns a rank starting from one. This rank represents the country's position relative to others in the dataset.

Within `create_rank_labels`, the nested `format_value` function formats each value according to its type. If the value represents total debt (`value_format="debt"`), the function returns the original debt label extracted from the source. If the value represents a percentage (`value_format="pct"`), it appends a `%` symbol. If the value represents debt per capita (`value_format="percap"`), the function converts large numbers into human-readable units, displaying billions as `B`, millions as `M`, or raw integers if smaller. This ensures the chart labels are easy to read and interpret.

Finally, the function combines the rank and formatted value into a single string for each row, storing it in a new column `Plot Label`. The function returns the DataFrame sorted by the original metric, ready for plotting with Plotly.

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

    df_sorted["Plot Label"] = df_sorted.apply(lambda x: f"rank {x['Rank']}, {format_value(x)}", axis=1)
    return df_sorted.sort_values(value_col)
```

The functions `build_bar_colors` and `build_text_colors` generate lists of colors for bars and text labels in plots, respectively. They highlight the user-selected countries with the specified colors and default to blue or black for other countries. Matching is exact, so any mismatch in country names will result in default coloring.

```python
def build_bar_colors(df, country_col="Country", user_country=None, user_color=None,
                     extra_country=None, extra_color=None):
    colors = []
    for c in df[country_col]:
        if user_country and c == user_country:
            colors.append(user_color)
        elif extra_country and c == extra_country:
            colors.append(extra_color)
        else:
            colors.append("#1f77b4")
    return colors

def build_text_colors(df, country_col="Country", user_country=None, user_color=None,
                     extra_country=None, extra_color=None):
    colors = []
    for c in df[country_col]:
        if user_country and c == user_country:
            colors.append(user_color)
        elif extra_country and c == extra_country:
            colors.append(extra_color)
        else:
            colors.append("black")
    return colors
```

The function `format_value_units` ensures that numeric values are displayed with appropriate units and formatting for readability in tables and charts. It handles percentages, debt in USD, and per capita debt, converting large numbers to trillions or billions and adding the `$` symbol.

```python
def format_value_units(val, metric):
    if metric == "Debt % GDP":
        return f"{val}%"
    elif metric in ["National Debt (USD)", "Debt Per Capita"]:
        if val >= 1e12:
            return f"${val/1e12:.2f} T"
        elif val >= 1e9:
            return f"${val/1e9:.2f} B"
        elif val >= 1e6:
            return f"${val/1e6:.2f} M"
        else:
            return f"${val:,}"
    return val
```

The function `get_rank` retrieves the rank of a specific country in a sorted DataFrame. It returns `None` if the country does not exist in the dataset.

```python
def get_rank(df_sorted, country):
    row = df_sorted[df_sorted["Country"]==country]
    return int(row["Rank"].iloc[0]) if not row.empty else None
```

### Bar Charts

The code prepares three separate DataFrames for plotting by using the previously defined `create_rank_labels` helper function. `df1_plot` contains national debt in USD, `df2_plot` contains debt as a percentage of GDP, and `df3_plot` contains debt per capita. The `dropna` method ensures that only rows with valid values are used for the percentage and per capita metrics, preventing errors during plotting.

```python
df1_plot = create_rank_labels(df, "Debt USD", "debt")
df2_plot = create_rank_labels(df.dropna(subset=["Debt % GDP"]), "Debt % GDP", "pct")
df3_plot = create_rank_labels(df.dropna(subset=["Debt Per Capita"]), "Debt Per Capita", "percap")
```

The three DataFrames are stored with titles and value column names in the `bar_charts` list. This allows the code to loop through each chart type and generate horizontal bar charts using Plotly.

```python
bar_charts = [
    ("National Debt by Country", "Debt USD", df1_plot),
    ("Debt as Percentage of GDP", "Debt % GDP", df2_plot),
    ("Debt Per Capita by Country", "Debt Per Capita", df3_plot)
]
```

For each chart, `go.Figure(go.Bar(...))` is used to create a horizontal bar chart. The x-axis is the metric value, the y-axis is the country name, and the text labels are derived from the `Plot Label` column created by `create_rank_labels`. Custom colors are applied using `build_bar_colors` for the bars and `build_text_colors` for the text, highlighting the user-selected countries. The chart height is dynamically adjusted based on the number of countries to ensure readability.

```python
for title, val_col, plot_df in bar_charts:
    fig = go.Figure(go.Bar(
        x=plot_df[val_col],
        y=plot_df["Country"],
        orientation="h",
        text=plot_df["Plot Label"],
        textposition="outside",
        marker=dict(
            color=build_bar_colors(plot_df, user_country=USER_COUNTRY, user_color=USER_COLOR,
                                   extra_country=HIGHLIGHT_COUNTRY, extra_color=HIGHLIGHT_COLOR)
        ),
        textfont=dict(
            color=build_text_colors(plot_df, user_country=USER_COUNTRY, user_color=USER_COLOR,
                                    extra_country=HIGHLIGHT_COUNTRY, extra_color=HIGHLIGHT_COLOR)
        )
    ))
    fig.update_layout(
        title=title,
        xaxis_title=val_col,
        height=max(600, len(plot_df)*28),
        margin=dict(l=220, r=320),
        template="plotly_white"
    )
    fig.show()
```

## Summary DataFrame

A summary DataFrame is created to consolidate the metrics and ranks for the two highlighted countries. The code extracts the metric values and ranks for `USER_COUNTRY` and `HIGHLIGHT_COUNTRY` from the previously prepared DataFrames. The `get_rank` function retrieves the rank for each metric.

Absolute differences and ratios between the countries are calculated to allow easy comparison. These calculations are stored in new columns: `Difference`, `Ratio (main/other)`, and `Ratio (other/main)`. The values are formatted into human-readable units using the `format_value_units` helper function.

```python
summary_df = pd.DataFrame(summary_data)
summary_df["Difference"] = diff_abs
summary_df["Ratio (main/other)"] = ratio_main_other
summary_df["Ratio (other/main)"] = ratio_other_main

for col in [USER_COUNTRY, HIGHLIGHT_COUNTRY, "Difference"]:
    summary_df[col] = [format_value_units(val, metric) for val, metric in zip(summary_df[col], summary_df["Metric"])]
```

A colored table is created using `go.Table` to display the metrics, differences, and ranks, with the user-selected countries highlighted in their respective colors.

```python
fig_table = go.Figure(data=[go.Table(
    header=dict(values=list(summary_df.columns), fill_color='lightgrey', align='center', font=dict(size=14, color='black')),
    cells=dict(values=table_values, fill_color=colors, align='center', font=dict(color='black', size=12))
)])
fig_table.update_layout(title="Country Debt Comparison Table")
fig_table.show()
```

## Narrative with Explanation

The function accepts three parameters: `df`, which is the summary DataFrame containing metrics and ranks; `main_country`, which is the primary country of interest; and `compare_country`, which is the secondary country for comparison. An empty list `statements` is initialized to collect the narrative strings.

```python
def narrative_with_explanation(df, main_country, compare_country):
    statements = []
```

The total number of countries in the dataset is determined using `len(df1_plot)`, and this is included in the introductory text to provide context. The introduction explains the metrics being analyzed: `National Debt (USD)`, `Debt % GDP`, and `Debt Per Capita`.

```python
total_countries = len(df1_plot)
statements.append(f"The Narrative and Comparison Statements\n")
statements.append(f"Out of {total_countries} countries, two countries are compared: {main_country} and {compare_country} in terms of Debt Profiles.\n")
statements.append("National Debt (USD) represents the total debt of a country in US Dollars.\n"
                  "Debt % GDP shows the ratio of national debt to the country's GDP, indicating debt burden.\n"
                  "Debt Per Capita indicates the average debt per person in the country.\n\n")
```

The function then iterates over each of the two countries to summarize individual metrics. For each country, it retrieves the value of `National Debt (USD)`, `Debt % GDP`, and `Debt Per Capita` from the summary DataFrame. It also retrieves the rank for each metric. These values are formatted into human-readable sentences and appended to the `statements` list.

```python
for country in [main_country, compare_country]:
    debt = df.loc[df['Metric']=="National Debt (USD)", country].values[0]
    pct_gdp = df.loc[df['Metric']=="Debt % GDP", country].values[0]
    per_capita = df.loc[df['Metric']=="Debt Per Capita", country].values[0]
    rank_debt = df.loc[df['Metric']=="National Debt (USD)", f"{country} Rank"].values[0]
    rank_gdp = df.loc[df['Metric']=="Debt % GDP", f"{country} Rank"].values[0]
    rank_pc = df.loc[df['Metric']=="Debt Per Capita", f"{country} Rank"].values[0]
    statements.append(f"{country} has a national debt of {debt} (Rank {rank_debt}), "
                      f"which is {pct_gdp}% of its GDP (Rank {rank_gdp}), and a per capita debt of {per_capita} (Rank {rank_pc}).\n")
```

Next, the function performs comparative analysis for each metric. It extracts the metric values and the ratios (`Ratio (main/other)` and `Ratio (other/main)`) for both countries. Using these, it constructs sentences comparing the main country to the secondary country, including both the actual values and how many times one is larger than the other. This provides a clear, quantitative comparison.

```python
for metric in df['Metric']:
    val_main = df.loc[df['Metric']==metric, main_country].values[0]
    val_other = df.loc[df['Metric']==metric, compare_country].values[0]
    ratio_main = df.loc[df['Metric']==metric, "Ratio (main/other)"].values[0]
    ratio_inverse = df.loc[df['Metric']==metric, "Ratio (other/main)"].values[0]
    rank_main = df.loc[df['Metric']==metric, f"{main_country} Rank"].values[0]
    rank_other = df.loc[df['Metric']==metric, f"{compare_country} Rank"].values[0]
    statements.append(
        f"For {metric}, {main_country} has {val_main} (Rank {rank_main}), which is {ratio_main}× of {compare_country}'s {val_other} (Rank {rank_other}).\n"
        f"On the other hand, {compare_country} is {ratio_inverse}× bigger than {main_country}.\n"
    )
```

Finally, the function joins all narrative statements into a single string separated by newlines and returns it. This string is printed to the console for the user.

```python
return "\n".join(statements)

print(narrative_with_explanation(summary_df, USER_COUNTRY, HIGHLIGHT_COUNTRY))
```

This approach ensures that all numerical insights from the summary DataFrame are translated into readable text, making the data accessible and interpretable. It is particularly useful for creating reports, dashboards, or presentations without requiring the viewer to manually read charts or tables.

### Geo Maps Data Preparation

First, a copy of the main DataFrame `df` is created and three new columns are added to store values for each map: `Debt USD Map`, `Debt % GDP Map`, and `Debt Per Capita Map`. This separation ensures that each metric can be independently visualized without modifying the original DataFrame.

```python
df_map = df.copy()
df_map["Debt USD Map"] = df_map["Debt USD"]
df_map["Debt % GDP Map"] = df_map["Debt % GDP"]
df_map["Debt Per Capita Map"] = df_map["Debt Per Capita"]
```

## Highlighting Selected Countries

The function `add_country_markers` is defined to overlay country names on the map for the main and highlight countries. It iterates over the two countries and adds a `scattergeo` trace with the country name as text. The `textfont` parameter sets the color and size of the label. This ensures that the user-selected countries stand out on the map.

```python
def add_country_markers(fig, main_country, highlight_country):
    for country, color in zip([main_country, highlight_country], [USER_COLOR, HIGHLIGHT_COLOR]):
        row = df_map[df_map["Country"] == country]
        if not row.empty:
            fig.add_scattergeo(
                locations=[country],
                locationmode="country names",
                text=[country],
                mode="text",
                textfont=dict(color=color, size=16),
            )
```

## National Debt (USD) Map

A choropleth map is created for total national debt using Plotly Express. The `locations` column specifies the country names, and `color` is mapped to `Debt USD Map`. Hover information includes all three metrics to provide additional context when a user inspects a country. The color scale `OrRd` represents increasing debt levels with progressively darker shades of orange and red. The map height and margins are adjusted to ensure readability.

```python
fig_map_usd = px.choropleth(
    df_map,
    locations="Country",
    locationmode="country names",
    color="Debt USD Map",
    hover_name="Country",
    hover_data={"Debt USD Map": True, "Debt % GDP": True, "Debt Per Capita": True},
    color_continuous_scale="OrRd",
    labels={"Debt USD Map": "National Debt (USD)"},
    title="National Debt (USD) by Country"
)
add_country_markers(fig_map_usd, USER_COUNTRY, HIGHLIGHT_COUNTRY)
fig_map_usd.update_layout(height=800, margin=dict(l=20, r=20, t=60, b=20))
fig_map_usd.show()

```

##  Debt as Percentage of GDP Map

A second choropleth map visualizes national debt as a percentage of GDP. It uses the `YlGnBu` color scale, where higher percentages appear in darker blue-green shades. This map helps users understand the relative debt burden in proportion to the size of each country’s economy.

```python
fig_map_gdp = px.choropleth(
    df_map,
    locations="Country",
    locationmode="country names",
    color="Debt % GDP Map",
    hover_name="Country",
    hover_data={"Debt USD Map": True, "Debt % GDP Map": True, "Debt Per Capita Map": True},
    color_continuous_scale="YlGnBu",
    labels={"Debt % GDP Map": "Debt % of GDP"},
    title="Debt as Percentage of GDP by Country"
)
add_country_markers(fig_map_gdp, USER_COUNTRY, HIGHLIGHT_COUNTRY)
fig_map_gdp.update_layout(height=800, margin=dict(l=20, r=20, t=60, b=20))
fig_map_gdp.show()
```

## Debt Per Capita Map

The third map visualizes debt per capita using the `PuRd` color scale. This map shows the average debt per person in each country, which can differ significantly from total debt or debt relative to GDP. Highlighting the selected countries helps focus attention on user-specific comparisons.

```python
fig_map_pc = px.choropleth(
    df_map,
    locations="Country",
    locationmode="country names",
    color="Debt Per Capita Map",
    hover_name="Country",
    hover_data={"Debt USD Map": True, "Debt % GDP Map": True, "Debt Per Capita Map": True},
    color_continuous_scale="PuRd",
    labels={"Debt Per Capita Map": "Debt Per Capita (USD)"},
    title="Debt Per Capita by Country"
)
add_country_markers(fig_map_pc, USER_COUNTRY, HIGHLIGHT_COUNTRY)
fig_map_pc.update_layout(height=800, margin=dict(l=20, r=20, t=60, b=20))
fig_map_pc.show()
```

## Important Notes

The choropleth maps rely on exact country names matching Plotly’s recognized names. Any discrepancies can result in missing countries or markers. The `hover_data` parameter ensures that all three debt metrics are visible when hovering over any country, providing a comprehensive overview. The `add_country_markers` function ensures that the selected countries are clearly labeled regardless of color scale or geographic location.

---
## What does it look like?
The Debt comparison profiles will dynamically change if we change the inputs in this section *(located at the very top part of the code). For simplicity, visit the Jupyter notebook file.* As for example,

```python
USER_COUNTRY = "Philippines"
USER_COLOR = "teal"
HIGHLIGHT_COUNTRY = "Japan"
HIGHLIGHT_COLOR = "orange"
```
The code allows the user to define two countries to highlight for comparison. The main country is stored in `USER_COUNTRY`, with its display color defined in `USER_COLOR`. The secondary country is stored in `HIGHLIGHT_COUNTRY` and is assigned `HIGHLIGHT_COLOR`. These inputs determine which countries will be emphasized in all charts, tables, and maps. It is critical that the country names exactly match the entries in the source website and the Plotly location database, including capitalization and spacing, because any mismatch can result in missing data or failed visualizations.

### Country Debt Comparison Table
<img width="1042" height="85" alt="image" src="https://github.com/user-attachments/assets/6b16d068-b7be-467c-afab-08a0f93e6f0d" />

### The Narrative and Comparison Statements
<img width="964" height="266" alt="image" src="https://github.com/user-attachments/assets/11146d8a-5c9b-4ca3-8e5a-13313b783cf7" />

### National Debt by Country
<img width="1122" height="641" alt="image" src="https://github.com/user-attachments/assets/d8b6519d-9be1-42f3-8b05-776d5ae9e0b2" />
<img width="1580" height="5320" alt="newplot" src="https://github.com/user-attachments/assets/3216ab58-8b18-4e84-979f-2f7298979cd8" />

### Debt as Percentage of GDP by Country
<img width="1151" height="652" alt="image" src="https://github.com/user-attachments/assets/1f00b45c-fae5-4440-8434-2f6f6f0929e9" />
<img width="1580" height="5320" alt="newplot (1)" src="https://github.com/user-attachments/assets/f4f2b51d-4c56-445f-aa25-aa6c01630e0d" />

### Debt Per Capita by Country
<img width="1112" height="668" alt="image" src="https://github.com/user-attachments/assets/4cf28770-497e-4eb1-ba55-02dbee0fb731" />
<img width="1580" height="5320" alt="newplot (2)" src="https://github.com/user-attachments/assets/f3c90277-bbac-42b3-9227-b749f2f7f011" />






