Filled Pause Perspective-Taking Study (Barr & Seyfeddinipur, 2010)
================

# Description

This repository contains data files, stimulus materials, and scripts
associated with the following psycholinguistic study.

Barr, D. J., & Seyfeddinipur, M. (2010). [The role of fillers in
listener attributions for speaker
disfluency.](https://doi.org/10.1080/01690960903047122) *Language and
Cognitive Processes*, *25*, 441–455.

  - `data_raw/` : raw data files in CSV format
  - `stimuli/` : stimulus files (Windows BMP image files and WAV audio
    files)

# Codebook for files in `data_raw/`

This section provides [R code](https://www.r-project.org) for importing
the raw data files as well as descriptions of the fields within each
resulting table. To use the code, load the
[tidyverse](https://tidyverse.org) package for R.

``` r
library("tidyverse")
```

## Files related to study design

### `areas_of_interest.csv`

Contains the coordinates for the two screen regions (areas of interest,
or AOIs) where pictures were displayed. Each picture was a 400x400
bitmap image, displayed on a computer screen with a 1024x768 resolution.

``` r
aoi <- read_csv(file.path("data_raw", "areas_of_interest.csv"),
                col_types = "ciiii")
```

| field | description                                  |
| :---- | :------------------------------------------- |
| AOI   | name of area of interest (left or right)     |
| x1    | upper left X coordinate of image in pixels   |
| y1    | upper left Y coordinate of image in pixels   |
| x2    | bottom right X coordinate of image in pixels |
| y2    | bottom right Y coordinate of image in pixels |

### `block_images.csv`

Information about what images go with which blocks. Each block had three
images, labelled A, B, C in the manuscript (see Barr & Seyfeddinipur,
2010, pp. 445–446).

``` r
block_images <- read_csv(file.path("data_raw", "block_images.csv"),
                         col_types = "iic")
```

| field   | description                             |
| :------ | :-------------------------------------- |
| block   | unique integer identifying blocks       |
| image   | bitmap image number (e.g., 52 = 52.bmp) |
| imgrole | role the image plays (A, B, C)          |

### `blocks.csv`

There were four versions of the stimulus materials created for
counterbalancing purposes. This file has information about what
condition each block was presented in across the four versions.

``` r
blocks <- read_csv(file.path("data_raw", "blocks.csv"),
                   col_types = "iicccccc")
```

| field     | description                                              |
| :-------- | :------------------------------------------------------- |
| vers      | which version of the stimuli (1-4)                       |
| block     | unique integer identifying blocks                        |
| blocktype | whether the block was critical or noncritical            |
| trainspkr | who the speaker was during training                      |
| testspkr  | who the speaker was at test                              |
| spkr      | what speaker condition the block was in (same, diff)     |
| pause     | what pause condition the block was in (filled, baseline) |
| soundfile | soundfile that played at test                            |

### `versions.csv`

Trial-by-trial information about training and test trials for each block
and each version of the stimulus materials.

``` r
versions <- read_csv(file.path("data_raw", "versions.csv"),
                     col_types = "iiiiiicci")
```

| field       | description                                                       |
| :---------- | :---------------------------------------------------------------- |
| vers        | which version of stimulus materials (1, 2, 3, or 4)               |
| block       | unique integer identifying each block                             |
| blocktrial  | integer identifying presentation order within the block           |
| left        | image number in the left area of interest (e.g., 32 means 32.BMP) |
| right       | image number in the right area of interest                        |
| target      | position of the target (0 = left, 1 = right)                      |
| soundfile   | name of the wave file (e.g., 4 = 4.wav, 6umE = 6umE.wav)          |
| who         | identify of the speaker                                           |
| showspeaker | display speaker identity before trial (0 = no, 1 = yes)           |

## Files related to subject data

### `subjects.csv`

Information about each subject in the experiment.

``` r
subjects <- read_csv(file.path("data_raw", "subjects.csv"),
                     col_types = "ici")
```

| field  | description                                         |
| :----- | :-------------------------------------------------- |
| SubjID | unique integer identifying each subject             |
| Gender | gender of the subject                               |
| vers   | which version of the materials the subject received |

### `trials.csv`

Information about each trial in the experiment.

``` r
trials <- read_csv(file.path("data_raw", "trials.csv"),
                   col_types = "iiiiiii")
```

| field     | description                                        |
| :-------- | :------------------------------------------------- |
| TrialID   | integer uniquely identifying each collected trial  |
| SubjID    | integer uniquely identifying each subject          |
| block     | block number                                       |
| btrial    | trial number within the block                      |
| tord      | trial order                                        |
| RT        | response time in ms (from start of audio playback) |
| Selection | which image was selected (0 = left, 1 = right)     |

### `mouse.csv`

Information about the position of the mouse cursor on each trial on a
display with 1024x768 resolution.

``` r
mouse <- read_csv(file.path("data_raw", "mouse.csv"),
                  col_types = "iiii")
```

| field   | description                                       |
| :------ | :------------------------------------------------ |
| TrialID | integer uniquely identifying each collected trial |
| Msec    | milliseconds from onset of soundfile              |
| X       | horizontal position of mouse cursor               |
| Y       | vertical position of mouse cursor                 |

# Scripts

## Data import and pre-processing

``` r
library("lme4")
library("tidyverse")

## import the data
subjects <- read_csv(file.path("data_raw", "subjects.csv"),
                     col_types = "ici")

blocks <- read_csv(file.path("data_raw", "blocks.csv"),
                   col_types = "iicccccc")

versions <- read_csv(file.path("data_raw", "versions.csv"),
                     col_types = "iiiiiicci")

trials <- read_csv(file.path("data_raw", "trials.csv"),
                   col_types = "iiiiiii")

mouse <- read_csv(file.path("data_raw", "mouse.csv"),
                  col_types = "iiii")

## audiotimings:
##   onset of filler/noise: 3056 ms
##   onset of noun phrase:  4871 ms
fponset <- 3056L
nponset <- 4871L

## keep only the test trials
crit <- trials %>%
  filter(btrial == 5L) %>%
  inner_join(subjects %>% select(-Gender), "SubjID") %>%
  semi_join(blocks %>% filter(blocktype == "critical"), c("vers", "block")) %>%
  inner_join(versions %>% select(vers, block, btrial, target),
             c("vers", "block", "btrial")) %>%
  mutate(fac = (target * 2L) - 1L,
         accurate = Selection == target) %>%
  filter(accurate)

## time-lock and calculate distance of cursor from target
mousedist <- mouse %>%
  inner_join(crit %>% select(TrialID, fac), "TrialID") %>%
  mutate(ms = Msec - fponset,
         sdist = fac * (X - 512) / 100L,
         x = if_else(abs(sdist) > 1, 1 * sign(sdist), sdist),
         win = factor(if_else(ms >= (nponset - fponset),
                              "expression", "filled interval"),
                      levels = c("filled interval", "expression"))) %>%
  filter(ms >= 0L, ms <= 4471L) %>%
  select(TrialID, win, ms, x)

## calculate distance travelled during window
## and match up with block info
dist <- mousedist %>%
  group_by(TrialID, win) %>%
  filter( (ms == min(ms)) | (ms == max(ms)) ) %>%
  mutate(step = paste0("S", row_number())) %>%
  select(-ms) %>%
  pivot_wider(names_from = step, values_from = x) %>%
  mutate(dist = S2 - S1) %>%
  ungroup() %>%
  inner_join(crit %>% select(SubjID, TrialID, vers, block), c("TrialID")) %>%
  inner_join(blocks %>% select(vers, block, spkr, pause),
             c("vers", "block")) %>%
  select(SubjID, block, win, spkr, pause, dist)
```

## Calculate means

``` r
descriptives <- dist %>%
  group_by(win, spkr, pause) %>%
  summarize(mean_dist = mean(dist),
            sd_dist = sd(dist))
```

    ## `summarise()` regrouping output by 'win', 'spkr' (override with `.groups` argument)

| win             | spkr | pause    | mean\_dist | sd\_dist |
| :-------------- | :--- | :------- | ---------: | -------: |
| filled interval | diff | baseline |       0.04 |     0.28 |
| filled interval | diff | filled   |     \-0.01 |     0.33 |
| filled interval | same | baseline |       0.00 |     0.30 |
| filled interval | same | filled   |       0.10 |     0.41 |
| expression      | diff | baseline |       0.60 |     0.61 |
| expression      | diff | filled   |       0.60 |     0.71 |
| expression      | same | baseline |       0.70 |     0.56 |
| expression      | same | filled   |       0.69 |     0.57 |

## Linear Mixed-Effects Model

### Filled interval

``` r
dat <- dist %>%
  mutate(P = if_else(pause == "baseline", -.5, .5),
         S = if_else(spkr == "diff", -.5, .5))

mod_fi <- lmer(dist ~ S * P +
                 (S * P || SubjID) +
                 (S * P || block),
               data = dat %>% filter(win == "filled interval"))
```

    ## boundary (singular) fit: see ?isSingular

``` r
summary(mod_fi)
```

    ## Linear mixed model fit by REML ['lmerMod']
    ## Formula: dist ~ S * P + ((1 | SubjID) + (0 + S | SubjID) + (0 + P | SubjID) +  
    ##     (0 + S:P | SubjID)) + ((1 | block) + (0 + S | block) + (0 +  
    ##     P | block) + (0 + S:P | block))
    ##    Data: dat %>% filter(win == "filled interval")
    ## 
    ## REML criterion at convergence: 679.2
    ## 
    ## Scaled residuals: 
    ##     Min      1Q  Median      3Q     Max 
    ## -5.6347 -0.2451 -0.0427  0.0870  5.6437 
    ## 
    ## Random effects:
    ##  Groups   Name        Variance  Std.Dev.
    ##  SubjID   (Intercept) 0.0059464 0.07711 
    ##  SubjID.1 S           0.0253838 0.15932 
    ##  SubjID.2 P           0.0003054 0.01747 
    ##  SubjID.3 S:P         0.0156026 0.12491 
    ##  block    (Intercept) 0.0000000 0.00000 
    ##  block.1  S           0.0003783 0.01945 
    ##  block.2  P           0.0000000 0.00000 
    ##  block.3  S:P         0.0004942 0.02223 
    ##  Residual             0.0973285 0.31198 
    ## Number of obs: 1075, groups:  SubjID, 92; block, 12
    ## 
    ## Fixed effects:
    ##             Estimate Std. Error t value
    ## (Intercept)  0.03367    0.01247   2.700
    ## S            0.03134    0.02590   1.210
    ## P            0.02969    0.01914   1.551
    ## S:P          0.15745    0.04079   3.860
    ## 
    ## Correlation of Fixed Effects:
    ##     (Intr) S      P     
    ## S   -0.010              
    ## P    0.005 -0.002       
    ## S:P -0.002  0.005 -0.016
    ## convergence code: 0
    ## boundary (singular) fit: see ?isSingular

### Referential expression

``` r
mod_re <- lmer(dist ~ S * P +
                 (S * P || SubjID) +
                 (S * P || block),
               data = dat %>% filter(win == "expression"))
```

    ## boundary (singular) fit: see ?isSingular

``` r
summary(mod_re)
```

    ## Linear mixed model fit by REML ['lmerMod']
    ## Formula: dist ~ S * P + ((1 | SubjID) + (0 + S | SubjID) + (0 + P | SubjID) +  
    ##     (0 + S:P | SubjID)) + ((1 | block) + (0 + S | block) + (0 +  
    ##     P | block) + (0 + S:P | block))
    ##    Data: dat %>% filter(win == "expression")
    ## 
    ## REML criterion at convergence: 1859.6
    ## 
    ## Scaled residuals: 
    ##     Min      1Q  Median      3Q     Max 
    ## -3.5291 -0.6780  0.1039  0.5550  2.8348 
    ## 
    ## Random effects:
    ##  Groups   Name        Variance  Std.Dev. 
    ##  SubjID   (Intercept) 2.395e-02 1.548e-01
    ##  SubjID.1 S           4.163e-02 2.040e-01
    ##  SubjID.2 P           1.033e-09 3.215e-05
    ##  SubjID.3 S:P         1.200e-03 3.464e-02
    ##  block    (Intercept) 5.484e-02 2.342e-01
    ##  block.1  S           1.587e-02 1.260e-01
    ##  block.2  P           2.149e-03 4.636e-02
    ##  block.3  S:P         0.000e+00 0.000e+00
    ##  Residual             2.891e-01 5.377e-01
    ## Number of obs: 1070, groups:  SubjID, 92; block, 12
    ## 
    ## Fixed effects:
    ##              Estimate Std. Error t value
    ## (Intercept)  0.640076   0.071435   8.960
    ## S            0.101991   0.053504   1.906
    ## P           -0.008024   0.035552  -0.226
    ## S:P         -0.012548   0.065966  -0.190
    ## 
    ## Correlation of Fixed Effects:
    ##     (Intr) S      P     
    ## S   -0.002              
    ## P    0.003 -0.001       
    ## S:P  0.000  0.007 -0.012
    ## convergence code: 0
    ## boundary (singular) fit: see ?isSingular
