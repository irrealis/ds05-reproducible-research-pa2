---
title: "Health and Financial Impact of Severe-Weather Events Between 2007 and 2011"
author: "Irrealis"
date: "July 1, 2018"
output:
    html_document:
        keep_md: true
---

# Synopsis

We began this analysis intending to explore two questions:

- Across the United States, which types of events are most harmful to population health?
- Across the United States, which types of events have the greatest economic cost?

We sought answers in NOAA Storm Database data from 1950 through 2011. Although this spans over 60 years, only five years (2007 through 2011) seemed suitable to comparative analysis, for reasons explained in the _Data Processing_ section. Thus we narrowed the questions' scope to 2007 through 2011.

We analyzed the top-5 average and top-2 year-by-year worst events during this period in terms of injuries, fatalities, and damage to property and crops. We found that tornadoes were unambiguously most harmful in terms of injuries, and also harmful in terms of fatalities. The most harmful events to public health also included excessive heat, flash floods, lightning, rip currents, and thunderstorm winds. The most economically harmful events in terms of property and crop damage included tornadoes, flash floods, frost and freeze, hail, and storm surge and tides.


# Data Processing

Data are loaded from cache if available, otherwise decompressed from an archive if available, and otherwise downloaded (see _Loading data_). Data not required for this analysis are removed in _Trimming unused data_. We then encounter and explore data-quality issues in _Exploring `EVTYPE`_. To troubleshoot these issues, a time series is constructed in _Construction of time series_ and visualized in _Visualizing `EVTYPE` evolution_ where causes are identified. They are addressed in _Subsetting data with correct `EVTYPE`_ by extracting a high-quality subset of data suitable for comparative analysis. Property and crop damages are computed in dollars in _Transforming economic data_. The data are tidied and saved in _Tidying_.


### Setting analysis parameters


```r
# Raw-data URL.
dat_url <- "https://d396qusza40orc.cloudfront.net/repdata%2Fdata%2FStormData.csv.bz2"
# Data subdirectory.
dat_dir <- "data"
# Data directory and raw-/intermediate-/clean-data subdirectories.
raw_dir <- file.path(dat_dir, "raw")
int_dir <- file.path(dat_dir, "intermediate")
tdy_dir <- file.path(dat_dir, "tidy")
# Raw-/cached-/tidy-data paths.
dat_pth <- file.path(raw_dir, "StormData.csv.bz2" )
cch_pth <- file.path(int_dir, "StormData.csv")
tdy_pth <- file.path(tdy_dir, "StormData-Tidy.csv")

# These control rank sizes when computing the top-k worst weather events.
average_k <- 5
by_year_k <- 2

# Cutoff year: observations from prior years are excluded from analysis.
limited_min_year = 2007
```

### Loading data


```r
# Load libraries.
library(data.table)
library(dplyr)
library(ggplot2)
library(lubridate)
library(readr)

# Create any missing directories.
for(d in c(dat_dir, raw_dir, int_dir, tdy_dir)) if(!dir.exists(d)) dir.create(d)

# Download and read StormData.csv.bz2.
if(!file.exists(cch_pth)){
    if(!file.exists(dat_pth)) download.file(dat_url, dat_pth)
    write.csv(read.csv(dat_pth), cch_pth)
}
dat <- fread(cch_pth)
```

### Trimming unused data


```r
# Select columns used in processing.
dat <- dat[, .(EVTYPE, BGN_DATE, BGN_TIME, TIME_ZONE, PROPDMG, PROPDMGEXP, CROPDMG, CROPDMGEXP, FATALITIES, INJURIES)]
```

### Exploring `EVTYPE`

