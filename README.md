## Objective
Migrate as many of the bokeh plots to plotly in https://github.com/ropoctl/dataprep as possible.

## Starting steps
1. Fork https://github.com/ropoctl/dataprep on github and checkout locally.
2. Install in a jupyter notebook environment. Note: dataprep has dependencies that make it work best in python 3.9. You will need to play with it interactively. Add plotly as a requirement to your local copy before installing. In the interim, plotly and bokeh will both be used.
3. Test out this titanic workflow for the baseline. Visuals should appear in your notebook. https://docs.dataprep.ai/user_guide/eda/titanic.html

## Starting steps - Solution

*I employed two approaches to clone the repository and work with it:* 
1) *Locally on my computer.* 
2) *Through Google Colab, as I encountered resource limitations that hindered error resolution in subsequent steps.*

*Next, I used the `poetry shell` command to create a virtual environment, where I added Plotly, and then initiated Jupyter Notebook. 
Should mention that `poetry install` encountered a dependency error with codecov 2.1.12, so I used `poetry add codecov` that included version 2.1.13 instead. 
After verifying the code of the script provided in the [Dataprep EDA Titanic User Guide](https://docs.dataprep.ai/user_guide/eda/titanic.html), I proceeded further.*


## Actual steps
1. Create a decorator function in python that will log calls to a function. The goal is to record the parameters sent to hist_viz and other plotting functions in dataprep. Put this function in some utils module and import it wherever plotting functions you are migrated are, and then decorate them.
2. Run one of the dataprep report generation functions in your jupyter notebook. Check that you have plotting calls logged.
3. Start with dataprep/eda/diff/render.py - There are about 5-6 calls to bokeh plotting in this module, migrate it first. Take one of the plotting functions like `bar_viz`, and make a version that uses plotly in your notebook. I recommend using https://phind.com https://www.bing.com/chat or chatgpt. These are very mechanical types of translations that the AI does well. Your main job will be to QA them side by side.
4. For each function you rewrite, you can call it with the plotly function- (if the plotly figure is that last cell of the notebook, as an expression, it will render). Bokeh plots need these functions `from bokeh.io import output_notebook, show`

## Actual Steps - Solution

*I developed a decorator to collect logs from function calls, displaying the arguments a function receives. Here is its code:*

```python
def log_function_call(func):
    def wrapper(*args, **kwargs):
        print(f"Called {func.__name__} with args: {args}, kwargs: {kwargs}")
        return func(*args, **kwargs)
    return wrapper
```
*After setting the groundwork, I proceeded by annotating `@log_function_call` in the `render.py` file before the declaration of the `hist_viz`, `bar_viz`, and several other functions with the `_viz_` suffix.*

*As per the instructions in step 2, the next task was to initiate the report generation function to verify the decorator's functionality.*

*When I attempted to use `create_report()`, it failed, presenting an error. Resolving this issue took some time. Ultimately, as we discussed, the root cause was identified within the `formatter.py` file lines: `"plots_tab": list(zip(rndrd["meta"][1:], insight_keys))` couldn't provide the three expected values for the loop.*

*Following your advice to utilize the `main` function, which in turn uses `format_report()`, I encountered another issue. Both `main` and `format_report()` only return a stream of formatted data in dictionary form. However, to verify the decorator's and the plotting functions' operations, I needed the actual plots to be explicitly generated.*

*I resolved the decorator functionality verification issue as follows: I took the `hist_viz` function and directly created a set of synthetic data upon which this function built a histogram, allowing me to confirm that the decorator was operational and passing information about the function's arguments.*

*Subsequently, I migrated the `hist_viz` function, which utilizes the Bokeh library, to `hist_viz_plotly`, which employs the Plotly library, and I tested this function's performance on the same dataset.*

### hist_viz_plotly

```python
def hist_viz_plotly(
    hist: List[Tuple[np.ndarray, np.ndarray]],
    nrows: List[int],
    col: str,
    yscale: str,
    plot_width: int,
    plot_height: int,
    show_yticks: bool,
    df_labels: List[str],
    orig: Optional[List[str]] = None,
):
    fig = go.Figure()

    for i, (counts, bins) in enumerate(hist):
        if sum(counts) == 0:
            continue
        # Intervals for hints
        intvls = [f"{left} - {right}" for left, right in zip(bins[:-1], bins[1:])]
        df = pd.DataFrame({
            "intvl": intvls,
            "left": bins[:-1],
            "right": bins[1:],
            "freq": counts,
            "pct": counts / nrows[i] * 100,
            "orig": orig[i] if orig else None,
        })

        # Tracing histogram
        for _, row in df.iterrows():
            fig.add_trace(go.Bar(
                x=[(row['left'] + row['right']) / 2],
                y=[row['freq']],
                width=row['right'] - row['left'],
                name=row['orig'],
                marker=dict(color=CATEGORY10[i]),
                hoverinfo='text',
                hovertext=f"Bin: {row['intvl']}<br>Frequency: {row['freq']}<br>Percent: {row['pct']:.2f}%<br>Source: {row['orig']}",
            ))

    # Title and axis
    fig.update_layout(
        title=col,
        xaxis_title=col,
        yaxis_title="Frequency" if show_yticks else None,
        yaxis_type=yscale,
        width=plot_width,
        height=plot_height,
        showlegend=True if orig else False,
    )

    return fig
    
``` 

## Test on the dataset as follows
```python

import numpy as np
import plotly.graph_objs as go

from dataprep.eda.diff.render import hist_viz_plotly

# Data to visualize
np.random.seed(42)
data = np.random.normal(loc=0, scale=1, size=1000)
hist_data, edges = np.histogram(data, bins=20)
hist = [(hist_data, edges)]
nrows = [1000]
col = "Demo Data"
yscale = "linear"
plot_width = 800
plot_height = 600
show_yticks = True
df_labels = ["Demo Dataset"]
orig = ["Dataset 1"]

# Call Plotly
fig = hist_viz_plotly(
    hist=hist,
    nrows=nrows,
    col=col,
    yscale=yscale,
    plot_width=plot_width,
    plot_height=plot_height,
    show_yticks=show_yticks,
    df_labels=df_labels,
    orig=orig
)

fig.show()

```
![output_hist_viz_plotly](https://github.com/kerrovitarr/report/raw/3e41ccb1a4049edf4b6935674c40922a774922ee/photo_2024-02-18_20-41-53.jpg)

![output_hist_viz](https://github.com/kerrovitarr/report/raw/3e41ccb1a4049edf4b6935674c40922a774922ee/photo_2024-02-18_20-41-35.jpg)


## Conclusion

*To summarize, the task is migrating functions that generate plots from the Bokeh library to Plotly. Concurrently, I'm faced with issues related to report generation (`create_report` isn't working, and `main`'s output lacks actual plots) and plot visualization. I managed to adapt the `hist_viz` function using synthetic data I provided, but replicating this process with other functions is not straightforward. Therefore, a more in-depth and detailed examination of your product is necessary to address this challenge.*

*Given these considerations, I would like to discuss this task, my decisions, and the final outcome of the hiring process with you.*
