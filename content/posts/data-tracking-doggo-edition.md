+++
title = 'Data Tracking Doggo Edition'
date = 2024-04-12T08:27:09-05:00
draft = false
+++
## Introduction

Tracking and visualizing relevant data can support improving our spending habits, nutrition, or strength at the gym. This has spawned a billion-dollar industry for wearables and related apps, empowering users to generate insights and develop goals from the data. We can take this approach to data tracking to explore and solve other aspects of our lives, like tracking factors that affect sleep quality, effectiveness and side effects of medications, or stress triggers. Here, I will outline how I gained insights into my dog’s :dog: medical condition and overall health by:

- :ledger: Logging the data,
- :chart_with_downwards_trend: visualizing the data and
- :envelope_with_arrow: sharing it with the veterinarians.

Let me first introduce Edison :dog:.
Eddie is a 2-year-old Welsh Springer Spaniel (WSS). The WSS differs from the English springer in size, appearance, and colouring. They are shorter, have smaller-sized and higher-set ears, a tapered head, and a distinctive red-and-white coat.

[![Edison as a puppy](/images/eddie-pickup.jpg)](/images/eddie-pickup.jpg) [![Edison now](/images/eddie-2024.jpg)](/images/eddie-2024.jpg)

## Edison's Medical Condition

Feel free to skip to the next section if you do not want to know about the background of my dog's :dog: health.

I got Edison when he was ten weeks old. Since then, I have kept a log book on feedings, poos :poop:, pees, and other observations, including energy level and behaviour. For various reasons, it is an analog logbook on paper, and you must use a pen :nerd_face: :joy:. Around six months of age, I noticed that his poos fluctuated between bad and good.
To better track and source the issue, applying a quantitative measure of stool quality was the first step.
*Luckily, there is already a fecal scoring chart used by veterinarians - the Bristol Stool Form Scale (BSFS), or Bristol stool scale. Thanks to Kenneth Willoughby Heaton †. I will let you research that yourself if you so choose* :wink:.
After eliminating different treats, I realized that none made a statistically significant difference in his average fecal scores. 
It turned out that the poor pupper has [inflammatory bowel disease](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC2835779/) (IBD).
Pharmacological treatment options are essentially immunosuppressants, sometimes combined with other medications to manage the sometimes severe side effects of the immunosuppressant.
Since the diagnosis, Eddie has been on a prescription hydrolyzed protein diet that resulted in some improvements; however, it got to a point to trial immunosuppressants in addition to the diet changes.
I wanted to know how it would affect one of the most accessible clinical signs to read, i.e., his fecal scores. Of course, I also logged other health indicators, including days in between vomiting, lethargy, and reactivity, to name a few. Some visualization was required to compare his fecal scores before administering an immunosuppressant (Atopica) and being on the medication. Would the medication effectively improve his stool?


## Data Science Light

