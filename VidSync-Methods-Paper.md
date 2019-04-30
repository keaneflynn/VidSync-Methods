Videogrammetry Metrics for Assessing Fish Foraging Behavior: Implications for Behavioral Ecology and Management
================
Keane Flynn
4/30/2019

**Introduction**
================

    Fish foraging behavior has long been a topic of natural observation, wonder, and recreational curiosity (Walton 1655). In recent years, this area of observation has extended beyond the realm as an enigmatic phenomenon and taken its place as a series of qualitative metrics in fisheries ecology. But with recent advances in computer technologies and programming languages, we are capable of extending these qualitatives behavioral metrics into quantitative values which can be further mathematically and statistically analyzed to show distinct differences in behavior. These differences in foraging behaviors have been shown to be linked to changes in physical habitat characteristics that might be indicative of decreasing habitat quality for foraging fish (Rossi et al. 2019 in prep; Steingrímmson 2012; Fausch et al. 1997). 
    Using the three dimensional videogrammetric program VidSync, quantifying previously generalized qualitative/quantitative values has become a relatively easy process that has the potential to offer results with more statistical significance than previous behavioral observation methods (Neuswanger 2014). This method makes use of two underwater cameras mounted in a fixed location relative to one another recording simultaneous video of fish behavior. Doing so allows for an almost entirely non-invasive way to record fish behavior, whereas many current methods rely on snorkel observations. Once the videos have been processed, an   X, Y, Z coordinate, a timecode, and a behavioral observation type are assigned to each point created for the video sample in the program. These coordinates, timecodes, and observations can then be exported and processed into code-based programs to create statistical and graphical representations of the data collected.
    This program offers a number of advantages to traditional fish foraging observations, the largest of which being ease of quantifying spatial habitat use metrics within sub-millimeter accuracy, which will be expanded upon in the methods section. Additionally, this method of observation allows to expand forage observations beyond focal animal sampling, which has been the most common method for recording fish behavior in natural experiments, to other methods of behavior sampling methods such as sampling all occurrences of behaviors, sequence sampling, and scan sampling (Altmann 1974). While previous field observation methods could only account for one of these observation methods based on physical restrictions, replayable video allows for a higher confidence in behavioral observation assignment. Being able to track multiple individuals throughout a video sample allows for a complete assessment of behavioural observations for all individuals in the sample, hence creating a larger sample size for improved statistical analysis.
    In previous research, these estimations of distances travelled, area of habitat, size of subjects, specific foraging patches, etc. conducted during snorkel surveys were capable of assigning numeric values to these metrics, but with a higher degree of uncertainty (Nakano et al. 1999; Nielsen 1992; Steingrímsson & Grant 2008). While some of these methods have proved useful for changes in habitat usage, these methods are inherently based on location estimations and lack critical pieces of information for properly classifying habitat use. The issue with these previously used methods is that the metrics for habitat use is generally two-dimensional, which disregards many phenomenon such as physical habitat characteristics and location of prey input that might affect location of fish along the Z-axis (Figure 1).

![**Figure 1** *Profile view of a pool-riffle sequence with X, Y, Z planes relative to a fish maintaining positive rheotaxis.*](XYZPlanes.png)

    We hope to address some of the potential uses of these methods, but also realize that there are situations in which it might not be in the best interest to use these methods: they involve more post-capture editing and analyzation than that of snorkel observations, plotting movement movement points and territories can be more complex than doing so using the animal movement extension in ArcView (Hooge & Eichenlaub 2000). Given the resources to use this method are available, we find that it has many attributes that make up for the shortcomings of previously used methods. In this paper, we will elaborate upon the metrics that can be accurately derived using VidSync and the ecological implications of each. 

Load Libraries
--------------

Importing VidSync CSV File
==========================

``` r
VidSync_Test_Data <- readr::read_csv(file = "Porter_BACI_RoachRun_5July2018_Part1.csv", 
                             skip = 2,
                             col_names = c("objects", "event", "timecode", "time_sec", "X", "Y", "Z", "pld_error", "projection_error", "nearest_camera_distance", "screen_coordinates"),
                    col_types = "cccdddddddd") %>% 
  mutate(subsample = as.numeric(str_extract(objects, "\\d"))) %>%
  mutate(fish_ID = as.numeric(str_extract(objects, "\\h\\d{1,2}"))) %>%
  mutate(species = str_extract(objects, "Omykiss|Okisutch")) %>% #Note that the use of regular expressions in this line are to be specified to whatever species are recorded in your experiment
  select(subsample, fish_ID, species, time_sec, X, Y, Z, objects, event) #All other data vectors were excluded for the sake of neatness and what is useful in the metrics below
```

    ## Warning in rbind(names(probs), probs_f): number of columns of result is not
    ## a multiple of vector length (arg 1)

    ## Warning: 475 parsing failures.
    ## row # A tibble: 5 x 5 col     row col   expected   actual     file                                   expected   <int> <chr> <chr>      <chr>      <chr>                                  actual 1     1 <NA>  11 columns 10 columns 'Porter_BACI_RoachRun_5July2018_Part1… file 2     2 <NA>  11 columns 10 columns 'Porter_BACI_RoachRun_5July2018_Part1… row 3     3 <NA>  11 columns 10 columns 'Porter_BACI_RoachRun_5July2018_Part1… col 4     4 <NA>  11 columns 10 columns 'Porter_BACI_RoachRun_5July2018_Part1… expected 5     5 <NA>  11 columns 10 columns 'Porter_BACI_RoachRun_5July2018_Part1…
    ## ... ................. ... .......................................................................... ........ .......................................................................... ...... .......................................................................... .... .......................................................................... ... .......................................................................... ... .......................................................................... ........ ..........................................................................
    ## See problems(...) for more details.

