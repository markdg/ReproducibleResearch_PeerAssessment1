# Reproducible Research: Peer Assessment 1

## The Data Set

The data set comes with the git repo at http://github.com/rdpeng/RepData_PeerAssessment1 when it is forked and cloned. It is contained in the file activity.zip.

The variables included in this dataset are:
- steps: Number of steps taking in a 5-minute interval (missing values are coded as NA)
- date: The date on which the measurement was taken in YYYY-MM-DD format
- interval: Identifier for the 5-minute interval in which measurement was taken

## Loading and preprocessing the data

Assignment statement:
>Show any code that is needed to  
>1. Load the data (i.e. read.csv())  
>2. Process/transform the data (if necessary) into a format suitable for your analysis

Transformations performed here are:  
1. Remove intervals that have no data and save in data frame named cleanDat  
2. Bin the steps data into daily totals and save in a data frame named dailySteps


```r
# Data has already been downloaded and unzipped in the working directory, now read it
dat <- read.csv("activity.csv")

# Remove the intervals that contain NAs and save in data frame named cleanDat
cleanDat <- dat[complete.cases(dat),]

# Use plyr to sum the total steps for each day and save in data frame named dailySteps
library(plyr)
dailySteps <- ddply(cleanDat[-3], .(date), numcolwise(sum))
```

## What is mean total number of steps taken per day?

Assignment statement:
>For this part of the assignment, you can ignore the missing values in the dataset.  
>1. Make a histogram of the total number of steps taken each day  
>2. Calculate and report the mean and median total number of steps taken per day

### Histogram of the number of steps taken per day.

The default of five bins is a bit coarse so the histogram is set to 10 bins.


```r
# Plot histogram with 10 bins
hist(dailySteps$steps, col="steelblue", breaks=10, xlab="Daily Steps", ylab="Number of Days", main="Histogram of Number of Steps per Day")
```

![plot of chunk unnamed-chunk-2](figure/unnamed-chunk-2.png) 

### Mean and median total number of steps taken per day

Mean and median are computed on the data set after missing data have been removed.


```r
mean(dailySteps$steps)
```

```
## [1] 10766
```

```r
median(dailySteps$steps)
```

```
## [1] 10765
```

## What is the average daily activity pattern?

Assignment statement:  
>1. Make a time series plot (i.e. type = "l") of the 5-minute interval (x-axis) and the average number of steps taken, averaged across all days (y-axis)  
>2. Which 5-minute interval, on average across all the days in the dataset, contains the maximum number of steps?

### Time series plot of interval steps versus avg steps taken, averaged across all days

Compute the average steps taken in each interval across all days.


```r
intervalAverages <- ddply(cleanDat[-2], .(interval), numcolwise(mean))
```

Plot the time series of interval averages averaged across all days.

```r
library(lattice)
xyplot(steps ~ interval, data=intervalAverages, type="l")
```

![plot of chunk unnamed-chunk-5](figure/unnamed-chunk-5.png) 

The interval with the maximum number of steps, on average across all days, is:


```r
# Interval with max number of steps, on average across all days
intervalAverages[which.max(intervalAverages$steps), "interval"]
```

```
## [1] 835
```

## Imputing missing values

Assignment statement:
>Note that there are a number of days/intervals where there are missing values (coded as NA). The presence of missing days may introduce bias into some calculations or summaries of the data.

>Calculate and report the total number of missing values in the dataset (i.e. the total number of rows with NAs)

>Devise a strategy for filling in all of the missing values in the dataset. The strategy does not need to be sophisticated. For example, you could use the mean/median for that day, or the mean for that 5-minute interval, etc.

>Create a new dataset that is equal to the original dataset but with the missing data filled in.

>Make a histogram of the total number of steps taken each day and Calculate and report the mean and median total number of steps taken per day. Do these values differ from the estimates from the first part of the assignment? What is the impact of imputing missing data on the estimates of the total daily number of steps?

### Total number of missing values (number of rows with NAs)

Previously created a data frame named cleanDat with rows containing NAs removed. So the number of rows with NAs is the difference between the number of rows in the original data frame and the number of rows in cleanDat, ala

