# NS_SDS_TakeHomeProject_SCohen
A take-home assignment for Natural State's Senior Data Scientist role covering BirdNET thresholding and vegetation data QA/QC including two deliverables, reproducibility, requirements for systems integration, and documentation on decision-making and limitations. 

## Status

| Deliverable | Status |
|---|---|
| Bird challenge (BirdNET thresholds) | In progress. Data exploration and QC are done. Modelling: the reference fit for one species (Nightjar) is done; the other three species are next. |
| Vegetation challenge (data quality report) | Not started. |

## Repository layout

```
Data_Birds/
  bird_analysis.Rmd     Source for the bird analysis (R Markdown)
  bird_analysis.md      Knitted output that GitHub displays
  bird_analysis_files/  Figures produced when knitting
  BirdNET_dev.R         Scratch script used for early exploration
```

## Where the data goes

The assignment data is **not in this repository**. I am waiting to hear whether Natural State is happy for it to be published, so it is kept outside the repo and excluded by `.gitignore` (`*.csv`). All code reads the data from a local folder, and the knitted outputs contain summaries only, not rows of data.

Place the files you were given in a folder with the same layout as the original assignment download:

```
<data folder>/
  Data Birds/
    birdnet_predictions.csv
    validation_results.csv
```

Then set the data location in the first code chunk of `Data_Birds/bird_analysis.Rmd`:

```r
data_dir <- "/path/to/your/data folder"
```

On my machine this is `/Users/scohen/Documents/NaturalState/Take Home Project`. It is the only line that needs to change.

## How to reproduce

The bird analysis is written in R, to match how Natural State's Biometrics team works.

**Requirements**
- R 4.6.1 (earlier 4.x versions are likely to work but are untested)
- The R packages `tidyverse`, `rmarkdown` and `knitr` (the modelling section also uses `mgcv`, which is installed with R by default)
- Pandoc, which is bundled with RStudio

**Steps**
1. Install the packages once:
   ```r
   install.packages(c("tidyverse", "rmarkdown", "knitr", "logistf"))
   ```
2. Put the data in place and set `data_dir` as described above.
3. Open `Data_Birds/bird_analysis.Rmd` in RStudio and click **Knit**, or run:
   ```r
   rmarkdown::render("Data_Birds/bird_analysis.Rmd")
   ```

This regenerates `bird_analysis.md` and the figures from the raw files. The QC check tables, summary tables and figures are computed when the document is knitted. The field-by-field schema tables and some figures quoted in the surrounding text are written by hand from what the code showed, so recheck them if the data changes.

`logistf` provides the Firth regression used for the species with almost no incorrect validation clips. Further packages will be added here if later sections need them.
