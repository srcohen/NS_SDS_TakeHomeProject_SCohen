# NS_SDS_TakeHomeProject_SCohen
A take-home assignment for Natural State's Senior Data Scientist role covering BirdNET thresholding and vegetation data QA/QC including two deliverables, reproducibility, requirements for systems integration, and documentation on decision-making and limitations. 

## Status

| Deliverable | Status |
|---|---|
| Bird challenge (BirdNET thresholds) | Done. Analysis and labeled output in `Data_Birds/bird_analysis.md`, requirements for the Tech team in `Data_Birds/Tech_Requirements.md`. |
| Vegetation challenge (data quality report) | Done. Report in `Data_Vegetation/vegetation_analysis.md`, checks in `Data_Vegetation/vegetation_analysis.ipynb`, requirements for the Tech team in `Data_Vegetation/Tech_Requirements.md`. |

## Repository layout

```
Data_Birds/
  bird_analysis.Rmd     Source for the bird analysis (R Markdown)
  bird_analysis.md      Knitted output that GitHub displays
  bird_analysis_files/  Figures produced when knitting
  BirdNET_dev.R         Scratch script used for early exploration
  Tech_Requirements.md  Requirements note for the Tech team
  dev/                  Longer earlier draft of the requirements note

Data_Vegetation/
  vegetation_analysis.ipynb  The data quality checks and the numbers and figures for the report (Python)
  vegetation_analysis.md     The data quality report that GitHub displays
  Tech_Requirements.md       Requirements note for the Tech team
  figures/                   Map and example plot, saved by the notebook
```

## Where the data goes

The assignment data is **not in this repository**. I am waiting to hear whether Natural State is happy for it to be published, so it is kept outside the repo and excluded by `.gitignore` (`*.csv`). All code reads the data from a local folder, and the knitted outputs contain summaries only, not rows of data.

Place the files you were given in a folder with the same layout as the original assignment download:

```
<data folder>/
  Data Birds/
    birdnet_predictions.csv
    validation_results.csv
  Data Vegetation/
    ODK Data Exports/   (the four survey and registration exports)
    Entity lists/       (vegplots, centroids, species, and the other entity lists)
```

The vegetation report is the exception to "summaries only": it embeds a map of the planned plot locations (`Data_Vegetation/figures/transect_map.png`).

For the bird analysis, set the data location in the first code chunk of `Data_Birds/bird_analysis.Rmd`:

```r
data_dir <- "/path/to/your/data folder"
```

On my machine this is `/Users/scohen/Documents/NaturalState/Take Home Project`. It is the only line that needs to change.

For the vegetation notebook, set `data_dir` in the second code cell of `Data_Vegetation/vegetation_analysis.ipynb` to the `Data Vegetation` folder inside your data folder.

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

### Vegetation notebook (Python)

Python 3.13 was used. The packages and versions are in `requirements.txt`.

1. Create an environment and install the packages once:
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   pip install -r requirements.txt
   ```
2. Put the data in place and set `data_dir` as described above.
3. Open `Data_Vegetation/vegetation_analysis.ipynb`, select the `.venv` kernel, and run all cells.

This reruns all 23 checks and rewrites the figures in `Data_Vegetation/figures/`. The last cell updates the "Numbers last computed on" date in `vegetation_analysis.md`. The numbers in the report text are written by hand from the notebook's output, so recheck them if the data changes. The map cell downloads satellite map tiles, so it needs an internet connection.