```r
nrow(dat) - nrow(cleanDat)
```

```
## [1] 2304
```

Another way, just to explore and confirm, is using which(), ala

```r
length(which(is.na(dat$steps)))
```

```
## [1] 2304
```
Which yields the same answer so must be correct!

### Strategy for filling in missing values

The strategy I chose is to replace the missing values on a per-day basis with the average value for that weekday. For example, the first day of the data set is Monday and it has missing values. So I compute the average number of steps for all the other Mondays and use that to replace the missing Monday data in the original data set.


```r
# Add a variable called day to the dailySteps data frame. Uses lubridate package.
library(lubridate)
```

```
## 
## Attaching package: 'lubridate'
## 
## The following object is masked from 'package:plyr':
## 
##     here
```

```r
dailySteps$day <- wday(dailySteps$date, label=TRUE)

# Get the days that have zero steps
zeroDays <- which(dailySteps$steps == 0)

# Subset to get only days with nonzero steps with which to compute averages
nonzeroDays <- dailySteps[dailySteps$steps !=0,]

# Compute the average steps for each weekday using days with nonzero data.
dayAverages <- ddply(nonzeroDays[-1], .(day), numcolwise(mean))
```

### Create new data set with imputed values for missing data


```r
# Create an new daily steps data frame without the NAs removed
newDailySteps <- ddply(dat[-3], .(date), numcolwise(sum))
newDailySteps$day <- wday(newDailySteps$date, label=TRUE)

# Substitute in the weekday average for days with NAs
for (i in 1:nrow(newDailySteps))
    if (is.na(newDailySteps[i, "steps"]))
        newDailySteps[i, "steps"] <- dayAverages[dayAverages$day==newDailySteps[i, "day"], "steps"]
```

### Histogram, mean and median of data with imputed missing values

Now, working with the data frame named newDailySteps that has had missing values replaced with imputed values, the histogram looks like this.


```r
# Plot histogram with 10 bins
hist(newDailySteps$steps, col="steelblue", breaks=10, xlab="Daily Steps", ylab="Number of Days", main="Histogram of Daily Steps with Imputed Data")
```

![plot of chunk unnamed-chunk-11](figure/unnamed-chunk-11.png) 

The  new histogram looks very similar to the original one, although it looks like the center bins are a little taller. Let's take a look at those bins side-by-side using the plotrix package.


```r
library(plotrix)
l <- list(dailySteps$steps, newDailySteps$steps)
multhist(l, xlab="Daily Steps", ylab="Frequency", col=c("blue", "green"))
legendText <- c("NAs removed", "Imputed Data")
legend("topright", legend=legendText, col=c("blue", "green"), pch=15)
```

![plot of chunk unnamed-chunk-12](figure/unnamed-chunk-12.png) 

Indeed, the three center bins have a significantly higher fequency of occurrence when using imputed data for the missing data.

The new mean and median are:


```r
mean(newDailySteps$steps)
```

```
## [1] 10821
```

```r
median(newDailySteps$steps)
```

```
## [1] 11015
```

In answer to the first question for this part of the assignment,
>Do these values differ from the estimates from the first part of the assignment?  

The answer is YES, they differ significantly.

In answer to the second question for this part of the assignment,
>What is the impact of imputing missing data on the estimates of the total daily number of steps?  

The impact is to raise the estimates significantly. This was the case because a large number of the imputed values fell into the bins for higher numbers of steps, causing the mean and median to increase.

## Are there differences in activity patterns between weekdays and weekends?

Assignment statement:  
>For this part the weekdays() function may be of some help here. Use the dataset with the filled-in missing values for this part.  
>1. Create a new factor variable in the dataset with two levels – “weekday” and “weekend” indicating whether a given date is a weekday or weekend day.  
>2. Make a panel plot containing a time series plot (i.e. type = "l") of the 5-minute interval (x-axis) and the average number of steps taken, averaged across all weekday days or weekend days (y-axis). The plot should look something like the following, which was creating using simulated data: (example charts)