My process was mostly about capturing data, preparing it, and visualizing it, which helped to inform conversations with his veterinarian. Due to my experience, I chose [Python](https://www.python.org/) with [pandas](https://pandas.pydata.org/), [numpy](https://numpy.org/), [matplotlib](https://matplotlib.org/), [pyplot](https://matplotlib.org/3.5.3/api/_as_gen/matplotlib.pyplot.html), and [seaborn](https://seaborn.pydata.org/). Let us call it data science light :wink:.
I decided on a Google spreadsheet to enter the data and add notes or comments as desired; however, I wanted to visualize the data programmatically.
Reading the data from the spreadsheet, manipulating it, and visualizing it is done inside a [Jupyter Notebook](https://jupyter.org/). A snapshot of the data is in this [GitHub repository](https://github.com/manderson-it/data-tracking-doggo-edition).

The prerequisites for accessing the spreadsheet from the notebook are covered in the article [Integrate Google Sheets with Jupyter Notebook](https://medium.com/@techno021/integrate-google-sheets-with-jupyter-notebook-e25a4c349828).
In the Jupyter Notebook, I start by reading, cleaning, and formatting the data from the Google spreadsheet.
Lines 34 to 45 in the code block below, show how I handled the initial data loading.

```python {linenos=table,hl_lines=["34-45"]}
# Imports
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
from bokeh.plotting import figure, output_file, save, show
from bokeh.models.formatters import DatetimeTickFormatter
from bokeh.models import ColumnDataSource, BoxZoomTool, PanTool, ResetTool, HoverTool, WheelZoomTool, Title
import gspread
from df2gspread import df2gspread as d2g
from oauth2client.service_account import ServiceAccountCredentials

# oauth scopes required
scope = [
   'https://spreadsheets.google.com/feeds',
   'https://www.googleapis.com/auth/drive'
]
#Name of our Service Account Key file
google_key_file = '<TopSecret>'
credentials = ServiceAccountCredentials.from_json_keyfile_name(google_key_file, scope)
gc = gspread.authorize(credentials)

# worksheet ID from Google Sheet URL
spreadsheet_key = '<YourSpreadSheetKey>'
# sheet name
wks_name = 'fcs'
# Read from worksheet (name: fcs) in Google Sheet file by using Worksheet ID/name
workbook = gc.open_by_key(spreadsheet_key)
# Selecting which sheet to pulling the data
sheet = workbook.worksheet(wks_name)
# Pulling the data and transform it to the data frame
values = sheet.get_all_values()

df = (pd.DataFrame(values[1:], columns = values[0])
      .replace(r'^\s*$', np.nan, regex=True)
      .fillna(0) # keep NaN, otherwise zeros will influence the mean()
      .apply(pd.to_numeric, errors='ignore') # Convert number strings to floats and ints, Score6 and Score7 are int64 not float64 which causes problems when aplying other functions
      .astype({
         "Score6":"float", # pd.to_numeric converts to int unfortunately, needs to be consistent
         "Score7":"float", # pd.to_numeric converts to int unfortunately, needs to be consistent
     }) # close astype()
     ) # end df

df['Date'] = pd.to_datetime(df['Date'])
df.sort_values(by=['Date'], inplace=True, ascending=True)

print(df.dtypes)
```

Too many data points can make a graphic unreadable.
I decided on an easy way to examine a specific date range, as outlined in the next code block.

```python {linenos=table}
mask = (df['Date'] > '2023-03-14') & (df['Date'] <= '2023-05-29')
df = df.loc[mask]

# create temporary dataframe with NaN. NaN will be skipped by mean()
sub_df = df.replace(0, np.nan)
df['Mean'] = sub_df[["Score1","Score2","Score3","Score4","Score5","Score6","Score7"]].mean(axis=1, skipna=True).round(2)
df['color'] = df.NoteMedical.map(
    lambda x: 'steelblue' if x == 'Atopica' else (
    'forestgreen' if x == 'AfterAtopicaEnd' else 'lightcoral'))
```

After many months of diet changes only, Eddie's health reached a point where it was prudent to begin a trial of immunosuppressants.
Visualizing the data revealed that while one clinical sign improved, another declined steeply. It was one of the most common side effects of the medication. Supposedly, it would improve over time again by tapering the drug to the minimum effective dose.
The improvement is shown in the diagram below.
The chart shows the mean fecal scores per day in lightcoral before beginning the immunosuppressant trial.
Days on the immunosuppressant Atopica are shown in steelblue.
Followed by days with Prednisone in green.
While his stool improved, the frequency of vomiting increased to daily during Atopica administration.

[![Chart](/images/Edison-FecalScoreChart-2023.png)](/images/Edison-FecalScoreChart-2023.png)

Once Edison stopped taking Atopica, the frequency of his vomiting reduced drastically again. Later, Edison was on the immunosuppressant Prednisone, which also had terrible side effects that did not warrant continuing the drugs at the time. For now, Edison manages on his hydrolyzed protein diet, and he remains a happy and mostly healthy dog.

## Summary

Data tracking and visualization help to determine areas of improvement or correlate different data. They are accessible ways to keep track of, for example, your sleep, nutrition, weights at the gym, and daily lows and highs.
In Edison's case, data tracking and visualization help determine his health status, and it is easy to share this information with any veterinarian or specialist instead of showing a lengthy table. A deeper analysis of the existing data is unlikely to reveal more insights, as the predominant clinical data cannot be retrieved on a regular basis, which would require full anesthesia. It is easy enough to evaluate the furry overlord's overall well-being and health trends by mindfully observing him.

