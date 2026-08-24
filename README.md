# IBM-DATA-SCIENE-PROJECT-CHAPTER-FIVE
Extracting and visualizing Tesla and GameStop stock data — yfinance for share prices, web scraping for quarterly revenue, matplotlib dashboards. IBM Python Project for Data Science final assignment.


# Extracting and Visualizing Stock Data — Tesla & GameStop

Final assignment for the IBM course **Python Project for Data Science** (PY0220EN).

This notebook builds a small stock-data dashboard for two companies — **Tesla (TSLA)** and
**GameStop (GME)**. Share-price history is pulled from the Yahoo Finance API, quarterly
revenue is scraped from static HTML pages, both are cleaned into tidy DataFrames, and the
result is plotted as a two-panel chart per company (price on top, revenue below).

## What it does

| # | Task | Output |
|---|------|--------|
| 1 | Extract Tesla share price with `yfinance` | `tesla_data` — Date, Open, High, Low, Close, Volume |
| 2 | Scrape Tesla quarterly revenue from HTML | `tesla_revenue` — Date, Revenue |
| 3 | Extract GameStop share price with `yfinance` | `gme_data` |
| 4 | Scrape GameStop quarterly revenue from HTML | `gme_revenue` |
| 5 | Plot Tesla price vs. revenue | Two-panel matplotlib figure |
| 6 | Plot GameStop price vs. revenue | Two-panel matplotlib figure |

Both charts are truncated at June 2021, which is where the assignment's revenue data ends.

## Approach

**Price data** comes from `yfinance`. A `Ticker` object is created per symbol and
`.history(period="max")` returns the full available history. `reset_index(inplace=True)`
moves the `Date` index into a regular column so it can be plotted and joined like any
other field.

**Revenue data** is scraped from two static HTML pages hosted by IBM Skills Network.
The page is fetched with `requests`, then `pandas.read_html` parses every `<table>` on the
page and the revenue table is selected by index. The `Revenue` column arrives as strings
like `$10,389`, so commas and dollar signs are stripped with a regex before the values can
be cast to float. Rows with nulls or empty strings are dropped.

**Plotting** uses a single shared `make_graph(stock_data, revenue_data, stock)` helper that
draws price and revenue as stacked subplots on a common date axis.

## Results at a glance

- **Tesla** — price and revenue move together. Revenue climbs steadily from near zero in
  2010 to roughly $10.4B per quarter by 2021, and the share price follows with a sharp
  run-up starting in 2020.
- **GameStop** — the two series come apart. Revenue is flat-to-declining with a strong
  seasonal sawtooth (Q4 holiday spikes to ~$3.5B), while the share price sits under $20 for
  a decade before the January 2021 short squeeze pushes it past $80. A clean illustration
  that price is not always a function of fundamentals.

## Tech stack

`yfinance` · `pandas` · `requests` · `BeautifulSoup` · `matplotlib`

## Running it

```bash
pip install yfinance bs4 nbformat matplotlib lxml
jupyter lab "Revenue Data and Building a Dashboard.ipynb"
```

Then **Kernel → Restart Kernel and Run All Cells**. The notebook is order-dependent —
each question builds on variables defined by the previous one — so run it top to bottom
rather than cell by cell.

### A note on the HTML parser

The notebook parses scraped pages with `BeautifulSoup(html, "html.parser")` rather than
`html5lib`. Python's built-in parser handles these pages fine and needs no extra
dependency; if you prefer `html5lib`, install it first and swap the argument.

## Files

```
Revenue Data and Building a Dashboard.ipynb   # the complete assignment
README.md
```

## License / attribution

Notebook scaffold and data sources © IBM Corporation, provided as part of the
IBM Skills Network course materials. Solutions are my own.