From section 2.1 of the [National Weather Service (NWS) storm data documentation](https://d396qusza40orc.cloudfront.net/repdata%2Fpeer2_doc%2Fpd01016005curr.pdf) we expect 48 storm data event types, but find 985.


```r
length(unique(dat$EVTYPE))
```

```
## [1] 985
```

Reasons include spelling variants and event conflations. For example, `TSTM WIND`, `THUNDERSTORM WINDS`, `THUNDERSTORM WIND`, `THUNDERSTORM WINS` all appear to represent "Thunderstorm Wind", while `ICE STORM/FLASH FLOOD` conflates two distinct event types. Spelling variants require curation prior to processing. Event conflations indicate that different recording methods were used prior to 2007, when such events should be recorded separately. Differences between recording methods may hinder comparative analysis.


```r
unique(dat$EVTYPE)[1:20]
```

```
##  [1] "TORNADO"                   "TSTM WIND"                
##  [3] "HAIL"                      "FREEZING RAIN"            
##  [5] "SNOW"                      "ICE STORM/FLASH FLOOD"    
##  [7] "SNOW/ICE"                  "WINTER STORM"             
##  [9] "HURRICANE OPAL/HIGH WINDS" "THUNDERSTORM WINDS"       
## [11] "RECORD COLD"               "HURRICANE ERIN"           
## [13] "HURRICANE OPAL"            "HEAVY RAIN"               
## [15] "LIGHTNING"                 "THUNDERSTORM WIND"        
## [17] "DENSE FOG"                 "RIP CURRENT"              
## [19] "THUNDERSTORM WINS"         "FLASH FLOOD"
```

Only three event types are recorded for the 1950s (and only TORNADO events for the early '50s). This indicates sampling of different populations at different times, which may hinder comparative analysis.


```r
unique(dat[grepl("195[[:digit:]]", BGN_DATE, ignore.case = F), EVTYPE])
```

```
## [1] "TORNADO"   "TSTM WIND" "HAIL"
```

The [National Oceanic and Atmospheric Administration (NOAA) Storm Events Database page](https://www.ncdc.noaa.gov/stormevents/details.jsp) states that only tornado events are represented for 1950 through 1954, and tornado, thunderstorm wind, and hail for 1955 through 1995, which seems to agree with our findings. But it also states that 48 event types are used for 1996 to present, which disagrees with our 985 distinct types. On the other hand, this suggests that constructing a time series may help to identify and extract a dataset suitable for our analysis.


### Construction of time series

Problems are encountered in the `TIME_ZONE` field, where timezones have noncanonical encodings, a few of which are misspelled. This is addressed by converting to canonical encodings compatible with _R_ and _lubridate_, and by discarding rows with misspelled encodings. A few more misspellings are encountered in the `BGN_TIME` field. Corresponding rows are discarded. New fields `begin_datetime` and `year` are synthesized for time-series analysis.


```r
# To setup a time series, we need to assign dates and times to each event.
# To do this we need to parse BGN_DATE, BGN_TIME, and TIME_ZONE.
# But TIME_ZONE uses old style names that are incompatible with R and lubridate,
# so here we convert to compatible versions.
#
# We also discard four time zones: UNK (unknown?) and ESY, CSC, SCT
# (misspellings of EST and CST?).
dat <- merge(
    dat,
    data.table(
        TIME_ZONE = c("SST", "GST", "UNK", "UTC", "GMT", "ADT", "AST", "EDT", "EST", "ESt", "ESY", "CDT", "CST", "CSt", "CSC", "SCT", "MDT", "MST", "PDT", "PST", "AKS", "HST"),
        timezone = as.factor(c("Etc/GMT-11", "Etc/GMT-10", "NA", "Etc/GMT+0", "Etc/GMT+0", "Etc/GMT+3", "Etc/GMT+4", "Etc/GMT+4", "Etc/GMT+5", "Etc/GMT+5", "NA", "Etc/GMT+5", "Etc/GMT+6", "Etc/GMT+6", "NA", "NA", "Etc/GMT+6", "Etc/GMT+7", "Etc/GMT+7", "Etc/GMT+8", "Etc/GMT+9", "Etc/GMT+10")
    )),
    sort=F
)

## Remove rows with BGN_TIME values that can't be parsed.
dat <- dat[!is.na(parse_date_time(BGN_TIME, c("HM", "HMS")))]

## We can now assign dates and times.
convert_to_datetime <- function(BGN_DATE, BGN_TIME, timezone){
    d <- parse_date_time(BGN_DATE, "mdY HMS")
    t <- parse_date_time(BGN_TIME, c("HM", "HMS"))
    make_datetime(
        year = year(d),
        month = month(d),
        day = day(d),
        hour = hour(t),
        min = minute(t),
        tz = as.character(timezone)
    )
}
dat[, begin_datetime := convert_to_datetime(BGN_DATE, BGN_TIME, timezone)]
dat[, year := year(with_tz(begin_datetime, 'Etc/GMT'))]
```

### Visualizing `EVTYPE` evolution

The numbers of event types in use each year are visualized, as well as additions of new event types and retirements of old ones. High-quality event data are identified spanning a period of five years.



We see that over 400 new event are added between about 1993 and 1995, the majority of which are retired within a few years. The number continues to decline for a decade before stabilizing slightly below 50 around 2005. After 2006, up to 46 are used in each year.

The number of distinct event types used between 2007 and 2011 is
48. They are exactly the permitted storm data events listed in table 1, section 2.1.1 of the [National Weather Service Instruction (NWSI) 10-1605](https://d396qusza40orc.cloudfront.net/repdata%2Fpeer2_doc%2Fpd01016005curr.pdf) dated August 17, 2007 (which, provided to us by our instructors, has in its publication date a hint). (Note: `LANDSLIDE` and "Debris Flow" are the same thing.)


```r
## Convenience variables used a couple of times later.
min_year = min(dat$year)
max_year = max(dat$year)

## Count numbers of event-type adds/dels per year.
uses_by_type <- dat[, .(added = min(year), removed = max(year)+1), by = EVTYPE]
adds_by_year <- uses_by_type[, .(additions = .N), by = added]
dels_by_year <- uses_by_type[, .(removals = .N), by = removed]

## Count number of actively-used event types per year.
mods_by_year <- data.table(year = min_year:max_year) %>%
    merge(
        adds_by_year,
        by.x="year",
        by.y="added",
        all=T
    ) %>%
    merge(
        dels_by_year,
        by.x="year",
        by.y="removed",
        all=T
    )
mods_by_year[is.na(mods_by_year)] = 0
mods_by_year[, active := cumsum(additions) - cumsum(removals)]

## Remove extra year (2012) at end.
## It's there because of `removed = max(year)+1` above.
mods_by_year <- mods_by_year[year <= max(dat$year),]

counts_by_year <- dat[, .(event_types = length(unique(EVTYPE))), by = year][order(year)]
distinct_types_07_to_11 <- unique(dat[limited_min_year <= year, EVTYPE])
```

```r
## Convert to long form for plotting with ggplot2.
long_mods_by_year <- melt(mods_by_year, id.var = c('year'), value.name = 'count')
## Use computed first and last years to setup plot title.
mods_by_year_title = paste("Storm-event type usage from", min_year, "through", max_year)
ggplot(long_mods_by_year, aes(x=year, y=count, color=variable)) +
    geom_line() +
    theme(legend.title=element_blank()) +
    labs(
        title = mods_by_year_title,
        caption = "Storm-event types introduced, used, and retired year-by-year"
    )
```

![](severe-weather-writeup_180701-1604_files/figure-html/unnamed-chunk-8-1.png)<!-- -->

```r
counts_by_year[2005 <= year,]
```

```
##    year event_types
## 1: 2005          46
## 2: 2006          50
## 3: 2007          46
## 4: 2008          46
## 5: 2009          46
## 6: 2010          46
## 7: 2011          46
```

```r
distinct_types_07_to_11
```

```
##  [1] "BLIZZARD"                 "THUNDERSTORM WIND"       
##  [3] "WINTER STORM"             "TORNADO"                 
##  [5] "HAIL"                     "FLASH FLOOD"             
##  [7] "LIGHTNING"                "FUNNEL CLOUD"            
##  [9] "HEAVY SNOW"               "STRONG WIND"             
## [11] "DROUGHT"                  "FROST/FREEZE"            
## [13] "RIP CURRENT"              "HEAVY RAIN"              
## [15] "HEAT"                     "DUST DEVIL"              
## [17] "HIGH SURF"                "EXTREME COLD/WIND CHILL" 
## [19] "HIGH WIND"                "FLOOD"                   
## [21] "WINTER WEATHER"           "ASTRONOMICAL LOW TIDE"   
## [23] "LANDSLIDE"                "COASTAL FLOOD"           
## [25] "ICE STORM"                "STORM SURGE/TIDE"        
## [27] "COLD/WIND CHILL"          "DUST STORM"              
## [29] "WILDFIRE"                 "EXCESSIVE HEAT"          
## [31] "DENSE FOG"                "DENSE SMOKE"             
## [33] "AVALANCHE"                "TROPICAL STORM"          
## [35] "TROPICAL DEPRESSION"      "HURRICANE"               
## [37] "LAKE-EFFECT SNOW"         "SLEET"                   
## [39] "FREEZING FOG"             "LAKESHORE FLOOD"         
## [41] "WATERSPOUT"               "MARINE THUNDERSTORM WIND"
## [43] "MARINE HAIL"              "MARINE STRONG WIND"      
## [45] "MARINE HIGH WIND"         "VOLCANIC ASHFALL"        
## [47] "SEICHE"                   "TSUNAMI"
```

### Subsetting data with correct `EVTYPE` values

Data for years prior to 2007 are discarded. To simplify troubleshooting we map `EVTYPE` values to names listed in NWSI 10-1605. `LANDSLIDE` maps to "Debris Flow", `HURRICANE` to "Hurricane (Typhoon)", and `VOLCANIC ASHFALL` to "Volcanic Ash". The remainder change capitalization only.


```r
dat07 <- merge(
    dat[limited_min_year <= year],
    data.frame(
        EVTYPE = c("ASTRONOMICAL LOW TIDE", "AVALANCHE", "BLIZZARD", "COASTAL FLOOD", "COLD/WIND CHILL", "LANDSLIDE", "DENSE FOG", "DENSE SMOKE", "DROUGHT", "DUST DEVIL", "DUST STORM", "EXCESSIVE HEAT", "EXTREME COLD/WIND CHILL", "FLASH FLOOD", "FLOOD", "FREEZING FOG", "FROST/FREEZE", "FUNNEL CLOUD", "HAIL", "HEAT", "HEAVY RAIN", "HEAVY SNOW", "HIGH SURF", "HIGH WIND", "HURRICANE", "ICE STORM", "LAKE-EFFECT SNOW", "LAKESHORE FLOOD", "LIGHTNING", "MARINE HAIL", "MARINE HIGH WIND", "MARINE STRONG WIND", "MARINE THUNDERSTORM WIND", "RIP CURRENT", "SEICHE", "SLEET", "STORM SURGE/TIDE", "STRONG WIND", "THUNDERSTORM WIND", "TORNADO", "TROPICAL DEPRESSION", "TROPICAL STORM", "TSUNAMI", "VOLCANIC ASHFALL", "WATERSPOUT", "WILDFIRE", "WINTER STORM", "WINTER WEATHER"),
        event_type = c("Astronomical Low Tide", "Avalanche", "Blizzard", "Coastal Flood", "Cold/Wind Chill", "Debris Flow", "Dense Fog", "Dense Smoke", "Drought", "Dust Devil", "Dust Storm", "Excessive Heat", "Extreme Cold/Wind Chill", "Flash Flood", "Flood", "Freezing Fog", "Frost/Freeze", "Funnel Cloud", "Hail", "Heat", "Heavy Rain", "Heavy Snow", "High Surf", "High Wind", "Hurricane (Typhoon)", "Ice Storm", "Lake-Effect Snow", "Lakeshore Flood", "Lightning", "Marine Hail", "Marine High Wind", "Marine Strong Wind", "Marine Thunderstorm Wind", "Rip Current", "Seiche", "Sleet", "Storm Surge/Tide", "Strong Wind", "Thunderstorm Wind", "Tornado", "Tropical Depression", "Tropical Storm", "Tsunami", "Volcanic Ash", "Waterspout", "Wildfire", "Winter Storm", "Winter Weather")
    ),
    sort=F
)
```


### Transforming economic data

Symbols 'K', 'M', and 'B' in fields `PROPDMGEXP` and `CROPDMGEXP` encode thousands, millions, and billions of dollars, respectively. This usage is consistent with NWSI 10-1605 documentation. We transform these encodings to numeric multipliers in order to synthesize property- and crop-damage values.



```r
# To transform economic-damage data to dollars, we need to multiply
# PROPDMG/CROPDMG by powers of ten encoded in PROPDMGEXP/CROPDMGEXP,
# so we first decode the latter pair of fields.
damage_exponent_transform <- data.frame(c("B", "M", "K", "0"), c(1e9, 1e6, 1e3, 1e0))

# Convert PROPDMGEXP, then transform property-damage data to dollars.
names(damage_exponent_transform) <- c("PROPDMGEXP", 'pdx')
dat07 <- merge(dat07, damage_exponent_transform, sort=F)
dat07[, property_damage := PROPDMG * as.numeric(as.character(pdx))]

# Convert CROPDMGEXP, then transform crop-damage data to dollars.
names(damage_exponent_transform) <- c("CROPDMGEXP", 'cdx')
dat07 <- merge(dat07, damage_exponent_transform, sort=F)
dat07[, crop_damage := CROPDMG * as.numeric(as.character(cdx))]
```


### Tidying

We finish tidying data and then save, discarding fields we will not analyze. The remainder have the following meanings:

- `event_type`: One of 48 permitted storm-event types listed in NWSI 10-1605 as of August 17, 2007. Type: factor.
- `begin_datetime`: The date and time at start of the event. Type: `POSIXct`.
- `injuries`: The number of direct injuries caused by the event. Type: integer.
- `fatalities`: The number of direct fatalities caused by the event. Type: integer.
- `property_damage`: The dollar amount of direct property damage caused by the event. Type: numeric.
- `crop_damage`: The dollar amount of direct crop damage caused by event. Type: numeric.


```r
tdy <- dat07[, .(year, event_type, injuries = INJURIES, fatalities = FATALITIES, property_damage, crop_damage)]
fwrite(tdy, tdy_pth)
```




# Quality assessment

We believe that our usable data cannot answer the original questions. We develop an explanation below, first discussing problems for comparative analysis, their solutions, and problems caused by those solutions. The latter prevent answering our original questions. In their place we will pose limited versions our data _can_ answer.

In the original data, the number of distinct event types in use during any given year ranges from one to over 400. This has at least three causes:

- At different times, very different populations of event types were sampled.
- At different times, very different recording methods were used. At times concurrent events of different types were sytematically recorded in single observations. at other times such events were systematically recorded separately.
- There are many examples of events of a single type designated at different times by different spellings.

Different populations and methods at different times result in incomparable observations that hinder comparative analysis. To some extent a devoted expert may be able to normalize such records using information in event narratives and other historical records. Similar by-hand curation may resolve spelling variants that hinder data-processing. We address these issues by constraining our dataset to a five year period for which data encodings use only the 48 permitted events listed in NWSI 10-1605.

But in doing so we discard over 50 years of data. The remainder span five years, or less than ten percent. Moreover, such global weather patterns as El Niño events have often lasted more than five years, with strong interspersed La Niña events. El Niǹo and La Niǹa events are associated with periods of local weather whose characteristics vary accordingly. We do not trust that such variations are comparable. Because the period spanned by our remaining data coincides with predominantly La Niña events, we suspect that the conclusions in this analysis cannot generalize to certain other, or longer, time spans, so we do not attempt to answer with generality the posed questions.

Based on this assessment we instead seek answers to the following:

- Across the United States _from 2007 to 2011_, which types of weather events were most harmful to population health?
- Across the United States _from 2007 to 2011_, which types of weather events had the greatest economic cost?


# Results

We will first explore average health and economic consequences of different types of storm events, then consider consequences year-by-year, exploring variation from averages consequences.


### Overall consequences of storm events



From 2007 through 2011, tornadoes appear to be by far the worst of storm events in
terms of injuries (9,608),
fatalities (863),
and property damage (\$1.5e+10). Floods appear to be
worst in terms of crop damage (\$2.9e+09). Second rank are floods in
property damage (\$1.4e+10), frost and freezes
in crop damage (\$9.3e+08),
thunderstorm winds for injuries (1,391), and flash floods for fatalities (293).


```r
counts_by_type <- tdy[
    ,
    .(n = .N),
    by = event_type
][order(event_type)]
sums_by_type <- tdy[
    ,
    lapply(data.table(injuries, fatalities, property_damage, crop_damage), sum),
    by = event_type
][order(event_type)]
counts_and_sums_by_type <- merge(
    counts_by_type,
    sums_by_type,
    sort=F
)[order(event_type)]

totals <- counts_and_sums_by_type %>%
    select(
        -event_type,
        total_events = n,
        total_injuries = injuries,
        total_fatalities = fatalities,
        total_property_damage = property_damage,
        total_crop_damage = crop_damage
    ) %>%
    summarize_all(sum)
proportions_by_type <- cbind(counts_and_sums_by_type, totals) %>%
    transmute(
        event_type = event_type,
        frequency = n/total_events,
        injuries = injuries/total_injuries,
        fatalities = fatalities/total_fatalities,
        property_damage = property_damage/total_property_damage,
        crop_damage = crop_damage/total_crop_damage,
    ) %>% data.table()

# I haven't figured out how to make top_n behave, so here's my version.
top_k <- function(data, k, sort_column, ...) {
    sort_column <- rlang::enexpr(sort_column)
    extra_columns <- rlang::quos(...)
    data %>%
        select(!!!extra_columns, !!sort_column) %>%
        do(arrange(., desc(!!sort_column)) %>% head(., k))
}
top_k_average_worst <- counts_and_sums_by_type %>% {
    list(
        frequency = top_k(., average_k, n, event_type),
        injuries = top_k(., average_k, injuries, event_type),
        fatalities = top_k(., average_k, fatalities, event_type),
        property_damage = top_k(., average_k, property_damage, event_type),
        crop_damage = top_k(., average_k, crop_damage, event_type)
    )
}
proportional_top_k_average_worst <- proportions_by_type %>% {
    list(
        frequency = top_k(., average_k, frequency, event_type),
        injuries = top_k(., average_k, injuries, event_type),
        fatalities = top_k(., average_k, fatalities, event_type),
        property_damage = top_k(., average_k, property_damage, event_type),
        crop_damage = top_k(., average_k, crop_damage, event_type)
    )
}
proportional_average_harm <- rbindlist(lapply(proportional_top_k_average_worst, function(x) {
    melt(x, id.var = c("event_type"), value.name = "proportional_harm")
}))

# Consolidate results.
average_worst_for_injuries <- top_k_average_worst$injuries$event_type
average_worst_for_fatalities <- top_k_average_worst$fatalities$event_type
average_worst_for_health <- sort(union(average_worst_for_injuries, average_worst_for_fatalities))
average_worst_for_property <- top_k_average_worst$property_damage$event_type
average_worst_for_crops <- top_k_average_worst$property_damage$event_type
average_worst_for_economy <- sort(union(average_worst_for_property, average_worst_for_crops))
average_worst <- sort(union(average_worst_for_health, average_worst_for_economy))
```


```r
average_worst_title = paste("Proportional harm by storm events,", limited_min_year, "through", max_year)
ggplot(
    filter(proportional_average_harm, variable != "frequency"),
    aes(x=event_type, y=proportional_harm)
) +
    geom_bar(stat = "identity") +
    coord_flip() +
    labs(
        title = average_worst_title,
        x = "event type",
        y = "proportional harm",
        caption =
            "Overall worst U.S. storm events and relative harm in terms of injuries, fatalities, and
            property- and crop-damage value. Top 5 per category."
    ) +
    facet_wrap(~ variable)
```

![](severe-weather-writeup_180701-1604_files/figure-html/unnamed-chunk-13-1.png)<!-- -->


```r
top_k_average_worst
```

```
## $frequency
##          event_type     n
## 1 Thunderstorm Wind 80665
## 2              Hail 72253
## 3       Flash Flood 19096
## 4             Flood 12450
## 5         High Wind 10390
## 
## $injuries
##          event_type injuries
## 1           Tornado     9608
## 2 Thunderstorm Wind     1391
## 3         Lightning      923
## 4    Excessive Heat      880
## 5              Heat      702
## 
## $fatalities
##    event_type fatalities
## 1     Tornado        863
## 2 Flash Flood        293
## 3 Rip Current        207
## 4        Heat        182
## 5       Flood        161
## 
## $property_damage
##         event_type property_damage
## 1          Tornado     14629323740
## 2            Flood     13969305800
## 3             Hail      6098997600
## 4      Flash Flood      5040672130
## 5 Storm Surge/Tide      4640643000
## 
## $crop_damage
##     event_type crop_damage
## 1        Flood  2886110000
## 2 Frost/Freeze   931801000
## 3         Hail   868793000
## 4  Flash Flood   711942000
## 5      Drought   425416000
```

```r
proportional_top_k_average_worst
```

```
## $frequency
##          event_type  frequency
## 1 Thunderstorm Wind 0.31619941
## 2              Hail 0.28322514
## 3       Flash Flood 0.07485457
## 4             Flood 0.04880286
## 5         High Wind 0.04072785
## 
## $injuries
##          event_type   injuries
## 1           Tornado 0.60446681
## 2 Thunderstorm Wind 0.08751180
## 3         Lightning 0.05806858
## 4    Excessive Heat 0.05536332
## 5              Heat 0.04416483
## 
## $fatalities
##    event_type fatalities
## 1     Tornado 0.32334208
## 2 Flash Flood 0.10977894
## 3 Rip Current 0.07755714
## 4        Heat 0.06819033
## 5       Flood 0.06032222
## 
## $property_damage
##         event_type property_damage
## 1          Tornado      0.25792181
## 2            Flood      0.24628539
## 3             Hail      0.10752818
## 4      Flash Flood      0.08886941
## 5 Storm Surge/Tide      0.08181670
## 
## $crop_damage
##     event_type crop_damage
## 1        Flood  0.41978603
## 2 Frost/Freeze  0.13553089
## 3         Hail  0.12636634
## 4  Flash Flood  0.10355229
## 5      Drought  0.06187695
```


### Year-by-year consequences of storm events




In this section we compare proportions of year-by-year total injuries, fatalities, property damages, and crop damages from 2007 through 2011.

We find that tornadoes consistently caused the highest proportions of injuries, with rates ranging from
29%
to
79%
of totals. They also tended to cause the highest proportions of fatalities with rates as high as
59%,
although
rip currents
during 2009 and
flash floods
during 2010 caused higher proportions at
12%
and
16%,
respectively.

The causes of proportionally highest property damages are more ambiguous, with tornadoes worst in 2007
(24%)
and 2011
(24%),
storm surges in 2008
(30%),
and hail in 2009
(28%)
and 2010
(37%)
. In terms of crop damage, floods were worst in four out of five years, with proportions of total damage ranging from
23%
to
62%
.
Hail
was worst in 2009 at
67%
.

Tornadoes seemed to be unambiguously most harmful to public health in terms of both injuries and fatalities. In terms of fatalities and damage to property and crops, the plots below reveal considerable variation in worst events. In order to visualize this variation over time, the data were normalized to allow comparison of different years. The impacts of weather events are shown as proportions of total harm. The plot of injuries seems to show the worst events, in this case tornadoes, most clearly. Ambiguity seems to increase in the plots of fatalities, property damage, and crop damage.

This strengthens our suspicion (see _Quality assessment_) that the overall conclusions of this analysis may not generalize well.



```r
counts_by_year_and_type <- tdy[
    ,
    .(n = .N),
    by = .(year, event_type)
][order(year, event_type)]
sums_by_year_and_type <- tdy[
    ,
    lapply(data.table(injuries, fatalities, property_damage, crop_damage), sum),
    by = .(year, event_type)
][order(event_type)]
counts_and_sums_by_year_and_type <- merge(
    counts_by_year_and_type,
    sums_by_year_and_type,
    sort=F
)[order(year, event_type)]
counts_and_sums_by_year <- counts_and_sums_by_year_and_type[
    ,
    lapply(data.table(n, injuries, fatalities, property_damage, crop_damage), sum),
    by = year
]
proportions_by_year_and_type <- merge(
    counts_and_sums_by_year_and_type,
    counts_and_sums_by_year,
    by=c("year"),
    suffixes=c("", "_annual")
)[
    ,
    .(
        year = year,
        event_type = event_type,
        frequency = n/n_annual,
        injuries = injuries/injuries_annual,
        fatalities = fatalities/fatalities_annual,
        property_damage = property_damage/property_damage_annual,
        crop_damage = crop_damage/crop_damage_annual
    )
]

group_top_k <- function(data, k, sort_column, by_columns, ...) {
    sort_column <- rlang::enexpr(sort_column)
    by_columns <- rlang::enexpr(by_columns)
    if(typeof(by_columns) == "language"){ by_columns <- rlang::lang_args(by_columns) }
    extra_columns <- rlang::quos(...)
    data %>%
        select(!!!by_columns, !!!extra_columns, !!sort_column) %>%
        group_by(!!!by_columns) %>%
        do(arrange(., desc(!!sort_column)) %>% head(., k))
}
top_k_worst_by_year <- proportions_by_year_and_type %>% {
    list(
        frequency = group_top_k(., by_year_k, frequency, year, event_type),
        injuries = group_top_k(., by_year_k, injuries, year, event_type),
        fatalities = group_top_k(., by_year_k, fatalities, year, event_type),
        property_damage = group_top_k(., by_year_k, property_damage, year, event_type),
        crop_damage = group_top_k(., by_year_k, crop_damage, year, event_type)
    )
}

year_by_year_harm <- rbindlist(lapply(top_k_worst_by_year, function(x) {
    melt(x, id.var = c("year", "event_type"), value.name = "harm")
}))

# Consolidate results.
worst_by_year_for_injuries <- top_k_worst_by_year$injuries$event_type
worst_by_year_for_fatalities <- top_k_worst_by_year$fatalities$event_type
worst_by_year_for_health <- sort(union(worst_by_year_for_injuries, worst_by_year_for_fatalities))
worst_by_year_for_property <- top_k_worst_by_year$property_damage$event_type
worst_by_year_for_crops <- top_k_worst_by_year$property_damage$event_type
worst_by_year_for_economy <- sort(union(worst_by_year_for_property, worst_by_year_for_crops))
worst_by_year <- sort(union(worst_by_year_for_health, worst_by_year_for_economy))

# Compare annual and year-by-year results.
common_events <- intersect(average_worst, worst_by_year)
only_in_average_events <- setdiff(average_worst, worst_by_year)
only_in_by_year_events <- setdiff(worst_by_year, average_worst)

tornado_injuries_min <- year_by_year_harm[(event_type == "Tornado")&(variable == "injuries"), 100*min(harm)]
tornado_injuries_max <- year_by_year_harm[(event_type == "Tornado")&(variable == "injuries"), 100*max(harm)]
tornado_fatalities_max <- year_by_year_harm[(event_type == "Tornado")&(variable == "fatalities"), 100*max(harm)]
flood_crop_min <- year_by_year_harm[(event_type == "Flood")&(variable == "crop_damage"), 100*min(harm)]
flood_crop_max <- year_by_year_harm[(event_type == "Flood")&(variable == "crop_damage"), 100*max(harm)]

max_fatalities_09 <- year_by_year_harm[(year==2009)&(variable == "fatalities"), 100*harm][1]
event_fatalities_09 <- year_by_year_harm[(year==2009)&(variable == "fatalities"), tolower(event_type)][1]

max_fatalities_10 <- year_by_year_harm[(year==2010)&(variable == "fatalities"), 100*harm][1]
event_fatalities_10 <- year_by_year_harm[(year==2010)&(variable == "fatalities"), tolower(event_type)][1]

tornado_property_max_07 <- year_by_year_harm[(year==2007)&(event_type == "Tornado")&(variable == "property_damage"), 100*harm]
storm_surge_max_08 <- year_by_year_harm[(year==2008)&(event_type == "Storm Surge/Tide")&(variable == "property_damage"), 100*harm]
hail_prop_max_09 <- year_by_year_harm[(year==2009)&(event_type == "Hail")&(variable == "property_damage"), 100*harm]
hail_crop_max_09 <- year_by_year_harm[(year==2009)&(event_type == "Hail")&(variable == "crop_damage"), 100*harm]
hail_prop_max_10 <- year_by_year_harm[(year==2010)&(event_type == "Hail")&(variable == "property_damage"), 100*harm]
tornado_property_max_11 <- year_by_year_harm[(year==2011)&(event_type == "Tornado")&(variable == "property_damage"), 100*harm]
```


```r
long_proportions_by_year_and_type <- melt(
    proportions_by_year_and_type,
    id.var = c('year', 'event_type'),
    value.name = "proportion"
)
worst_by_year_and_type <- with(top_k_worst_by_year,
    long_proportions_by_year_and_type %>%
    filter((
        (event_type %in% injuries$event_type) & (variable == 'injuries')
    )|( (event_type %in% fatalities$event_type) & (variable == 'fatalities')
    )|( (event_type %in% property_damage$event_type) & (variable == 'property_damage')
    )|( (event_type %in% crop_damage$event_type) & (variable == 'crop_damage')
    ))
)
worst_by_year_title = paste("Relative harm by storm events,", limited_min_year, "through", max_year)
ggplot(worst_by_year_and_type, aes(x=year, y=proportion, color=event_type)) +
    geom_line() +
    facet_wrap(~ variable) +
    guides(color = guide_legend(title="Event type")) +
    labs(
        title = worst_by_year_title,
        caption =
            "Year-by-year worst U.S. storm events and relative harm
            in terms of injuries, fatalities, and property- and crop-damage value"
    )
```

![](severe-weather-writeup_180701-1604_files/figure-html/unnamed-chunk-15-1.png)<!-- -->


```r
top_k_worst_by_year
```

```
## $frequency
## # A tibble: 10 x 3
## # Groups:   year [5]
##     year event_type        frequency
##    <dbl> <fct>                 <dbl>
##  1  2007 Thunderstorm Wind     0.300
##  2  2007 Hail                  0.294
##  3  2008 Hail                  0.315
##  4  2008 Thunderstorm Wind     0.302
##  5  2009 Thunderstorm Wind     0.292
##  6  2009 Hail                  0.290
##  7  2010 Thunderstorm Wind     0.329
##  8  2010 Hail                  0.227
##  9  2011 Thunderstorm Wind     0.349
## 10  2011 Hail                  0.286
## 
## $injuries
## # A tibble: 10 x 3
## # Groups:   year [5]
##     year event_type        injuries
##    <dbl> <fct>                <dbl>
##  1  2007 Tornado             0.301 
##  2  2007 Excessive Heat      0.245 
##  3  2008 Tornado             0.625 
##  4  2008 Thunderstorm Wind   0.0936
##  5  2009 Tornado             0.293 
##  6  2009 Lightning           0.148 
##  7  2010 Tornado             0.377 
##  8  2010 Thunderstorm Wind   0.174 
##  9  2011 Tornado             0.791 
## 10  2011 Heat                0.0784
## 
## $fatalities
## # A tibble: 10 x 3
## # Groups:   year [5]
##     year event_type  fatalities
##    <dbl> <fct>            <dbl>
##  1  2007 Tornado         0.192 
##  2  2007 Flash Flood     0.166 
##  3  2008 Tornado         0.264 
##  4  2008 Flash Flood     0.113 
##  5  2009 Rip Current     0.117 
##  6  2009 Lightning       0.102 
##  7  2010 Flash Flood     0.158 
##  8  2010 Rip Current     0.115 
##  9  2011 Tornado         0.586 
## 10  2011 Flash Flood     0.0679
## 
## $property_damage
## # A tibble: 10 x 3
## # Groups:   year [5]
##     year event_type          property_damage
##    <dbl> <fct>                         <dbl>
##  1  2007 Tornado                       0.242
##  2  2007 Flash Flood                   0.212
##  3  2008 Storm Surge/Tide              0.295
##  4  2008 Hurricane (Typhoon)           0.155
##  5  2009 Hail                          0.275
##  6  2009 Thunderstorm Wind             0.267
##  7  2010 Hail                          0.368
##  8  2010 Flood                         0.335
##  9  2011 Tornado                       0.470
## 10  2011 Flood                         0.369
## 
## $crop_damage
## # A tibble: 10 x 3
## # Groups:   year [5]
##     year event_type        crop_damage
##    <dbl> <fct>                   <dbl>
##  1  2007 Flood                   0.307
##  2  2007 Frost/Freeze            0.224
##  3  2008 Flood                   0.488
##  4  2008 Flash Flood             0.186
##  5  2009 Hail                    0.672
##  6  2009 Frost/Freeze            0.124
##  7  2010 Flood                   0.619
##  8  2010 Frost/Freeze            0.198
##  9  2011 Flood                   0.232
## 10  2011 Thunderstorm Wind       0.210
```

```r
common_events
```

```
##  [1] "Excessive Heat"    "Flash Flood"       "Flood"            
##  [4] "Hail"              "Heat"              "Lightning"        
##  [7] "Rip Current"       "Storm Surge/Tide"  "Thunderstorm Wind"
## [10] "Tornado"
```

```r
only_in_average_events
```

```
## character(0)
```

```r
only_in_by_year_events
```

```
## [1] "Hurricane (Typhoon)"
```

### Comparison of overall and year-by-year consequences

#### Weather events most harmful to public health

The worst events in terms of injuries appear to be tornados, followed by thunderstorm winds, lightening, and excessive heat. Overall, from 2007 through 2011, tornadoes caused the most injuries each year.

Less clear is the worst cause of fatalities. In most years, tornadoes are worst, but rip currents caused more fatalities in 2009, and flash floods in 2010. Overall, tornadoes are worst, followed by flash floods and rip currents.

#### Most economically harmful weather events

Overall, tornadoes cause the greatest property damage, followed by floods and hail. However, this does not seem to generalize well. Tornadoes were most costly to property in 2007 and 2011; storm surge and tides in 2008; and hail in 2009 and 2010.

In terms of crop damage, floods tend to be most costly, although hail was worse in 2009. Frost and freezing were second-most costly in four out of five years. Overall, floods are worst, followed by frost and freeze, then hail.