Nearest Neighbor Distance (NND)
===============================

NND is calculated by finding the nearest neighbor to each unique fish over the course of a subsample. The final NND column will represent the closest nearest neighbor for each fish and exclude any larger distances (in cm) so as to not skew the data away from any values that are the absolute nearest neighbor point over the course of the subsample. It is important to note that subsamples with only one observed fish will produce an NND that is infinite and should be dealt with accordingly when graphing/calculating statistics on these values.

``` r
NND_Data <- VidSync_Test_Data %>% 
  filter(grepl("^Subsample.*", objects)) %>%
  filter(!grepl("^Length.*", event)) %>%
  filter(!grepl("^Attack.*", event)) %>%
  filter(!grepl("^Nip.*", event)) %>%
  na.omit() %>% #To remove any additional rows from the data set not associated with fish (i.e. surface shots)
  arrange(time_sec) %>%
  group_by(time_sec) %>%
  mutate(distance_between_X = X - lag(X, default = first(X))) %>%
  mutate(distance_between_Y = Y - lag(Y, default = first(Y))) %>%
  mutate(distance_between_Z = Z - lag(Z, default = first(Z))) %>%
  mutate(nnd_cm = sqrt((distance_between_X)^2 + 
                      (distance_between_Y)^2 +
                      (distance_between_Z)^2)) %>%
  group_by(time_sec) %>%
  filter(!nnd_cm == 0) %>%
  filter(nnd_cm == min(nnd_cm)) %>%
  ungroup() %>% 
  distinct(fish_ID, .keep_all = TRUE) %>% 
  arrange(subsample, fish_ID, time_sec) %>%
  select(subsample, fish_ID, time_sec, nnd_cm) 
```

Distance per Time
=================

``` r
DistancePerTime_Data <- VidSync_Test_Data %>% 
  filter(!grepl("Surface_Shots.*", objects)) %>%
  filter(!grepl("^Length.*", event)) %>%
  group_by(fish_ID) %>% 
  mutate(distance_travelled_X_cm = X - lag(X, default = first(X))) %>%
  mutate(distance_travelled_Y_cm = Y - lag(Y, default = first(Y))) %>%
  mutate(distance_travelled_Z_cm = Z - lag(Z, default = first(Z))) %>%
  mutate(fish_distance_travelled_cm = sqrt((distance_travelled_X_cm)^2
                                        + (distance_travelled_Y_cm)^2
                                        + (distance_travelled_Z_cm)^2)) %>%
  ungroup() %>% 
  group_by(fish_ID) %>%
  mutate(distance_cm_per_sec = fish_distance_travelled_cm /(time_sec - lag(time_sec, default = first(time_sec)))) %>%
  filter(!distance_cm_per_sec == Inf) %>% 
  arrange(subsample, fish_ID, time_sec) %>%
  select(subsample, fish_ID, time_sec, distance_cm_per_sec)
```

Proportion of Forage Behaviors
==============================

``` r
ForageBehaviors_Data <- VidSync_Test_Data %>% 
  filter(!grepl("Surface_Shots.*", objects)) %>%
  filter(!grepl("^Length.*", event)) %>%
  mutate(Behaviors = if_else(grepl("^Drift_Forage", event), "Drift Forage", 
                     if_else(grepl("^Search_Forage", event), "Search Forage", 
                     if_else(grepl("^Search_Forage", event), "Search Forage", 
                     if_else(grepl("^Benthic_Forage", event), "Benthic Forage", 
                     if_else(grepl("Nip|Attack", event), "Attack", 
                     if_else(grepl("^Surface_Strike", event), "Surface Strike", 
                     if_else(grepl("^Movement", event), "Movement", "NA")))))))) %>% #Note that "nip" has since been changed to the definition of "attack" and attack to feint
  arrange(subsample, fish_ID, time_sec) %>% 
  select(subsample, fish_ID, time_sec, Behaviors)
```

Centroid to Forage Attempt Point
================================

Distance from Surface
=====================

Length Calculations
===================
