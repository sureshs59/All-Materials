# 📊 Matplotlib Programming Exercises — Beginner to Expert

A comprehensive collection of Matplotlib exercises covering everything from basic line plots to advanced 3D visualisations, animations, and publication-quality figures. Every exercise includes a problem description, hints, and a collapsible solution with full working code.

---

## 📋 Table of Contents

- [Level Guide](#-level-guide)
- [🟢 Beginner — Core Plotting Basics](#-beginner--core-plotting-basics)
- [🟡 Elementary — Plot Types & Customisation](#-elementary--plot-types--customisation)
- [🟠 Intermediate — Subplots, Styles & Statistical Plots](#-intermediate--subplots-styles--statistical-plots)
- [🔵 Upper Intermediate — Advanced Chart Types & Annotations](#-upper-intermediate--advanced-chart-types--annotations)
- [🔴 Advanced — 3D Plots, Colormaps & Image Visualisation](#-advanced--3d-plots-colormaps--image-visualisation)
- [⚫ Expert — Animations, Custom Artists & Publication Figures](#-expert--animations-custom-artists--publication-figures)
- [📝 Quick Reference Cheat Sheet](#-quick-reference-cheat-sheet)

---

## 📊 Level Guide

| Level | Label | Description |
|-------|-------|-------------|
| 🟢 **1** | Beginner | Figure/axes basics, line plots, labels, saving |
| 🟡 **2** | Elementary | Bar, scatter, pie, histogram, colours, legends |
| 🟠 **3** | Intermediate | Subplots, twin axes, styles, box plots, violin plots |
| 🔵 **4** | Upper Intermediate | Heatmaps, contour, annotations, broken axes, fill |
| 🔴 **5** | Advanced | 3D surface/scatter, colormaps, image processing |
| ⚫ **6** | Expert | Animations, custom artists, GridSpec, publication-quality |

---

## 🟢 Beginner — Core Plotting Basics

---

### B1 — Your First Line Plot

**Question:**
Create a simple line plot of y = x² for x values from -5 to 5. Add a title, x-label, y-label, and save the figure as a PNG file.

**Hints:**
- Use `np.linspace()` to generate x values
- Use `plt.plot()`, `plt.title()`, `plt.xlabel()`, `plt.ylabel()`
- Use `plt.savefig()` to save and `plt.show()` to display

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np

# Generate data
x = np.linspace(-5, 5, 200)
y = x ** 2

# Create the plot
plt.plot(x, y)

# Add labels and title
plt.title("y = x² (Quadratic Function)")
plt.xlabel("x")
plt.ylabel("y = x²")

# Add a grid for readability
plt.grid(True, linestyle='--', alpha=0.7)

# Save and display
plt.savefig("quadratic.png", dpi=150, bbox_inches='tight')
plt.show()
```

**Output:** A smooth parabola opening upward with grid lines.
</details>

---

### B2 — Figure and Axes Objects (OOP Style)

**Question:**
Re-create the quadratic plot from B1 using the object-oriented API (`fig, ax = plt.subplots()`). Understand the difference between the stateful (`plt.*`) and stateless (`ax.*`) interfaces.

**Hints:**
- `fig, ax = plt.subplots()` returns a Figure and an Axes object
- On `ax` use `.set_title()`, `.set_xlabel()`, `.set_ylabel()`
- The OOP style is preferred for all non-trivial code

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np

x = np.linspace(-5, 5, 200)
y = x ** 2

# OOP style — recommended for all real projects
fig, ax = plt.subplots(figsize=(8, 5))

ax.plot(x, y, color='steelblue', linewidth=2)

ax.set_title("y = x² — OOP style", fontsize=14)
ax.set_xlabel("x", fontsize=12)
ax.set_ylabel("y = x²", fontsize=12)
ax.grid(True, linestyle='--', alpha=0.6)

# Add figure-level description
fig.suptitle("Matplotlib OOP Interface Demo", fontsize=10, color='gray')

plt.tight_layout()
plt.savefig("quadratic_oop.png", dpi=150)
plt.show()

# Key differences:
# plt.title()  vs  ax.set_title()
# plt.xlabel() vs  ax.set_xlabel()
# plt.xlim()   vs  ax.set_xlim()
# plt.legend() vs  ax.legend()
```
</details>

---

### B3 — Multiple Lines on One Plot

**Question:**
Plot sin(x), cos(x), and tan(x) (clipped to [-3, 3]) on the same axes for x ∈ [0, 2π]. Add a legend, different colours and line styles for each.

**Hints:**
- Clip tan values with `np.clip()`
- Pass `label=` to each `ax.plot()` call
- Call `ax.legend()` to show the legend

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np

x = np.linspace(0, 2 * np.pi, 400)

fig, ax = plt.subplots(figsize=(9, 5))

ax.plot(x, np.sin(x),               color='#e74c3c', linewidth=2,
        linestyle='-',  label='sin(x)')
ax.plot(x, np.cos(x),               color='#3498db', linewidth=2,
        linestyle='--', label='cos(x)')
ax.plot(x, np.clip(np.tan(x),-3,3), color='#2ecc71', linewidth=2,
        linestyle=':',  label='tan(x)  [clipped ±3]')

ax.axhline(0, color='black', linewidth=0.8, linestyle='-')  # zero line

ax.set_title("Trigonometric Functions", fontsize=14)
ax.set_xlabel("x (radians)")
ax.set_ylabel("y")
ax.set_xlim(0, 2 * np.pi)
ax.set_ylim(-3.2, 3.2)

# Custom x-tick labels in terms of π
ax.set_xticks([0, np.pi/2, np.pi, 3*np.pi/2, 2*np.pi])
ax.set_xticklabels(['0', 'π/2', 'π', '3π/2', '2π'])

ax.legend(loc='upper right', framealpha=0.9)
ax.grid(True, alpha=0.3)

plt.tight_layout()
plt.show()
```
</details>

---

### B4 — Line Styles, Markers and Colours

**Question:**
Plot five different mathematical functions on separate lines, each with a distinct colour, line style, and marker. Use both named colours and hex codes.

**Hints:**
- Marker options: `'o'`, `'s'`, `'^'`, `'D'`, `'*'`
- Line style options: `'-'`, `'--'`, `'-.'`, `':'`
- Use `markersize`, `markevery` to avoid cluttered markers

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np

x = np.linspace(0, 4, 50)

functions = [
    (x,         'Linear',      'royalblue',  '-',   'o', 5),
    (x**2,      'Quadratic',   '#e74c3c',    '--',  's', 5),
    (x**3/10,   'Cubic/10',    'darkorange', '-.',  '^', 5),
    (np.sqrt(x),'Square root', '#9b59b6',    ':',   'D', 6),
    (np.log1p(x),'log(1+x)',   '#1abc9c',    '-',   '*', 7),
]

fig, ax = plt.subplots(figsize=(10, 6))

for y_vals, label, color, ls, marker, ms in functions:
    ax.plot(x, y_vals, label=label, color=color,
            linestyle=ls, marker=marker, markersize=ms,
            markevery=8, linewidth=1.8)

ax.set_title("Line Styles, Markers & Colours Demo", fontsize=13)
ax.set_xlabel("x")
ax.set_ylabel("f(x)")
ax.legend(loc='upper left', fontsize=10)
ax.grid(True, alpha=0.3)
plt.tight_layout()
plt.show()
```
</details>

---

### B5 — Setting Axis Limits, Ticks and Tick Labels

**Question:**
Create a sine wave and customise the axis limits, tick positions, tick labels, and tick appearance (size, colour, rotation).

**Hints:**
- Use `ax.set_xlim()`, `ax.set_ylim()`
- Use `ax.set_xticks()` and `ax.set_xticklabels()`
- Use `ax.tick_params()` for tick styling

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np

x = np.linspace(0, 4 * np.pi, 300)
y = np.sin(x)

fig, ax = plt.subplots(figsize=(10, 4))

ax.plot(x, y, color='steelblue', linewidth=2)

# Custom axis limits
ax.set_xlim(0, 4 * np.pi)
ax.set_ylim(-1.3, 1.5)

# Custom tick positions and labels (in terms of π)
x_ticks = [0, np.pi/2, np.pi, 3*np.pi/2,
           2*np.pi, 5*np.pi/2, 3*np.pi, 7*np.pi/2, 4*np.pi]
x_labels = ['0', 'π/2', 'π', '3π/2', '2π',
            '5π/2', '3π', '7π/2', '4π']

ax.set_xticks(x_ticks)
ax.set_xticklabels(x_labels, fontsize=11)

# Y tick customisation
ax.set_yticks([-1, -0.5, 0, 0.5, 1])

# Tick appearance
ax.tick_params(axis='x', labelsize=11, labelcolor='#2c3e50',
               length=6, width=1.2, rotation=0)
ax.tick_params(axis='y', labelsize=10, colors='#7f8c8d',
               length=4)

ax.set_title("Sine Wave with Custom Ticks", fontsize=13)
ax.axhline(0, color='gray', linewidth=0.8)
ax.grid(True, alpha=0.3)
plt.tight_layout()
plt.show()
```
</details>

---

### B6 — Saving Figures in Multiple Formats

**Question:**
Create a simple plot and save it in PNG, PDF, SVG, and EPS formats at 300 DPI. Demonstrate key `savefig()` parameters: `dpi`, `bbox_inches`, `facecolor`, `transparent`.

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np

x = np.linspace(0, 10, 100)
y = np.sin(x) * np.exp(-x / 10)

fig, ax = plt.subplots(figsize=(8, 4))
ax.plot(x, y, color='steelblue', linewidth=2)
ax.fill_between(x, y, alpha=0.2, color='steelblue')
ax.set_title("Damped Sine Wave")
ax.set_xlabel("x")
ax.set_ylabel("y")
ax.grid(True, alpha=0.3)
plt.tight_layout()

# Save in different formats
fig.savefig("plot.png",  dpi=300, bbox_inches='tight')
fig.savefig("plot.pdf",  bbox_inches='tight')
fig.savefig("plot.svg",  bbox_inches='tight')

# Transparent background (for slides/web)
fig.savefig("plot_transparent.png", dpi=300,
            bbox_inches='tight', transparent=True)

# White background override
fig.savefig("plot_white.png", dpi=300,
            bbox_inches='tight', facecolor='white')

print("Saved in PNG, PDF, SVG formats")
plt.show()
```
</details>

---

## 🟡 Elementary — Plot Types & Customisation

---

### E1 — Bar Chart: Vertical and Horizontal

**Question:**
Create a vertical bar chart of monthly sales data. Then create a horizontal bar chart of the same data sorted by value. Add value labels on top of each bar.

**Hints:**
- Use `ax.bar()` for vertical, `ax.barh()` for horizontal
- Use `ax.bar_label()` to add value labels automatically

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np

months = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun',
          'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec']
sales  = [42, 55, 61, 48, 73, 85, 92, 88, 67, 74, 80, 95]

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(14, 5))

# ── Vertical bar chart ────────────────────────────────────────────
colors = ['#e74c3c' if s == max(sales) else '#3498db' for s in sales]
bars = ax1.bar(months, sales, color=colors, edgecolor='white', linewidth=0.8)
ax1.bar_label(bars, fmt='%d', padding=3, fontsize=9)
ax1.set_title("Monthly Sales — Vertical", fontsize=13)
ax1.set_xlabel("Month")
ax1.set_ylabel("Sales (units)")
ax1.set_ylim(0, 110)
ax1.grid(axis='y', alpha=0.3)

# ── Horizontal bar chart (sorted) ────────────────────────────────
sorted_idx   = np.argsort(sales)
sorted_months = [months[i] for i in sorted_idx]
sorted_sales  = [sales[i]  for i in sorted_idx]
sorted_colors = ['#e74c3c' if s == max(sales) else '#2ecc71'
                 for s in sorted_sales]

hbars = ax2.barh(sorted_months, sorted_sales,
                 color=sorted_colors, edgecolor='white')
ax2.bar_label(hbars, fmt='%d', padding=3, fontsize=9)
ax2.set_title("Monthly Sales — Horizontal (sorted)", fontsize=13)
ax2.set_xlabel("Sales (units)")
ax2.set_xlim(0, 110)
ax2.grid(axis='x', alpha=0.3)

plt.tight_layout()
plt.show()
```
</details>

---

### E2 — Grouped and Stacked Bar Charts

**Question:**
Create a grouped bar chart comparing quarterly sales for three products. Then create a stacked bar chart of the same data.

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np

quarters  = ['Q1', 'Q2', 'Q3', 'Q4']
product_a = [40,  55,  70,  65]
product_b = [30,  48,  52,  80]
product_c = [25,  35,  45,  55]

x     = np.arange(len(quarters))
width = 0.25

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(14, 5))

# ── Grouped bar chart ─────────────────────────────────────────────
b1 = ax1.bar(x - width, product_a, width, label='Product A',
             color='#3498db', edgecolor='white')
b2 = ax1.bar(x,          product_b, width, label='Product B',
             color='#e74c3c', edgecolor='white')
b3 = ax1.bar(x + width,  product_c, width, label='Product C',
             color='#2ecc71', edgecolor='white')

ax1.set_xticks(x)
ax1.set_xticklabels(quarters)
ax1.set_title("Grouped Bar Chart", fontsize=13)
ax1.set_ylabel("Sales")
ax1.legend()
ax1.grid(axis='y', alpha=0.3)

# ── Stacked bar chart ─────────────────────────────────────────────
ax2.bar(quarters, product_a, label='Product A', color='#3498db')
ax2.bar(quarters, product_b, label='Product B', color='#e74c3c',
        bottom=product_a)
ax2.bar(quarters, product_c, label='Product C', color='#2ecc71',
        bottom=[a+b for a,b in zip(product_a, product_b)])

ax2.set_title("Stacked Bar Chart", fontsize=13)
ax2.set_ylabel("Total Sales")
ax2.legend()
ax2.grid(axis='y', alpha=0.3)

plt.tight_layout()
plt.show()
```
</details>

---

### E3 — Scatter Plot with Size and Colour Encoding

**Question:**
Create a scatter plot of 200 random points where the colour encodes a third variable (temperature) and the point size encodes a fourth variable (population). Add a colourbar.

**Hints:**
- Pass `c=` and `s=` to `ax.scatter()`
- Use `plt.colorbar()` to add a colour scale

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np

np.random.seed(42)
n = 200

x           = np.random.randn(n)
y           = np.random.randn(n)
temperature = np.random.uniform(15, 45, n)   # colour variable
population  = np.random.uniform(50, 2000, n) # size variable

fig, ax = plt.subplots(figsize=(9, 6))

scatter = ax.scatter(x, y,
                     c=temperature,
                     s=population / 10,
                     cmap='RdYlBu_r',
                     alpha=0.7,
                     edgecolors='white',
                     linewidths=0.5)

# Colourbar
cbar = plt.colorbar(scatter, ax=ax)
cbar.set_label("Temperature (°C)", fontsize=11)

# Size legend
for size_val, label in [(50, '500'), (100, '1000'), (200, '2000')]:
    ax.scatter([], [], s=size_val, c='gray', alpha=0.5,
               label=f'Pop: {label}k')

ax.legend(title="Population", loc='upper left',
          framealpha=0.9, fontsize=9)
ax.set_title("Multi-Variable Scatter Plot\n(colour=temperature, size=population)",
             fontsize=12)
ax.set_xlabel("Longitude (normalised)")
ax.set_ylabel("Latitude (normalised)")
ax.grid(True, alpha=0.2)

plt.tight_layout()
plt.show()
```
</details>

---

### E4 — Histogram and KDE Overlay

**Question:**
Generate 1000 random samples from a normal distribution and plot a normalised histogram. Overlay a KDE (kernel density estimate) curve and a theoretical normal distribution curve.

**Hints:**
- Use `density=True` in `ax.hist()` to normalise
- Use `scipy.stats.norm` for the theoretical curve
- Use `scipy.stats.gaussian_kde` for the KDE

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np
from scipy import stats

np.random.seed(42)
data = np.random.normal(loc=50, scale=15, size=1000)

fig, ax = plt.subplots(figsize=(9, 5))

# Histogram (normalised to density)
ax.hist(data, bins=30, density=True, alpha=0.4,
        color='steelblue', edgecolor='white',
        linewidth=0.5, label='Histogram')

# KDE curve
kde = stats.gaussian_kde(data)
x_range = np.linspace(data.min() - 10, data.max() + 10, 300)
ax.plot(x_range, kde(x_range), color='#e74c3c',
        linewidth=2.5, label='KDE')

# Theoretical normal PDF
mu, sigma = data.mean(), data.std()
ax.plot(x_range, stats.norm.pdf(x_range, mu, sigma),
        color='#f39c12', linewidth=2,
        linestyle='--', label=f'Normal PDF (μ={mu:.1f}, σ={sigma:.1f})')

# Vertical lines for mean and ±1σ
ax.axvline(mu, color='black', linewidth=1.2, linestyle='-',  label=f'Mean={mu:.1f}')
ax.axvline(mu - sigma, color='gray', linewidth=1, linestyle=':', label='±1σ')
ax.axvline(mu + sigma, color='gray', linewidth=1, linestyle=':')

ax.set_title("Histogram with KDE and Theoretical Normal Curve", fontsize=13)
ax.set_xlabel("Value")
ax.set_ylabel("Density")
ax.legend(fontsize=10)
ax.grid(True, alpha=0.3)

plt.tight_layout()
plt.show()
```
</details>

---

### E5 — Pie and Donut Charts

**Question:**
Create a pie chart of expense categories. Then create a donut chart of the same data with a total in the centre. Add percentage labels and explode the largest slice.

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np

categories = ['Food', 'Transport', 'Utilities', 'Entertainment', 'Health', 'Other']
values     = [35, 20, 15, 12, 10, 8]
colors     = ['#e74c3c','#3498db','#f39c12','#9b59b6','#1abc9c','#95a5a6']

# Explode the largest slice
explode = [0.05 if v == max(values) else 0 for v in values]

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(14, 6))

# ── Standard pie chart ────────────────────────────────────────────
wedges, texts, autotexts = ax1.pie(
    values, labels=categories, colors=colors,
    autopct='%1.1f%%', explode=explode,
    startangle=90, pctdistance=0.8,
    wedgeprops=dict(edgecolor='white', linewidth=1.5)
)
for t in autotexts:
    t.set_fontsize(9)
    t.set_color('white')
    t.set_fontweight('bold')
ax1.set_title("Expense Breakdown — Pie Chart", fontsize=13)

# ── Donut chart ───────────────────────────────────────────────────
wedges2, _, autotexts2 = ax2.pie(
    values, labels=categories, colors=colors,
    autopct='%1.1f%%', startangle=90,
    pctdistance=0.82,
    wedgeprops=dict(width=0.5, edgecolor='white', linewidth=1.5)
)
for t in autotexts2:
    t.set_fontsize(9)
    t.set_color('white')
    t.set_fontweight('bold')

# Total in the centre
ax2.text(0, 0, f'Total\n{sum(values)}%',
         ha='center', va='center', fontsize=13, fontweight='bold')
ax2.set_title("Expense Breakdown — Donut Chart", fontsize=13)

plt.tight_layout()
plt.show()
```
</details>

---

### E6 — Fill Between: Area Charts

**Question:**
Plot two time series (portfolio A and portfolio B returns) and shade the area between them — green where A outperforms B, red where B outperforms A.

**Hints:**
- Use `ax.fill_between(x, y1, y2, where=condition)`

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np

np.random.seed(7)
days = np.arange(1, 251)
port_a = 100 + np.cumsum(np.random.normal(0.05, 1.2, 250))
port_b = 100 + np.cumsum(np.random.normal(0.03, 1.0, 250))

fig, ax = plt.subplots(figsize=(12, 5))

ax.plot(days, port_a, color='#2ecc71', linewidth=1.8, label='Portfolio A')
ax.plot(days, port_b, color='#e74c3c', linewidth=1.8, label='Portfolio B')

# Fill where A > B (A wins — green)
ax.fill_between(days, port_a, port_b,
                where=(port_a >= port_b),
                alpha=0.3, color='#2ecc71',
                label='A outperforms B')

# Fill where B > A (B wins — red)
ax.fill_between(days, port_a, port_b,
                where=(port_a < port_b),
                alpha=0.3, color='#e74c3c',
                label='B outperforms A')

ax.set_title("Portfolio Performance Comparison", fontsize=13)
ax.set_xlabel("Trading Day")
ax.set_ylabel("Portfolio Value ($)")
ax.legend(fontsize=10)
ax.grid(True, alpha=0.3)
plt.tight_layout()
plt.show()
```
</details>

---

## 🟠 Intermediate — Subplots, Styles & Statistical Plots

---

### I1 — Subplots Grid Layout

**Question:**
Create a 2×3 grid of subplots, each showing a different function (linear, quadratic, cubic, sqrt, log, exponential). Share x-axes within columns.

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np

x = np.linspace(0.1, 5, 200)

functions = [
    (x,           'Linear: f(x) = x',         '#3498db'),
    (x**2,        'Quadratic: f(x) = x²',      '#e74c3c'),
    (x**3,        'Cubic: f(x) = x³',          '#9b59b6'),
    (np.sqrt(x),  'Square root: f(x) = √x',    '#2ecc71'),
    (np.log(x),   'Logarithm: f(x) = ln(x)',   '#f39c12'),
    (np.exp(x/2), 'Exponential: f(x) = eˣ/²', '#1abc9c'),
]

fig, axes = plt.subplots(2, 3, figsize=(13, 7))

for ax, (y, title, color) in zip(axes.flat, functions):
    ax.plot(x, y, color=color, linewidth=2)
    ax.set_title(title, fontsize=10, fontweight='bold')
    ax.grid(True, alpha=0.3)
    ax.set_xlabel('x', fontsize=9)
    ax.spines['top'].set_visible(False)
    ax.spines['right'].set_visible(False)

fig.suptitle("Common Mathematical Functions", fontsize=14, y=1.01)
plt.tight_layout()
plt.show()
```
</details>

---

### I2 — Twin Axes (Dual Y-Axis)

**Question:**
Plot monthly temperature (line, left y-axis in °C) and rainfall (bar, right y-axis in mm) on the same chart using twin axes.

**Hints:**
- Use `ax2 = ax1.twinx()` to create a second y-axis sharing the same x-axis

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np

months = ['Jan','Feb','Mar','Apr','May','Jun',
          'Jul','Aug','Sep','Oct','Nov','Dec']
temp     = [5, 7, 11, 15, 20, 25, 28, 27, 22, 16, 10, 6]
rainfall = [80, 60, 55, 45, 40, 30, 20, 25, 50, 70, 90, 85]

fig, ax1 = plt.subplots(figsize=(11, 5))

# Right axis for rainfall (bars)
ax2 = ax1.twinx()
bars = ax2.bar(months, rainfall, color='#3498db', alpha=0.4,
               label='Rainfall (mm)', width=0.6)
ax2.set_ylabel("Rainfall (mm)", color='#3498db', fontsize=11)
ax2.tick_params(axis='y', labelcolor='#3498db')
ax2.set_ylim(0, 150)

# Left axis for temperature (line — drawn after so it appears on top)
line, = ax1.plot(months, temp, color='#e74c3c', linewidth=2.5,
                 marker='o', markersize=7, label='Temperature (°C)')
ax1.set_ylabel("Temperature (°C)", color='#e74c3c', fontsize=11)
ax1.tick_params(axis='y', labelcolor='#e74c3c')
ax1.set_ylim(0, 40)

# Combined legend
lines = [line, bars]
labels = [l.get_label() for l in lines]
ax1.legend(lines, labels, loc='upper left', framealpha=0.9)

ax1.set_title("Monthly Temperature and Rainfall", fontsize=13)
ax1.grid(True, alpha=0.2)
fig.tight_layout()
plt.show()
```
</details>

---

### I3 — Box Plots and Violin Plots

**Question:**
Generate exam score data for 5 subjects and display them as (a) box plots and (b) violin plots side by side. Add individual data points as a strip on the violin plot.

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np

np.random.seed(42)
subjects = ['Maths', 'Science', 'English', 'History', 'Art']
data = [
    np.random.normal(72, 12, 60),
    np.random.normal(65, 15, 60),
    np.random.normal(78, 8,  60),
    np.random.normal(60, 18, 60),
    np.random.normal(82, 10, 60),
]
colors = ['#3498db','#e74c3c','#2ecc71','#9b59b6','#f39c12']

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(14, 6))

# ── Box plots ──────────────────────────────────────────────────────
bp = ax1.boxplot(data, labels=subjects, patch_artist=True,
                 notch=True, vert=True,
                 medianprops=dict(color='white', linewidth=2))
for patch, color in zip(bp['boxes'], colors):
    patch.set_facecolor(color)
    patch.set_alpha(0.7)
ax1.set_title("Box Plots — Exam Scores by Subject", fontsize=12)
ax1.set_ylabel("Score")
ax1.grid(axis='y', alpha=0.3)

# ── Violin plots with strip ────────────────────────────────────────
parts = ax2.violinplot(data, positions=range(1,6),
                       showmeans=True, showmedians=True)
for pc, color in zip(parts['bodies'], colors):
    pc.set_facecolor(color)
    pc.set_alpha(0.6)

# Add jittered data points
for i, (d, color) in enumerate(zip(data, colors), start=1):
    jitter = np.random.uniform(-0.08, 0.08, len(d))
    ax2.scatter(np.full_like(d, i) + jitter, d,
                alpha=0.3, s=12, color=color)

ax2.set_xticks(range(1, 6))
ax2.set_xticklabels(subjects)
ax2.set_title("Violin Plots with Data Points", fontsize=12)
ax2.set_ylabel("Score")
ax2.grid(axis='y', alpha=0.3)

plt.tight_layout()
plt.show()
```
</details>

---

### I4 — Matplotlib Styles and Themes

**Question:**
Plot the same data using 4 different built-in Matplotlib styles: `'seaborn-v0_8'`, `'ggplot'`, `'dark_background'`, and `'bmh'`. Display them in a 2×2 grid.

**Hints:**
- Use `with plt.style.context('style_name'):` to apply a style temporarily
- Run `plt.style.available` to see all available styles

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np

x = np.linspace(0, 10, 100)
styles = ['seaborn-v0_8', 'ggplot', 'dark_background', 'bmh']
titles = ['seaborn-v0_8', 'ggplot', 'dark_background', 'bmh']

fig, axes = plt.subplots(2, 2, figsize=(12, 8))

for ax, style, title in zip(axes.flat, styles, titles):
    with plt.style.context(style):
        # Redraw in the context — we must draw manually
        ax.plot(x, np.sin(x),   label='sin(x)', linewidth=2)
        ax.plot(x, np.cos(x),   label='cos(x)', linewidth=2)
        ax.plot(x, np.sin(2*x), label='sin(2x)',linewidth=2)
        ax.set_title(f"Style: '{title}'", fontsize=11)
        ax.legend(fontsize=8)
        ax.grid(True)

fig.suptitle("Matplotlib Built-in Styles Comparison", fontsize=14)
plt.tight_layout()
plt.show()

# List all available styles:
# print(plt.style.available)
```
</details>

---

### I5 — Step Plot, Stem Plot and Error Bars

**Question:**
Create three subplots: (1) a step plot for digital signal data, (2) a stem plot for a discrete signal, and (3) a line plot with asymmetric error bars.

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np

np.random.seed(0)
fig, (ax1, ax2, ax3) = plt.subplots(1, 3, figsize=(15, 5))

# ── Step plot (digital signal) ─────────────────────────────────────
t_step = np.arange(0, 10, 0.5)
signal = np.where(np.sin(t_step) > 0, 1, 0)
ax1.step(t_step, signal, where='post', color='#3498db', linewidth=2)
ax1.fill_between(t_step, signal, step='post', alpha=0.3, color='#3498db')
ax1.set_ylim(-0.2, 1.4)
ax1.set_title("Step Plot — Digital Signal", fontsize=11)
ax1.set_xlabel("Time")
ax1.set_ylabel("Voltage (0/1)")
ax1.grid(True, alpha=0.3)

# ── Stem plot (discrete signal) ────────────────────────────────────
t_stem = np.linspace(0, 4*np.pi, 20)
y_stem = np.sin(t_stem)
markerline, stemlines, baseline = ax2.stem(t_stem, y_stem)
plt.setp(stemlines,   color='#9b59b6', linewidth=1.5)
plt.setp(markerline,  color='#9b59b6', markersize=8)
plt.setp(baseline,    color='gray',    linewidth=0.8)
ax2.set_title("Stem Plot — Discrete Signal", fontsize=11)
ax2.set_xlabel("Time")
ax2.grid(True, alpha=0.3)

# ── Error bars ─────────────────────────────────────────────────────
x_err  = np.arange(1, 8)
y_err  = np.array([2.3, 3.1, 2.8, 4.5, 3.9, 5.2, 4.8])
yerr_lower = np.array([0.3, 0.4, 0.2, 0.5, 0.3, 0.6, 0.4])
yerr_upper = np.array([0.5, 0.3, 0.4, 0.4, 0.5, 0.3, 0.6])

ax3.errorbar(x_err, y_err,
             yerr=[yerr_lower, yerr_upper],
             fmt='o-', color='#e74c3c',
             ecolor='#c0392b', elinewidth=2,
             capsize=5, capthick=2,
             linewidth=1.8, markersize=7)
ax3.set_title("Line Plot with Asymmetric Error Bars", fontsize=11)
ax3.set_xlabel("Measurement")
ax3.set_ylabel("Value")
ax3.grid(True, alpha=0.3)

plt.tight_layout()
plt.show()
```
</details>

---

## 🔵 Upper Intermediate — Advanced Chart Types & Annotations

---

### U1 — Heatmap with Annotations

**Question:**
Create a correlation heatmap of a 6×6 matrix with cell value annotations, a colourbar, and a colour scheme appropriate for correlation data (diverging around 0).

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np

# Generate a symmetric correlation matrix
np.random.seed(42)
variables = ['Revenue', 'Cost', 'Profit', 'Sales', 'Employees', 'Growth']
raw = np.random.randn(100, 6)
corr = np.corrcoef(raw.T)

fig, ax = plt.subplots(figsize=(8, 6))

# Draw heatmap
im = ax.imshow(corr, cmap='RdYlGn', vmin=-1, vmax=1, aspect='auto')

# Add colourbar
cbar = plt.colorbar(im, ax=ax, fraction=0.046, pad=0.04)
cbar.set_label("Correlation Coefficient", fontsize=10)

# Annotate each cell
for i in range(len(variables)):
    for j in range(len(variables)):
        val = corr[i, j]
        text_color = 'white' if abs(val) > 0.6 else 'black'
        ax.text(j, i, f'{val:.2f}',
                ha='center', va='center',
                fontsize=9, color=text_color,
                fontweight='bold' if abs(val) > 0.6 else 'normal')

# Tick labels
ax.set_xticks(range(len(variables)))
ax.set_yticks(range(len(variables)))
ax.set_xticklabels(variables, rotation=45, ha='right')
ax.set_yticklabels(variables)

ax.set_title("Correlation Heatmap", fontsize=13, pad=15)
plt.tight_layout()
plt.show()
```
</details>

---

### U2 — Contour and Filled Contour Plots

**Question:**
Create a filled contour plot (`contourf`) and a contour line overlay (`contour`) of the function z = sin(x)·cos(y). Add contour labels and a colourbar.

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np

x = np.linspace(-3, 3, 200)
y = np.linspace(-3, 3, 200)
X, Y = np.meshgrid(x, y)
Z = np.sin(X) * np.cos(Y)

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(13, 5))

# ── Filled contour ─────────────────────────────────────────────────
cf = ax1.contourf(X, Y, Z, levels=20, cmap='RdBu_r')
plt.colorbar(cf, ax=ax1, label='z = sin(x)·cos(y)')
cs = ax1.contour(X, Y, Z, levels=10, colors='black',
                 linewidths=0.5, alpha=0.5)
ax1.clabel(cs, fmt='%.1f', fontsize=7)
ax1.set_title("Filled Contour (contourf)", fontsize=12)
ax1.set_xlabel("x")
ax1.set_ylabel("y")

# ── Contour lines only with colour coding ──────────────────────────
cs2 = ax2.contour(X, Y, Z, levels=15, cmap='plasma')
ax2.clabel(cs2, fmt='%.1f', fontsize=7, inline=True)
plt.colorbar(cs2, ax=ax2, label='z value')
ax2.set_title("Contour Lines with Labels", fontsize=12)
ax2.set_xlabel("x")

plt.tight_layout()
plt.show()
```
</details>

---

### U3 — Annotations, Arrows and Text Boxes

**Question:**
Create a stock price line chart and annotate key events: a peak (with an arrow), a trough (with an arrow), and a news event (with a text box and shaded region).

**Hints:**
- Use `ax.annotate()` with `arrowprops` for arrows
- Use `ax.axvspan()` for shaded regions
- Use `bbox=dict(...)` in `ax.text()` for a text box

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np

np.random.seed(10)
days   = np.arange(250)
price  = 100 + np.cumsum(np.random.normal(0.1, 1.5, 250))

peak_idx   = price.argmax()
trough_idx = price.argmin()
event_idx  = 120

fig, ax = plt.subplots(figsize=(13, 5))

ax.plot(days, price, color='#2c3e50', linewidth=1.5, label='Stock price')
ax.fill_between(days, price, price.min() - 5, alpha=0.1, color='#3498db')

# Peak annotation with arrow
ax.annotate(f"Peak\n${price[peak_idx]:.2f}",
            xy=(peak_idx, price[peak_idx]),
            xytext=(peak_idx - 40, price[peak_idx] + 8),
            fontsize=10, color='#27ae60', fontweight='bold',
            arrowprops=dict(arrowstyle='->', color='#27ae60',
                            lw=1.8, connectionstyle='arc3,rad=0.2'))

# Trough annotation with arrow
ax.annotate(f"Trough\n${price[trough_idx]:.2f}",
            xy=(trough_idx, price[trough_idx]),
            xytext=(trough_idx + 20, price[trough_idx] - 10),
            fontsize=10, color='#e74c3c', fontweight='bold',
            arrowprops=dict(arrowstyle='->', color='#e74c3c',
                            lw=1.8, connectionstyle='arc3,rad=-0.2'))

# Shaded event region + text box
ax.axvspan(event_idx - 3, event_idx + 3, alpha=0.2,
           color='#f39c12', label='News event')
ax.text(event_idx, price.max() - 5, "Earnings\nReport",
        ha='center', fontsize=9, color='#854d0b',
        bbox=dict(boxstyle='round,pad=0.4', facecolor='#fef3c7',
                  edgecolor='#f39c12', linewidth=1.2))

ax.set_title("Stock Price with Annotations", fontsize=13)
ax.set_xlabel("Trading Day")
ax.set_ylabel("Price ($)")
ax.legend()
ax.grid(True, alpha=0.2)
plt.tight_layout()
plt.show()
```
</details>

---

### U4 — Polar Plot and Radar Chart

**Question:**
Create (a) a polar line plot of a rose curve and (b) a radar chart comparing skill levels across 6 categories for two candidates.

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np

fig = plt.figure(figsize=(13, 6))

# ── Polar rose curve ───────────────────────────────────────────────
ax1 = fig.add_subplot(121, projection='polar')
theta = np.linspace(0, 2*np.pi, 1000)
r = np.abs(np.cos(3 * theta))   # 3-petal rose

ax1.plot(theta, r, color='#e74c3c', linewidth=1.8)
ax1.fill(theta, r, alpha=0.2, color='#e74c3c')
ax1.set_title("Rose Curve: r = |cos(3θ)|", pad=15, fontsize=11)

# ── Radar chart ────────────────────────────────────────────────────
categories = ['Python', 'ML', 'SQL', 'Visualisation', 'Statistics', 'Communication']
N = len(categories)
angles = np.linspace(0, 2*np.pi, N, endpoint=False).tolist()
angles += angles[:1]   # close the polygon

candidate_a = [9, 7, 8, 9, 6, 8]
candidate_b = [6, 9, 7, 6, 9, 7]
for lst in [candidate_a, candidate_b]:
    lst += lst[:1]

ax2 = fig.add_subplot(122, projection='polar')
ax2.plot(angles, candidate_a, 'o-', linewidth=2,
         color='#3498db', label='Candidate A')
ax2.fill(angles, candidate_a, alpha=0.2, color='#3498db')

ax2.plot(angles, candidate_b, 's-', linewidth=2,
         color='#e74c3c', label='Candidate B')
ax2.fill(angles, candidate_b, alpha=0.2, color='#e74c3c')

ax2.set_xticks(angles[:-1])
ax2.set_xticklabels(categories, fontsize=9)
ax2.set_ylim(0, 10)
ax2.set_yticks([2, 4, 6, 8, 10])
ax2.set_yticklabels(['2','4','6','8','10'], fontsize=7)
ax2.legend(loc='upper right', bbox_to_anchor=(1.3, 1.1))
ax2.set_title("Skill Radar Chart", pad=15, fontsize=11)

plt.tight_layout()
plt.show()
```
</details>

---

### U5 — Log Scale, Broken Axis and Inset Axes

**Question:**
Demonstrate (a) log scale on both axes, (b) a broken y-axis to hide a large gap in data, and (c) an inset zoom-in axes using `ax.inset_axes()`.

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np
from mpl_toolkits.axes_grid1.inset_locator import mark_inset

np.random.seed(5)
fig, axes = plt.subplots(1, 3, figsize=(15, 5))

# ── Log-log scale ──────────────────────────────────────────────────
x_log = np.logspace(0, 4, 100)
y_log = x_log ** 2.3
axes[0].loglog(x_log, y_log, color='steelblue', linewidth=2)
axes[0].set_title("Log-Log Scale", fontsize=11)
axes[0].set_xlabel("x (log scale)")
axes[0].set_ylabel("y (log scale)")
axes[0].grid(True, which='both', alpha=0.3)

# ── Broken y-axis ──────────────────────────────────────────────────
# Simulate data with a big gap
x_brk = np.arange(10)
y_brk = [2, 3, 2.5, 3.5, 3, 150, 160, 155, 165, 158]

ax_top = axes[1]
ax_bot = ax_top.twinx()

ax_top.set_ylim(130, 175)
ax_top.plot(x_brk, y_brk, 'o-', color='#e74c3c')
ax_top.set_ylabel("High values", color='#e74c3c')
ax_top.tick_params(axis='y', labelcolor='#e74c3c')

ax_bot.set_ylim(0, 6)
ax_bot.plot(x_brk, y_brk, 'o-', color='#3498db')
ax_bot.set_ylabel("Low values", color='#3498db')
ax_bot.tick_params(axis='y', labelcolor='#3498db')

axes[1].set_title("Dual Y-axis (broken gap)", fontsize=11)
axes[1].set_xlabel("Sample")

# ── Inset zoom ─────────────────────────────────────────────────────
x_main = np.linspace(0, 10, 300)
y_main = np.sin(x_main) + 0.05*np.random.randn(300)

axes[2].plot(x_main, y_main, color='#2c3e50', linewidth=1.2)
axes[2].set_title("Main Plot with Inset Zoom", fontsize=11)
axes[2].set_xlabel("x")
axes[2].grid(True, alpha=0.3)

# Inset axes
axins = axes[2].inset_axes([0.55, 0.55, 0.4, 0.35])
axins.plot(x_main, y_main, color='#2c3e50', linewidth=1.5)
axins.set_xlim(4.5, 6.5)
axins.set_ylim(-1.2, 1.2)
axins.set_xticklabels([])
axins.set_yticklabels([])
axes[2].indicate_inset_zoom(axins, edgecolor='#e74c3c', linewidth=1.5)

plt.tight_layout()
plt.show()
```
</details>

---

## 🔴 Advanced — 3D Plots, Colormaps & Image Visualisation

---

### A1 — 3D Surface Plot

**Question:**
Create a 3D surface plot of z = sin(√(x²+y²)) for x,y ∈ [-6, 6]. Apply a colourmap, add lighting effect, and add 2D contour projections on the floor.

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np
from mpl_toolkits.mplot3d import Axes3D

x = np.linspace(-6, 6, 120)
y = np.linspace(-6, 6, 120)
X, Y = np.meshgrid(x, y)
R = np.sqrt(X**2 + Y**2) + 1e-10
Z = np.sin(R) / R      # sinc-like surface

fig = plt.figure(figsize=(12, 7))
ax  = fig.add_subplot(111, projection='3d')

# Surface plot
surf = ax.plot_surface(X, Y, Z, cmap='viridis',
                       alpha=0.85, rstride=2, cstride=2,
                       antialiased=True)
fig.colorbar(surf, ax=ax, shrink=0.5, aspect=12, label='z value')

# Contour projections on the floor
offset = Z.min() - 0.3
ax.contourf(X, Y, Z, zdir='z', offset=offset,
            levels=15, cmap='viridis', alpha=0.4)

ax.set_zlim(offset, Z.max() + 0.1)
ax.set_title("3D Surface: z = sin(r)/r", fontsize=13, pad=12)
ax.set_xlabel("x")
ax.set_ylabel("y")
ax.set_zlabel("z")

# Better viewing angle
ax.view_init(elev=30, azim=225)
plt.tight_layout()
plt.show()
```
</details>

---

### A2 — 3D Scatter and Wire Frame

**Question:**
Create a 3D scatter plot of 300 random points coloured by their z-value. Then create a wireframe plot of a torus (donut shape).

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np

fig = plt.figure(figsize=(13, 6))

# ── 3D Scatter ─────────────────────────────────────────────────────
ax1 = fig.add_subplot(121, projection='3d')
np.random.seed(42)
n = 300
x = np.random.randn(n)
y = np.random.randn(n)
z = x**2 + y**2 + np.random.randn(n) * 0.5

sc = ax1.scatter(x, y, z, c=z, cmap='plasma',
                 s=40, alpha=0.8, edgecolors='none')
plt.colorbar(sc, ax=ax1, shrink=0.5, label='z value')
ax1.set_title("3D Scatter Plot", fontsize=12)
ax1.set_xlabel("x"); ax1.set_ylabel("y"); ax1.set_zlabel("z")

# ── Torus Wireframe ────────────────────────────────────────────────
ax2 = fig.add_subplot(122, projection='3d')

R, r = 3, 1   # major and minor radius
u = np.linspace(0, 2*np.pi, 40)
v = np.linspace(0, 2*np.pi, 40)
U, V = np.meshgrid(u, v)

Xt = (R + r*np.cos(V)) * np.cos(U)
Yt = (R + r*np.cos(V)) * np.sin(U)
Zt = r * np.sin(V)

ax2.plot_wireframe(Xt, Yt, Zt, rstride=2, cstride=2,
                   color='steelblue', linewidth=0.5, alpha=0.8)
ax2.set_title("Torus Wireframe", fontsize=12)
ax2.set_xlabel("x"); ax2.set_ylabel("y"); ax2.set_zlabel("z")
ax2.view_init(elev=25, azim=45)

plt.tight_layout()
plt.show()
```
</details>

---

### A3 — Colourmap Exploration and Custom Colourmap

**Question:**
Display 6 different colourmap families (sequential, diverging, cyclic, qualitative). Then create a custom colourmap by blending two colours.

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import matplotlib.colors as mcolors
import numpy as np

# ── Colourmap showcase ─────────────────────────────────────────────
cmap_groups = {
    'Sequential':  ['viridis', 'plasma', 'magma'],
    'Diverging':   ['RdYlGn', 'coolwarm', 'bwr'],
    'Perceptual':  ['cividis', 'turbo', 'inferno'],
    'Cyclic':      ['hsv', 'twilight', 'twilight_shifted'],
}

gradient = np.linspace(0, 1, 256).reshape(1, -1)

fig, axes = plt.subplots(len(cmap_groups), 3, figsize=(11, 8))

for row_axes, (group, cmaps) in zip(axes, cmap_groups.items()):
    for ax, cmap in zip(row_axes, cmaps):
        ax.imshow(gradient, aspect='auto', cmap=cmap)
        ax.set_title(f'{cmap}\n({group})', fontsize=8)
        ax.axis('off')

fig.suptitle("Matplotlib Colourmap Families", fontsize=13, y=1.01)
plt.tight_layout()
plt.show()

# ── Custom colourmap ───────────────────────────────────────────────
colors_custom = ['#1a1a2e', '#16213e', '#0f3460', '#e94560']
custom_cmap   = mcolors.LinearSegmentedColormap.from_list(
    'midnight', colors_custom, N=256)

x2 = np.linspace(-3, 3, 200)
y2 = np.linspace(-3, 3, 200)
X2, Y2 = np.meshgrid(x2, y2)
Z2 = np.sin(X2**2 + Y2**2)

fig2, ax2 = plt.subplots(figsize=(7, 5))
im = ax2.imshow(Z2, cmap=custom_cmap, origin='lower',
                extent=[-3,3,-3,3], aspect='auto')
plt.colorbar(im, ax=ax2, label='z value')
ax2.set_title("Custom 'midnight' Colourmap", fontsize=12)
plt.tight_layout()
plt.show()
```
</details>

---

### A4 — Image Display and Manipulation

**Question:**
Load (or generate) a synthetic image array. Apply and compare: grayscale conversion, histogram equalisation, Gaussian blur, Sobel edge detection, and false-colour mapping.

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import numpy as np
from scipy import ndimage

# Generate a synthetic image (gradient + circles)
size = 256
x_img, y_img = np.meshgrid(np.linspace(-4,4,size), np.linspace(-4,4,size))
img = (np.sin(x_img**2 + y_img**2) * 127.5 + 127.5).astype(np.uint8)

# Grayscale normalised to [0, 1]
img_f = img / 255.0

# Gaussian blur
blurred = ndimage.gaussian_filter(img_f, sigma=3)

# Sobel edge detection
sx = ndimage.sobel(img_f, axis=0)
sy = ndimage.sobel(img_f, axis=1)
edges = np.hypot(sx, sy)
edges = edges / edges.max()

# Histogram equalisation
hist, bins = np.histogram(img.ravel(), 256, [0, 256])
cdf = hist.cumsum()
cdf_norm = (cdf - cdf.min()) * 255 / (cdf.max() - cdf.min())
img_eq = cdf_norm[img].astype(np.uint8)

fig, axes = plt.subplots(2, 3, figsize=(14, 8))

axes[0,0].imshow(img,     cmap='gray');             axes[0,0].set_title("Original")
axes[0,1].imshow(img_eq,  cmap='gray');             axes[0,1].set_title("Histogram Equalised")
axes[0,2].imshow(blurred, cmap='gray');             axes[0,2].set_title("Gaussian Blur (σ=3)")
axes[1,0].imshow(edges,   cmap='gray');             axes[1,0].set_title("Sobel Edge Detection")
axes[1,1].imshow(img,     cmap='hot');              axes[1,1].set_title("False-colour: 'hot'")
axes[1,2].imshow(img,     cmap='RdYlBu');           axes[1,2].set_title("False-colour: 'RdYlBu'")

for ax in axes.flat:
    ax.axis('off')

fig.suptitle("Image Processing with Matplotlib + SciPy", fontsize=13)
plt.tight_layout()
plt.show()
```
</details>

---

## ⚫ Expert — Animations, Custom Artists & Publication Figures

---

### X1 — Animation: Sine Wave in Motion

**Question:**
Create an animated sine wave where the phase shifts with each frame. Save it as a GIF. Demonstrate both `FuncAnimation` and `ArtistAnimation`.

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import matplotlib.animation as animation
import numpy as np

x = np.linspace(0, 4 * np.pi, 400)

fig, ax = plt.subplots(figsize=(9, 4))
ax.set_xlim(0, 4*np.pi)
ax.set_ylim(-1.5, 1.5)
ax.set_title("Animated Sine Wave", fontsize=13)
ax.set_xlabel("x")
ax.set_ylabel("sin(x + phase)")
ax.grid(True, alpha=0.3)

line,   = ax.plot([], [], color='#e74c3c', linewidth=2.5)
phase_text = ax.text(0.02, 0.92, '', transform=ax.transAxes,
                     fontsize=10, color='#2c3e50')

def init():
    line.set_data([], [])
    phase_text.set_text('')
    return line, phase_text

def animate(frame):
    phase = frame * 0.15
    y     = np.sin(x + phase)
    line.set_data(x, y)
    phase_text.set_text(f'Phase: {phase:.2f} rad')
    return line, phase_text

ani = animation.FuncAnimation(
    fig, animate, init_func=init,
    frames=80, interval=50, blit=True
)

# Save as GIF (requires Pillow: pip install Pillow)
ani.save('sine_animation.gif', writer='pillow', fps=20, dpi=100)

# Save as MP4 (requires ffmpeg)
# ani.save('sine_animation.mp4', writer='ffmpeg', fps=20, dpi=150)

plt.show()
print("Animation saved as sine_animation.gif")
```
</details>

---

### X2 — Animation: Particle Simulation

**Question:**
Animate 50 bouncing particles inside a box. Each particle has a random position, velocity, and colour. Particles bounce off the walls.

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import matplotlib.animation as animation
import numpy as np

np.random.seed(42)
N = 50

# Initial positions, velocities, colours
pos  = np.random.uniform(0.05, 0.95, (N, 2))
vel  = np.random.uniform(-0.02, 0.02, (N, 2))
cols = plt.cm.hsv(np.random.rand(N))
sizes = np.random.uniform(20, 100, N)

fig, ax = plt.subplots(figsize=(7, 7))
ax.set_xlim(0, 1)
ax.set_ylim(0, 1)
ax.set_facecolor('#0a0a1a')
ax.set_xticks([]); ax.set_yticks([])
ax.set_title("Bouncing Particles", fontsize=13, color='white')
fig.patch.set_facecolor('#0a0a1a')

scat = ax.scatter(pos[:, 0], pos[:, 1],
                  c=cols, s=sizes, alpha=0.85, zorder=5)
frame_text = ax.text(0.02, 0.97, '', transform=ax.transAxes,
                     color='white', fontsize=9)

def animate(frame):
    global pos, vel
    pos += vel

    # Bounce off walls
    hit_wall = (pos < 0.02) | (pos > 0.98)
    vel[hit_wall] *= -1
    pos = np.clip(pos, 0.02, 0.98)

    scat.set_offsets(pos)
    frame_text.set_text(f'Frame: {frame}')
    return scat, frame_text

ani = animation.FuncAnimation(
    fig, animate, frames=150,
    interval=30, blit=True
)

ani.save('particles.gif', writer='pillow', fps=30, dpi=100)
plt.show()
```
</details>

---

### X3 — GridSpec: Complex Multi-Panel Layout

**Question:**
Create a publication-quality figure with a complex layout using `GridSpec`: a large main plot (top), two smaller plots (bottom-left and bottom-right), and a narrow colourbar panel on the far right.

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import matplotlib.gridspec as gridspec
import numpy as np

np.random.seed(42)
fig = plt.figure(figsize=(14, 8))

# GridSpec: 2 rows × 4 columns with width ratios
gs = gridspec.GridSpec(2, 4,
                       width_ratios=[3, 2, 2, 0.15],
                       height_ratios=[1.5, 1],
                       hspace=0.35, wspace=0.35)

# ── Main plot (top, spans first 3 columns) ─────────────────────────
ax_main = fig.add_subplot(gs[0, :3])
t = np.linspace(0, 10, 500)
ax_main.plot(t, np.sin(t) * np.exp(-t/8), color='#2c3e50', linewidth=2)
ax_main.fill_between(t, np.sin(t)*np.exp(-t/8), alpha=0.1, color='#3498db')
ax_main.set_title("(a) Damped Oscillation — Main Panel", fontsize=12)
ax_main.set_xlabel("Time (s)")
ax_main.set_ylabel("Amplitude")
ax_main.grid(True, alpha=0.3)

# ── Bottom-left: histogram ─────────────────────────────────────────
ax_hist = fig.add_subplot(gs[1, 0])
data_hist = np.random.normal(0, 1, 500)
ax_hist.hist(data_hist, bins=25, color='#9b59b6', alpha=0.8, edgecolor='white')
ax_hist.set_title("(b) Distribution", fontsize=11)
ax_hist.set_xlabel("Value")
ax_hist.set_ylabel("Count")

# ── Bottom-centre: scatter ─────────────────────────────────────────
ax_scat = fig.add_subplot(gs[1, 1])
sc_x = np.random.randn(100)
sc_y = sc_x * 0.6 + np.random.randn(100) * 0.5
sc_z = sc_x**2 + sc_y**2
scat = ax_scat.scatter(sc_x, sc_y, c=sc_z, cmap='viridis',
                       s=30, alpha=0.7, edgecolors='none')
ax_scat.set_title("(c) Scatter", fontsize=11)
ax_scat.set_xlabel("x")
ax_scat.set_ylabel("y")

# ── Bottom-right: heatmap ──────────────────────────────────────────
ax_heat = fig.add_subplot(gs[1, 2])
heat_data = np.random.rand(8, 8)
im = ax_heat.imshow(heat_data, cmap='plasma', aspect='auto')
ax_heat.set_title("(d) Heatmap", fontsize=11)

# ── Far-right colourbar panel (spans both rows) ────────────────────
ax_cb = fig.add_subplot(gs[:, 3])
plt.colorbar(im, cax=ax_cb, label='Intensity')

fig.suptitle("Complex GridSpec Layout — Publication Figure",
             fontsize=14, y=1.01)
plt.savefig("gridspec_figure.png", dpi=200, bbox_inches='tight')
plt.show()
```
</details>

---

### X4 — Custom Artist: Drawing Shapes and Patches

**Question:**
Use `matplotlib.patches` to draw custom geometric shapes: circles, rectangles, ellipses, polygons, wedges, and fancy arrows. Build a simple architecture diagram using only patches.

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import matplotlib.patches as mpatches
import matplotlib.patheffects as pe
import numpy as np

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(14, 6))

# ── Patch showcase ─────────────────────────────────────────────────
patches_demo = [
    mpatches.Circle((0.2, 0.75),   0.12,  fc='#3498db', ec='#2980b9', lw=2),
    mpatches.Rectangle((0.42, 0.63), 0.24, 0.24, fc='#e74c3c', ec='#c0392b', lw=2),
    mpatches.Ellipse((0.8, 0.75),  0.25, 0.15, fc='#2ecc71', ec='#27ae60', lw=2),
    mpatches.RegularPolygon((0.2, 0.3), 6, radius=0.12,
                             fc='#9b59b6', ec='#8e44ad', lw=2),
    mpatches.Wedge((0.55, 0.3), 0.15, 30, 210,
                   fc='#f39c12', ec='#d68910', lw=2),
    mpatches.FancyArrowPatch((0.7, 0.2), (0.95, 0.4),
                              arrowstyle='->', mutation_scale=20,
                              color='#1abc9c', lw=2),
]

labels = ['Circle', 'Rectangle', 'Ellipse', 'Hexagon', 'Wedge', 'Arrow']
for patch, label in zip(patches_demo, labels):
    ax1.add_patch(patch)
    cx = patch.center[0] if hasattr(patch, 'center') else \
         patch.xy[0] + 0.12 if hasattr(patch, 'xy') else 0.55
    cy = patch.center[1] if hasattr(patch, 'center') else \
         patch.xy[1] - 0.1 if hasattr(patch, 'xy') else 0.15
    ax1.text(cx, cy, label, ha='center', va='center', fontsize=8, fontweight='bold')

ax1.set_xlim(0, 1); ax1.set_ylim(0, 1)
ax1.set_aspect('equal')
ax1.set_title("Matplotlib Patches Demo", fontsize=12)
ax1.axis('off')

# ── Simple system diagram using patches ───────────────────────────
def draw_box(ax, x, y, w, h, label, color='#3498db'):
    rect = mpatches.FancyBboxPatch((x - w/2, y - h/2), w, h,
                                    boxstyle='round,pad=0.02',
                                    fc=color, ec='white', lw=1.5, alpha=0.9)
    ax.add_patch(rect)
    ax.text(x, y, label, ha='center', va='center',
            fontsize=9, color='white', fontweight='bold')

def draw_arrow(ax, x1, y1, x2, y2):
    ax.annotate('', xy=(x2, y2), xytext=(x1, y1),
                arrowprops=dict(arrowstyle='->', color='#7f8c8d', lw=1.5))

ax2.set_xlim(0, 10); ax2.set_ylim(0, 7)
ax2.axis('off')
ax2.set_title("Simple System Architecture Diagram", fontsize=12)
ax2.set_facecolor('#f8f9fa')
fig.patch.set_facecolor('#f8f9fa')

draw_box(ax2, 5, 6,   3, 0.8, "User Browser",       '#2c3e50')
draw_box(ax2, 5, 4.5, 3, 0.8, "API Gateway",         '#8e44ad')
draw_box(ax2, 2.5, 3, 3, 0.8, "Auth Service",        '#c0392b')
draw_box(ax2, 7.5, 3, 3, 0.8, "Data Service",        '#27ae60')
draw_box(ax2, 5, 1.5, 3, 0.8, "PostgreSQL Database", '#2980b9')

draw_arrow(ax2, 5, 5.6,  5, 4.9)
draw_arrow(ax2, 3.5, 4.1, 2.5, 3.4)
draw_arrow(ax2, 6.5, 4.1, 7.5, 3.4)
draw_arrow(ax2, 2.5, 2.6, 4, 1.9)
draw_arrow(ax2, 7.5, 2.6, 6, 1.9)

plt.tight_layout()
plt.show()
```
</details>

---

### X5 — Publication-Quality Figure: Full Pipeline

**Question:**
Produce a complete publication-ready figure with: custom `rcParams` for fonts and sizes, tight layout, proper figure size for a journal column width, LaTeX-style math in labels, sub-panel labels (a, b, c), and export at 300 DPI.

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import matplotlib.gridspec as gridspec
import numpy as np
from scipy import stats

# ── Publication rcParams ───────────────────────────────────────────
plt.rcParams.update({
    'font.family':        'serif',
    'font.size':          10,
    'axes.titlesize':     11,
    'axes.labelsize':     10,
    'xtick.labelsize':    9,
    'ytick.labelsize':    9,
    'legend.fontsize':    9,
    'figure.dpi':         150,
    'axes.spines.top':    False,
    'axes.spines.right':  False,
    'axes.grid':          True,
    'grid.alpha':         0.25,
    'grid.linestyle':     '--',
    'lines.linewidth':    1.8,
    'lines.markersize':   5,
})

np.random.seed(42)

# ── Data ───────────────────────────────────────────────────────────
x     = np.linspace(0, 2*np.pi, 300)
y1    = np.sin(x) * np.exp(-x/8)
y2    = np.cos(x) * np.exp(-x/8)

n_samples  = 80
x_scatter  = np.random.randn(n_samples)
y_scatter  = 1.5 * x_scatter + np.random.randn(n_samples) * 0.8
slope, intercept, r, p, se = stats.linregress(x_scatter, y_scatter)
x_fit      = np.linspace(-3, 3, 100)

hist_data  = np.random.gamma(2, 2, 400)

# ── Figure setup: journal two-column width = 3.5 in × 2 rows ──────
fig = plt.figure(figsize=(7.2, 7.2))   # double-column width
gs  = gridspec.GridSpec(2, 2, hspace=0.42, wspace=0.38)

# ── Panel (a): Damped oscillation ─────────────────────────────────
ax_a = fig.add_subplot(gs[0, :])   # full-width top
ax_a.plot(x, y1, color='#2c3e50', label=r'$\sin(x)\,e^{-x/8}$')
ax_a.plot(x, y2, color='#e74c3c', linestyle='--',
          label=r'$\cos(x)\,e^{-x/8}$')
ax_a.fill_between(x, y1, y2, alpha=0.1, color='#3498db')
ax_a.set_xlabel(r'$x$ (radians)')
ax_a.set_ylabel(r'Amplitude $f(x)$')
ax_a.legend(loc='upper right')
ax_a.set_xlim(0, 2*np.pi)
ax_a.text(-0.07, 1.05, '(a)', transform=ax_a.transAxes,
          fontsize=12, fontweight='bold')

# ── Panel (b): Scatter with regression ────────────────────────────
ax_b = fig.add_subplot(gs[1, 0])
ax_b.scatter(x_scatter, y_scatter, s=20, alpha=0.6,
             color='#8e44ad', edgecolors='none')
ax_b.plot(x_fit, slope*x_fit + intercept, 'k-', linewidth=1.5,
          label=f'$r={r:.2f}$, $p={p:.3f}$')
ax_b.set_xlabel(r'Predictor $X$')
ax_b.set_ylabel(r'Response $Y$')
ax_b.legend(loc='upper left', handlelength=1)
ax_b.text(-0.15, 1.05, '(b)', transform=ax_b.transAxes,
          fontsize=12, fontweight='bold')

# ── Panel (c): Gamma distribution with theoretical PDF ────────────
ax_c = fig.add_subplot(gs[1, 1])
ax_c.hist(hist_data, bins=30, density=True, alpha=0.5,
          color='#27ae60', edgecolor='white', linewidth=0.5)
xg  = np.linspace(0, hist_data.max(), 200)
ax_c.plot(xg, stats.gamma.pdf(xg, a=2, scale=2),
          color='#1a5276', linewidth=2,
          label=r'Gamma$(\alpha=2,\,\beta=2)$')
ax_c.set_xlabel(r'Value $x$')
ax_c.set_ylabel(r'Probability density')
ax_c.legend(loc='upper right')
ax_c.text(-0.15, 1.05, '(c)', transform=ax_c.transAxes,
          fontsize=12, fontweight='bold')

fig.suptitle("Publication-Ready Figure — Matplotlib Expert Demo",
             fontsize=11, y=1.01)

plt.savefig("publication_figure.pdf", dpi=300,
            bbox_inches='tight', backend='pdf')
plt.savefig("publication_figure.png", dpi=300, bbox_inches='tight')
print("Saved: publication_figure.pdf + .png")
plt.show()

# ── Reset rcParams to defaults ─────────────────────────────────────
plt.rcParams.update(plt.rcParamsDefault)
```
</details>

---

### X6 — Interactive Plot with Widgets

**Question:**
Create an interactive sine wave explorer where the user can adjust frequency, amplitude, and phase using `matplotlib.widgets` sliders. Add a Reset button.

<details>
<summary>💡 View Solution</summary>

```python
import matplotlib.pyplot as plt
import matplotlib.widgets as widgets
import numpy as np

# Initial parameters
init_freq  = 1.0
init_amp   = 1.0
init_phase = 0.0

x = np.linspace(0, 4 * np.pi, 500)

fig, ax = plt.subplots(figsize=(10, 6))
plt.subplots_adjust(bottom=0.35)

y_init = init_amp * np.sin(2 * np.pi * init_freq * x + init_phase)
line,  = ax.plot(x, y_init, color='#e74c3c', linewidth=2)
ax.set_ylim(-3, 3)
ax.set_xlim(0, 4 * np.pi)
ax.set_title("Interactive Sine Wave Explorer", fontsize=13)
ax.set_xlabel("x")
ax.set_ylabel("y = A·sin(2πfx + φ)")
ax.axhline(0, color='gray', linewidth=0.8)
ax.grid(True, alpha=0.3)

param_text = ax.text(0.02, 0.92, '', transform=ax.transAxes,
                     fontsize=10, color='#2c3e50')

# Slider axes
ax_freq  = plt.axes([0.2, 0.22, 0.65, 0.03])
ax_amp   = plt.axes([0.2, 0.15, 0.65, 0.03])
ax_phase = plt.axes([0.2, 0.08, 0.65, 0.03])
ax_reset = plt.axes([0.82, 0.01, 0.1, 0.05])

s_freq  = widgets.Slider(ax_freq,  'Frequency',  0.1, 5.0, valinit=init_freq,  color='#3498db')
s_amp   = widgets.Slider(ax_amp,   'Amplitude',  0.1, 3.0, valinit=init_amp,   color='#e74c3c')
s_phase = widgets.Slider(ax_phase, 'Phase (rad)', 0,  2*np.pi, valinit=init_phase, color='#2ecc71')
b_reset = widgets.Button(ax_reset, 'Reset',   color='#ecf0f1', hovercolor='#bdc3c7')

def update(val):
    freq  = s_freq.val
    amp   = s_amp.val
    phase = s_phase.val
    line.set_ydata(amp * np.sin(2 * np.pi * freq * x + phase))
    param_text.set_text(f'A={amp:.2f},  f={freq:.2f} Hz,  φ={phase:.2f} rad')
    fig.canvas.draw_idle()

def reset(event):
    s_freq.reset(); s_amp.reset(); s_phase.reset()

s_freq.on_changed(update)
s_amp.on_changed(update)
s_phase.on_changed(update)
b_reset.on_clicked(reset)

update(None)
plt.show()
```
</details>

---

## 📝 Quick Reference Cheat Sheet

### Setup
```python
import matplotlib.pyplot as plt
import numpy as np

fig, ax = plt.subplots(figsize=(8, 5))   # OOP style (recommended)
plt.show()                                # display
fig.savefig("out.png", dpi=300, bbox_inches='tight')
```

### Common Plot Types
```python
ax.plot(x, y)                  # line plot
ax.scatter(x, y, c=z, s=size) # scatter (colour + size encoding)
ax.bar(x, height)              # vertical bar
ax.barh(y, width)              # horizontal bar
ax.hist(data, bins=30)         # histogram
ax.boxplot(data)               # box plot
ax.violinplot(data)            # violin plot
ax.pie(values, labels=labels)  # pie / donut
ax.imshow(array, cmap='viridis') # image / heatmap
ax.contourf(X, Y, Z, levels=20)  # filled contour
ax.errorbar(x, y, yerr=err)      # error bars
ax.fill_between(x, y1, y2)       # shaded area
ax.step(x, y, where='post')      # step plot
ax.stem(x, y)                    # stem plot
```

### Labels & Annotations
```python
ax.set_title("Title", fontsize=13)
ax.set_xlabel("x label")
ax.set_ylabel("y label")
ax.legend(loc='upper right')
ax.annotate("text", xy=(x,y), xytext=(xt,yt),
            arrowprops=dict(arrowstyle='->'))
ax.text(x, y, "text", ha='center', bbox=dict(boxstyle='round'))
ax.axhline(y=0, color='gray')
ax.axvspan(x1, x2, alpha=0.2, color='yellow')
```

### Axis Control
```python
ax.set_xlim(0, 10)
ax.set_ylim(-1, 1)
ax.set_xticks([0, np.pi, 2*np.pi])
ax.set_xticklabels(['0', 'π', '2π'])
ax.tick_params(axis='x', labelsize=10, rotation=45)
ax.set_xscale('log')           # log scale
ax2 = ax.twinx()               # second y-axis
```

### Subplots
```python
fig, axes = plt.subplots(2, 3, figsize=(12, 7))
fig, axes = plt.subplots(2, 3, sharex=True, sharey=True)

import matplotlib.gridspec as gridspec
gs  = gridspec.GridSpec(2, 3, width_ratios=[2,1,1])
ax  = fig.add_subplot(gs[0, :])   # span all columns
```

### Styles & rcParams
```python
plt.style.use('seaborn-v0_8')
with plt.style.context('ggplot'):
    plt.plot(x, y)             # style scoped to block

plt.rcParams.update({
    'font.size': 12,
    'axes.spines.top': False,
    'figure.dpi': 150,
})
plt.rcParams.update(plt.rcParamsDefault)  # reset
```

### 3D Plots
```python
from mpl_toolkits.mplot3d import Axes3D
ax = fig.add_subplot(111, projection='3d')
ax.plot_surface(X, Y, Z, cmap='viridis')
ax.scatter(x, y, z, c=z, cmap='plasma')
ax.plot_wireframe(X, Y, Z, rstride=2, cstride=2)
ax.view_init(elev=30, azim=45)
```

### Animation
```python
from matplotlib.animation import FuncAnimation
ani = FuncAnimation(fig, update_func, frames=100,
                    interval=50, blit=True)
ani.save('out.gif', writer='pillow', fps=20)
```

---

## 🔧 Setup

```bash
pip install matplotlib numpy scipy pillow
```

```python
import matplotlib
import matplotlib.pyplot as plt
print("Matplotlib version:", matplotlib.__version__)

# Check available backends
print(matplotlib.get_backend())

# For Jupyter notebooks — inline display
# %matplotlib inline
# %matplotlib widget   # interactive widgets in JupyterLab
```

---

*Happy plotting! 📊*
