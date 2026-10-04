NS Take-Home Assignment - Bird Challenge
================

## Contents

1.  [Data exploration and summaries](#1-data-exploration-and-summaries)
    1.  [Data schema:
        `birdnet_predictions.csv`](#11-data-schema-birdnet_predictionscsv)
    2.  [Data schema:
        `validation_results.csv`](#12-data-schema-validation_resultscsv)
    3.  [Cross-file checks](#13-cross-file-checks)
    4.  [Distributions](#14-distributions)

## 1. Data exploration and summaries

### 1.1 Data schema: `birdnet_predictions.csv`

29,491 rows, 13 fields, no missing values. Each row is one BirdNET
*prediction*: one species predicted in one 3-second segment of one audio
file, with a confidence score of at least 0.1. The field layout is the
Raven selection-table format that BirdNET writes out. Descriptions below
are my reading of the data and the take-home assignment description;
items marked *(inferred)* are not stated in the description.

| Field | Type | Description | Observed in this dataset |
|----|----|----|----|
| `selection` | numeric (integer) | Row counter within a Raven selection table. Restarts for each audio file, so it is **not** a unique ID. *(inferred)* | 1 to 270 |
| `view` | character | Raven display view the selection was made in. Constant. | Always `Spectrogram 1` |
| `channel` | numeric (integer) | Audio channel analysed. Constant. | Always `1` |
| `begin_time_s` | numeric (integer) | Start of the 3-second segment, in seconds from the start of the audio file. | 0 to 3,599 |
| `end_time_s` | numeric (integer) | End of the segment, in seconds from the start of the file. Always `begin_time_s + 3`. | 3 to 3,602 |
| `low_freq_hz` | numeric | Lower frequency bound of the analysed band (Hz). Constant. | Always `0` |
| `high_freq_hz` | numeric | Upper frequency bound of the analysed band (Hz). Constant. | Always `15000` |
| `common_name` | character | Predicted species (the class). | 4 species: Abyssinian Nightjar, African Black-headed Oriole, Red-billed Firefinch, Three-banded Plover |
| `species_code` | character | Short code for the predicted species. One-to-one with `common_name`. | 4 codes: species_code: abynig1, abhori1, rebfir2, thbplo1 |
| `confidence` | numeric | BirdNET confidence score for this species in this segment. **Not a probability**; its meaning differs by species (Wood & Kahl 2024). Predictions below 0.1 were not kept. | 0.1 to 1.0 |
| `begin_path` | character | Relative path to the source audio file: `Grid/Recorder/Recorder_YYYYMMDD_HHMMSS.WAV`. Encodes the recorder, date and start hour of a 1-hour recording. | 1,696 distinct files, all in `Grid2` |
| `file_offset_s` | numeric (integer) | Offset of the segment within the file, in seconds. Identical to `begin_time_s` in every row, so redundant. | 0 to 3,599 |
| `folder` | character | Recorder (device) ID. Matches the recorder folder inside `begin_path`. | 43 recorders (`RBS02`, …) |

**Key and constants.** No single field identifies a prediction. The
combination `begin_path` + `begin_time_s` + `common_name` is unique for
all 29,491 rows, and that is the key I will use when labelling. `view`,
`channel`, `low_freq_hz` and `high_freq_hz` never vary and carry no
information for the analysis.

``` r
# Reproduces the claims above directly from the data
predictions %>%
  summarise(
    rows = n(),
    missing_cells = sum(is.na(across(everything()))),
    unique_key_rows = n_distinct(begin_path, begin_time_s, common_name),
    offset_equals_begin = all(file_offset_s == begin_time_s),
    segment_is_3s = all(end_time_s - begin_time_s == 3),
    recorder_matches_path = all(str_split_i(begin_path, "/", 2) == folder)
  ) %>%
  pivot_longer(everything())
```

    ## # A tibble: 6 × 2
    ##   name                  value
    ##   <chr>                 <int>
    ## 1 rows                  29491
    ## 2 missing_cells             0
    ## 3 unique_key_rows       29491
    ## 4 offset_equals_begin       1
    ## 5 segment_is_3s             1
    ## 6 recorder_matches_path     1

Predictions per species:

``` r
predictions %>% count(common_name, species_code, sort = TRUE)
```

    ## # A tibble: 4 × 3
    ##   common_name                 species_code     n
    ##   <chr>                       <chr>        <int>
    ## 1 African Black-headed Oriole abhori1      18054
    ## 2 Abyssinian Nightjar         abynig1      10187
    ## 3 Three-banded Plover         thbplo1       1015
    ## 4 Red-billed Firefinch        rebfir2        235

#### QC checks: predictions

Each check is computed from the data when this document is knitted.
**PASS** means nothing to act on. **FLAG** means a problem I have to
handle in the analysis. **INFO** means an oddity to be aware of and
record, not necessarily a problem.

| Check | Result | Status |
|:---|:---|:--:|
| No missing values | 0 missing cells | PASS |
| `common_name` and `species_code` map one-to-one | 4 names, 4 codes, 4 distinct pairs | PASS |
| No leading/trailing whitespace in text fields | 0 values affected | PASS |
| `confidence` within 0.1 to 1 | range 0.1 to 1 | PASS |
| No `confidence` of exactly 1.0 (its logit is infinite) | 4 predictions at 1.0; clip before the logit transform | FLAG |
| Key (`begin_path`, `begin_time_s`, `common_name`) is unique | 0 duplicate keys | PASS |
| Every segment is 3 s long | 0 rows differ | PASS |
| No overlapping segments for the same species in a file | 0 overlaps | PASS |
| `begin_path` follows `Grid/Recorder/Recorder_YYYYMMDD_HHMMSS.WAV` | 0 paths differ | PASS |
| Recorder in `folder` matches recorder in `begin_path` | all rows match | PASS |
| Segments end within a 3,600 s recording | 14 segments end after 3,600 s (latest end 3602 s) | INFO |
| One species predicted per segment | 3 segments have more than one species | INFO |

**Reading the flags.** The four `confidence` values of exactly 1.0
cannot go through the logit transform (`ln(c / (1 - c))` is infinite),
so they need to be clipped to just below 1 OR removed from the data;
either way, this will be recoded as a decision with context for the tech
team to be aware of. The INFO rows are small oddities: a few segments
run up to 2 seconds past the one-hour mark, and 3 segments have
predictions for two species, which is another reason the key includes
the species.

### 1.2 Data schema: `validation_results.csv`

551 rows, 6 fields, no missing values. Each row is one BirdNET
prediction that a human ornithologist listened to and marked correct or
incorrect. This is the only file with ground truth, so it is what the
threshold model is fitted on. Items marked *(inferred)* are my reading
and are not stated in the take-home assignment description.

| Field | Type | Description | Observed in this dataset |
|----|----|----|----|
| `scientificName` | character | Scientific (Latin) name of the predicted species. One-to-one with `commonName`. | 4 species: *Caprimulgus poliocephalus*, *Charadrius tricollaris*, *Lagonosticta senegala*, *Oriolus larvatus* |
| `commonName` | character | Predicted species (the class). Same labels as `common_name` in the predictions file. | 4 species: Abyssinian Nightjar, Three-banded Plover, Red-billed Firefinch, African Black-headed Oriole |
| `vBirdNET` | character | BirdNET model version that produced the prediction. Constant. | Always `v2.4` |
| `filename` | character | Name of the validated audio clip: `<confidence>_<n>_<recorder>_<YYYYMMDD>_<HHMMSS>.wav`. The leading number is the confidence score, so clips can be matched to scores (Wood & Kahl 2024). `<n>` is an integer of unknown meaning, possibly a clip index *(inferred)*. Unique per row. | 551 distinct filenames; 32 recorders; dates 2023-06-20 to 2023-07-12 |
| `confidence` | numeric | BirdNET confidence score for the validated prediction. Identical to the leading number in `filename`. Rounded to 3 decimals, whereas the predictions file keeps 4. | 0.101 to 1.0 |
| `outcome` | numeric (0/1) | Ornithologist’s verdict: `1` = prediction correct (true positive), `0` = incorrect (false positive). This is the response variable. | 422 correct (1), 129 incorrect (0) |

**Notes.**

- The two files have no shared ID, and many validation clips cannot be
  traced back to a row in the predictions file (some come from
  recordings that start off the hour, unlike every file in the
  predictions). This does not affect fitting, which needs only
  `confidence` and `outcome`, for now flag but follow up with question
  before deciding. Details are in [1.3](#13-cross-file-checks).
- Correctness varies hugely by species (see below), which is what makes
  a single global threshold inappropriate.

``` r
# Reproduces the claims above directly from the data
validation %>%
  summarise(
    rows = n(),
    missing_cells = sum(is.na(across(everything()))),
    unique_filenames = n_distinct(filename),
    one_model_version = n_distinct(vBirdNET) == 1,
    outcome_is_0_or_1 = all(outcome %in% c(0, 1)),
    confidence_matches_filename = all(abs(as.numeric(str_extract(filename, "^[0-9.]+")) - confidence) < 1e-9),
    recorders = n_distinct(str_match(filename, "^[0-9.]+_[0-9]+_([A-Za-z0-9]+)_")[, 2])
  ) %>%
  pivot_longer(everything())
```

    ## # A tibble: 7 × 2
    ##   name                        value
    ##   <chr>                       <int>
    ## 1 rows                          551
    ## 2 missing_cells                   0
    ## 3 unique_filenames              551
    ## 4 one_model_version               1
    ## 5 outcome_is_0_or_1               1
    ## 6 confidence_matches_filename     1
    ## 7 recorders                      32

Validated clips and correct rate per species:

``` r
validation %>%
  group_by(commonName, scientificName) %>%
  summarise(n = n(),
            n_correct = sum(outcome),
            pct_correct = round(100 * mean(outcome), 1),
            min_conf = min(confidence),
            max_conf = max(confidence),
            .groups = "drop") %>%
  arrange(desc(pct_correct)) %>%
  knitr::kable(
    col.names = c("Species", "Scientific name", "Clips validated", "Correct",
                  "% correct", "Lowest score", "Highest score"),
    align = "llrrrrr"
  )
```

| Species | Scientific name | Clips validated | Correct | % correct | Lowest score | Highest score |
|:---|:---|---:|---:|---:|---:|---:|
| African Black-headed Oriole | Oriolus larvatus | 150 | 150 | 100.0 | 0.104 | 0.989 |
| Three-banded Plover | Charadrius tricollaris | 150 | 149 | 99.3 | 0.102 | 0.994 |
| Abyssinian Nightjar | Caprimulgus poliocephalus | 150 | 117 | 78.0 | 0.102 | 1.000 |
| Red-billed Firefinch | Lagonosticta senegala | 101 | 6 | 5.9 | 0.101 | 0.907 |

**What this shows.** Before fitting anything, this is how well BirdNET’s
predictions held up when a human checked them. The overall rate (422 of
551, 77%) hides large differences: every Oriole clip was correct, nearly
every Plover clip was correct, the Nightjar was in between, and the
Firefinch was almost always wrong. The lowest and highest scores show
the range of confidence each species was validated across, which limits
where a fitted curve can be trusted.

#### QC checks: validation

| Check | Result | Status |
|:---|:---|:--:|
| No missing values | 0 missing cells | PASS |
| `scientificName` and `commonName` map one-to-one | 4 common names, 4 scientific names, 4 distinct pairs | PASS |
| No leading/trailing whitespace in text fields | 0 values affected | PASS |
| `outcome` is 0 or 1 | 0 other values | PASS |
| `filename` is unique | 0 duplicates | PASS |
| `filename` follows `confidence_n_Recorder_YYYYMMDD_HHMMSS.wav` | 0 filenames differ | PASS |
| `confidence` equals the score in `filename` | all rows match | PASS |
| Single BirdNET version | v2.4 | PASS |
| No `confidence` of exactly 1.0 (its logit is infinite) | 2 clips at 1.0; clip before the logit transform | FLAG |
| Every species has both correct and incorrect clips | all clips have the same outcome for: African Black-headed Oriole | FLAG |
| No species is nearly one-sided (only 1 correct or 1 incorrect clip) | 1 clip differs for: Three-banded Plover | INFO |

**Reading the flags.** The file is structurally clean. The two
substantive flags are about the modelling. Two clips sit at exactly 1.0
and need the same clipping as the predictions. More importantly, one
species has only correct clips, and another has a single incorrect clip.
A standard logistic regression has no stable solution when the outcome
never varies (“perfect separation”), so those species need an explicit
rule; this is a modelling decision I make in the next section.

### 1.3 Cross-file checks

The two files should describe the same recordings and species. These
checks compare them directly.

| Check | Result | Status |
|:---|:---|:--:|
| Same species in both files | 4 in predictions, 4 in validation | PASS |
| Same date range in both files | predictions 20230620 to 20230712; validation 20230620 to 20230712 | PASS |
| Every validated recorder appears in the predictions | 32 validated recorders | PASS |
| Every recorder in the predictions has validation clips | 11 of 43 recorders have none | FLAG |
| Validation recordings start on the hour, like the predictions | 288 of 551 validation clips come from recordings starting off the hour | FLAG |

**Can the validated clips be traced back to a prediction?** Matching on
recording, species and confidence (to 3 decimals):

| Recording start | No matching prediction | One matching prediction | Several matching predictions |
|:---|---:|---:|---:|
| Starts off the hour | 288 | 0 | 0 |
| Starts on the hour | 10 | 207 | 46 |

All of the clips from off-the-hour recordings have no match, because
every file in the predictions starts on the hour. So a large part of the
validation set comes from recordings that are not in the predictions
file. Most of the on-the-hour clips do match, but a few match several
predictions (the same score occurs more than once in a file once it is
rounded to 3 decimals). This does not stop the threshold fit, which
needs only `confidence` and `outcome`, but it means I cannot say the
validation set is a subset of this predictions file. For now flag and
follow up with question before making a decision.

**Were the validated clips representative of the predictions?** Compare
the score distributions:

| Species | Set | n | Median score | Share at 0.9 or higher |
|:---|:---|---:|---:|---:|
| Abyssinian Nightjar | Predictions | 10187 | 0.32 | 18.0% |
| Abyssinian Nightjar | Validation | 150 | 0.65 | 36.0% |
| African Black-headed Oriole | Predictions | 18054 | 0.28 | 3.3% |
| African Black-headed Oriole | Validation | 150 | 0.57 | 26.0% |
| Red-billed Firefinch | Predictions | 235 | 0.14 | 0.4% |
| Red-billed Firefinch | Validation | 101 | 0.14 | 1.0% |
| Three-banded Plover | Predictions | 1015 | 0.29 | 7.3% |
| Three-banded Plover | Validation | 150 | 0.58 | 28.0% |

For three of the four species, the validated clips have much higher
scores than the predictions as a whole (the Nightjar median is about
twice as high). The validation set was therefore weighted towards high
scores rather than drawn at random. That is sensible for finding a
high-precision threshold, since it puts more clips where false positives
turn into true positives, as Wood & Kahl recommend. The consequence is
that the overall 77% correct rate from the validation file is **not**
the precision of the whole predictions file and should not be quoted as
such. The Firefinch is the exception: its validated and predicted scores
are both low, and almost no clip reaches 0.9.

### 1.4 Distributions

Four views of how the data are spread across scores, correctness,
recorders and time. The colours are fixed: blue for the predictions,
orange for the validation set. Each species gets its own panel, because
BirdNET scores mean different things for different species.

#### Score histograms per species: predictions vs validation set

<img src="bird_analysis_files/figure-gfm/plot-score-distribution-1.png" alt="Four panels, one per species, comparing the distribution of BirdNET confidence scores in the predictions and in the validation set."  />

Bars show the share of each set falling in each score bin, so the two
sets are comparable even though one has about 29,000 clips and the other
about 550. Where the orange bars sit well above the blue bars at high
scores, the validation set over-represents high-scoring clips. That is
the case for the Nightjar, Oriole and Plover. For the Firefinch the two
sets match, and most clips score below 0.2.

#### Share of validated clips that were correct, by score band and species

<img src="bird_analysis_files/figure-gfm/plot-share-correct-1.png" alt="Four panels, one per species, showing the share of validated clips that were correct within bands of BirdNET confidence, with a dashed reference line at 99 percent."  />

This is a model-free preview of what the logistic regression will
estimate: the share of validated clips that were correct, in bands of
score. The dashed line marks the 99% target, and the point size shows
how many clips each point rests on. Bands with only a few clips can jump
around, so read the large points first.

- **Nightjar:** the share correct climbs from about a third at the
  lowest scores to 100% from about 0.4 upwards, which is the shape the
  method expects.
- **Oriole:** every band is 100% correct, including the lowest scores.
  Nothing in the data shows where correctness starts to fail.
- **Plover:** the same, apart from a single incorrect clip in the 0.2 to
  0.3 band.
- **Firefinch:** almost every clip below 0.6 was wrong, and there is one
  clip above that, which was correct. One clip cannot support a
  threshold.

The Oriole and Plover panels are why the standard fit breaks down: even
the lowest-scoring clips were correct, so the data cannot show where
correctness starts to fail, and a threshold may sit at or below the
lowest score BirdNET reports (0.1). How to handle that is the main
modelling decision for the next section.

#### Predictions per recorder, from busiest to quietest

<img src="bird_analysis_files/figure-gfm/plot-recorders-1.png" alt="Horizontal bar chart of the number of predictions for each of the 43 recorders, sorted from most to fewest."  />

The predictions are concentrated in a few recorders. The five busiest
account for 62% of all predictions, and the busiest, RBS58, alone has
5,617. Many recorders contribute only a handful. Predictions from the
same recorder, and from neighbouring segments of a continuous call, are
not independent of each other, so the effective sample size is smaller
than the row count suggests. Thresholds fitted on the validation clips
are applied to all of these recorders, and 11 of them have no validation
clips (see 1.3).

#### Predictions per day and by hour of day (hour of day shown per species)

<img src="bird_analysis_files/figure-gfm/plot-time-1.png" alt="Bar chart of predictions per day over the 23 recording days."  />

<img src="bird_analysis_files/figure-gfm/plot-hour-1.png" alt="Four panels, one per species, showing predictions by hour of day at which the recording started."  />

Predictions are available for every day from 20 June to 12 July 2023 but
the daily counts vary a lot. They reflect how much was recorded as well
as how much the birds called, and I don’t have the recording effort per
day, so I would not read the day-to-day pattern as bird activity. By
hour of day, the patterns differ by species: the Nightjar is
concentrated in the evening and the early morning hours, and the Oriole
peaks in the morning. Each panel has its own vertical scale, since the
species have very different numbers of predictions. Time of day is not
part of the planned fit, so any effect of it on scores would be absorbed
into a single curve per species rather than modelled.
