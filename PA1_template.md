# Reproducible Research: Peer Assessment 1

This assignment makes use of [data from a personal activity monitoring device][1]. 
This device collects data at 5 minute intervals throughout the day. The data 
consists of two months of data from an anonymous individual collected during
the months of October and November, 2012 and include the number of steps 
taken in 5 minute intervals each day.

[1]: https://d396qusza40orc.cloudfront.net/repdata%2Fdata%2Factivity.zip

## Loading the required packages

The following packages are required to perform the analysis.


```r
library("dplyr", warn.conflicts = FALSE)
library("lubridate")
library("ggplot2")
```

## Loading and preprocessing the data

First, the CSV file is extracted from the ZIP file and the data 
is loaded into a `data.frame` called `original_data`.


```r
zip_file <- "activity.zip"
unzip(zip_file, overwrite = TRUE)
csv_file <- "activity.csv"
original_data <- read.csv(csv_file, header = TRUE)
```

Next, the string values in the `date` column are transformed into 
`POSIXct` objects.


```r
original_data <- mutate(original_data, date = ymd(date))
str(original_data)
```

```
## 'data.frame':	17568 obs. of  3 variables:
##  $ steps   : int  NA NA NA NA NA NA NA NA NA NA ...
##  $ date    : POSIXct, format: "2012-10-01" "2012-10-01" ...
##  $ interval: int  0 5 10 15 20 25 30 35 40 45 ...
```

## What is the mean total number of steps taken per day?


```r
daily_activity <- original_data %>%
    group_by(date) %>%
    summarize(total_steps = sum(steps, na.rm = TRUE))

mean_steps <- mean(daily_activity$total_steps)
median_steps <- median(daily_activity$total_steps)
    
ggplot(daily_activity, aes(x = total_steps)) +
    geom_histogram(binwidth = 1000) +
    ggtitle("Original dataset") +
    xlab("Total number of steps") +
    ylab("Count")
```

![](fig/daily_activity_original-1.png) 

The mean and median total number of steps taken per day are 9354.2295082 
and 10395, respectively.

## What is the average daily activity pattern?


```r
activity_pattern <- original_data %>%
    group_by(interval) %>%
    summarize(average_steps = mean(steps, na.rm = TRUE))

i <- which.max(activity_pattern$average_steps)
max_average_steps <- activity_pattern$average_steps[i]
max_interval <- activity_pattern$interval[i]
max_color <- "red"

ggplot(activity_pattern, aes(x = interval, y = average_steps)) +
    geom_line() +
    geom_point(x = max_interval, y = max_average_steps,
               color = max_color, size = 4) +
    xlab("Interval") +
    ylab("Average number of steps")
```

![](fig/activity_pattern-1.png) 

The maximum average number of steps is reached at the 5-minute interval 
835 (red dot).

## Imputing missing values

Calculating the total number of missing values in the dataset:


```r
missing_values <- sum(!complete.cases(original_data))
```

There are 2304 missing values in the dataset. These values will
be filled using the median for the 5-minute interval in question across the 
entire dataset. First, the substitutes for each 5-minute interval are computed:


```r
substitution_data <- original_data %>%
    group_by(interval) %>%
    summarize(substitute = median(steps, na.rm = TRUE))
```

Then, a new `data.frame` called `filled_data` is created from `original_data` 
but with the missing data filled in:


```r
complete_data <- original_data %>% filter(!is.na(steps))

substituted_data <- original_data %>% filter(is.na(steps)) %>%
    left_join(substitution_data, by = "interval") %>%
    mutate(steps = substitute) %>%
    select(steps, date, interval)

filled_data <- rbind(complete_data, substituted_data)
```

Re-create the histogram of the total number of steps taken each day using
the filled data.


```r
daily_activity <- filled_data %>%
    group_by(date) %>%
    summarize(total_steps = sum(steps))

mean_steps <- mean(daily_activity$total_steps)
median_steps <- median(daily_activity$total_steps)

ggplot(daily_activity, aes(x = total_steps)) +
    geom_histogram(binwidth = 1000) +
    ggtitle("Dataset with missing values filled-in") +
    xlab("Total number of steps") +
    ylab("Count")
```

![](fig/daily_activity_filled-1.png) 

In the filled dataset the mean and median total number of steps taken 
per day are 9503.8688525 and 10395, respectively. 
The mean is larger than the value from the first part of the assignment, 
but the median is the same. Since the imputed values are equal or greater 
than zero, the total number of steps taken each day will be equal or greater 
than the values computed for the original dataset.

## Are there differences in activity patterns between weekdays and weekends?


```r
day_of_week <- function (date) {
    weekend_days <- c("Saturday", "Sunday")
    ifelse(weekdays(date) %in% weekend_days, "weekend", "weekday")
}

activity_pattern <- filled_data %>%
    mutate(day_of_week = day_of_week(date)) %>%
    group_by(day_of_week, interval) %>%
    summarize(average_steps = mean(steps, na.rm = TRUE))

ggplot(activity_pattern, aes(x = interval, y = average_steps)) +
    geom_line() +
    facet_grid(day_of_week ~ .) +
    xlab("Interval") +
    ylab("Average number of steps")
```

![](fig/activity_pattern_day_of_week-1.png) 

The maximum average number of steps is higher during weekdays (around 200)
than on weekends (around 150). However, the data suggest an increase of the
activity on weekends between the 1000 and 2000 5-minute intervals.
