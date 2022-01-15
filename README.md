## Outlier-and-Anomaly-Detection

This project goes over ways to identify potential outliers/anomalies in a large dataset on domestic U.S. flights. One of the challenges with the data is its size - there are almost 590,000 observations making the computation taxing and lengthy. With the use of known methods and data exploration, I was able to narrow down the possible outliers and then confirm that these are in fact odd by using validation techniques. 


```{r setup, include=FALSE, echo=TRUE, cache=TRUE}
library(tidyverse)
library(dplyr)
library(kableExtra)
library(ggplot2)
library(maps)
library(usmap)
library(plotly)
library(viridis)
library(cowplot)
library(plotly)
library(hrbrthemes)
library(tidyr)
library(viridis)
library(ggpubr)
library(dbscan)
library(DescTools)
library(isotree)
```

## Data Exploration 

We begin with loading the data and exploring the patterns seen in the variables. First, we wish to understand what the variables are and infer whether we'll be needing them for the future analyses.

```{r}
flights <- read.csv("~/Documents/GitHub/DataAnalysis-Sasha-Ced/Flights1_2019_1.csv")
summary(flights)
```

Looks like there are a lot of variable roughly describing the same things - like those relating to location (airport IDs and city names), so we won't be needing both types. Furthermore, there are "corrected" versions of the arrival times - we will keep the original delays (the "new" arrival delay seems to have replaced negative values with zeroes) and arr_del15 - it may come in handy later. The section below further explands on the decisions made to keep/drop certain variables.

## Data Dictionary

```{r,echo=FALSE,message=FALSE,warning=FALSE,cache=TRUE}
results <- matrix(c("YEAR","year of flight","numeric","{2019}","no",
                    "DAY_OF_WEEK","day of week for the flight","numeric","[1, 7]","yes",
                    "FL_DATE","date of flight","character","[2019-01-01, 2019-01-31]","yes",
                    "ORIGIN_AIRPORT_ID","airport ID at the origin","numeric","[10,135, 16,218]","no",
                    "ORIGIN_AIRPORT_SEQ_ID","airport ID at the origin (at the sequential level)","numeric","[1,013,505, 1,621,802]","no",
                    "ORIGIN_CITY_MARKET_ID","identification number assigned to identify a city market at the origin","numeric","[30,070, 35,991]","no",
                    "ORIGIN_CITY_NAME","US city and state name at the origin","character","*340 unique cities","yes",
                    "DEST_AIRPORT_ID","airport ID at the destination","numeric","[10,135, 16,218]","no",
                    "DEST_AIRPORT_SEQ_ID","airport ID at the destination (at the sequential level)","numeric","[1,013,505, 1,621,802]","no",
                    "DEST_CITY_MARKET_ID","identification number assigned to identify a city market at the destination","numeric","[30,070, 35,991]","no",
                    "DEST_CITY_NAME","US city and state name at the destination","character","*340 unique cities","yes",
                    "DEST_STATE_ABR","abbreviated name of state at the destination","character","*52 unique names","yes",
                    "DEP_DELAY","delay of departure in minutes","numeric","[-47, 1,651]","yes",
                    "ARR_TIME","time of arrival","numeric","[1, 2,400]","no",
                    "ARR_DELAY","delay of arrival in minutes","numeric","[-85, 1,638]","yes",
                    "ARR_DELAY_NEW","adjusted delay of arrival in minutes","numeric","[0, 1,638]","yes",
                    "ARR_DEL15","unknown","numeric","{0, 1}","yes",
                    "X","unknown","uknown","none","no"),
                  nrow=18,byrow=TRUE)
colnames(results) <- c("Name","Definition","Data Type","Domain","Required")
rownames(results) <- NULL
knitr::opts_chunk$set(fig.pos = 'H')
kable(results,caption="Flights, or RITA (Reporting Carrier On-Time Performance), Data Dictionary",
      align="llclc",booktabs=TRUE) %>%
  kable_styling(latex_options = c("striped","scale_down"),font_size = 10) %>%
  row_spec(0,bold=TRUE) %>%
  column_spec(1,italic=TRUE,latex_valign = "m")  %>%
  column_spec(2,width="4cm",latex_valign = "m")  %>%
  column_spec(4,width="4cm",latex_valign = "m")
```

For the variables **origin_city_name** and **dest_city_name**, we can separate the state name after the city name into its own variable. Also, we will drop variables that we previously stated were considered to be unlikely to contribute to the outlier detection.

```{r}
flights$ORIGIN_STATE_ABR <- sapply(strsplit(flights$ORIGIN_CITY_NAME, ", "), "[", 2)
flights$ORIGIN_CITY_NAME <- sapply(strsplit(flights$ORIGIN_CITY_NAME, ", "), "[", 1)
flights$DEST_CITY_NAME <- sub('\\,.*', '', flights$DEST_CITY_NAME)
flights <- subset(flights,select=-c(YEAR,ORIGIN_AIRPORT_ID,DEST_AIRPORT_SEQ_ID,ARR_TIME,
                                    X,ORIGIN_AIRPORT_SEQ_ID,ORIGIN_CITY_MARKET_ID,
                                    DEST_AIRPORT_ID,DEST_CITY_MARKET_ID))
```

We can visualize how many flights occurred on any given day, of which there were 31 (the month of January in 2019), and see if there are any visible differences in the data. We see that the majority of the number of flights were above 18,000, with 5 observations below that threshold. With the boxplot, we see that there are three outliers (using Tukey's boxplot test). These occur on the 12th, 19th, and 26th of the month.

```{r,fig.align = 'center',out.width="100%",fig.cap = "Number of Floights Exploration",echo=FALSE}
par(mfrow=c(1,2))
flights$VALUE <- 1
flights$FL_DATE <- as.Date(flights$FL_DATE)
dateflight <- aggregate(flights$VALUE, by=list(flights$FL_DATE), sum)
colnames(dateflight) <- c("Date","FlightNum")
plot(dateflight$Date,dateflight$FlightNum,xlab="Date",ylab="Number of Flights")
boxplot(dateflight$FlightNum,col="lightblue2")
```

Some of the destination cities include two city names - presumably because the persons travelling are flying to an airport closest to their actual destination (perhaps there isn't an airport at the final destination). For those cases, we will only keep the name of the city that has an airport for simplicity. This may have some drawbacks - airports located on the edge of the state may cause some confusion (e.g., Dulles airport is located in Virginia, but people often arrive there when travelling to Washington, D.C.) or mislabeling the actual destination. We will carry on with the changes as we want to see the name of the city, rather than the location of the airport.

```{r}
flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Allentown/Bethlehem/Easton")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Allentown/Bethlehem/Easton")] <- "Allentown"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Arcata/Eureka")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Arcata/Eureka")] <- "McKinleyville"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Beaumont/Port Arthur")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Beaumont/Port Arthur")] <- "Beaumont"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Bend/Redmond")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Bend/Redmond")] <- "Redmond"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Bismarck/Mandan")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Bismarck/Mandan")] <- "Bismarck"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Bloomington/Normal")] <- 
flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Bloomington/Normal")] <- "Bloomington"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Bristol/Johnson City/Kingsport")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Bristol/Johnson City/Kingsport")] <- "Blountville"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Cedar Rapids/Iowa City")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Cedar Rapids/Iowa City")] <- "Cedar Rapids"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Champaign/Urbana")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Champaign/Urbana")] <- "Savoy"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Charleston/Dunbar")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Charleston/Dunbar")] <- "Charleston"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Clarksburg/Fairmont")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Clarksburg/Fairmont")] <- "Bridgeport"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="College Station/Bryan")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="College Station/Bryan")] <- "College Station"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="CONCORD")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="CONCORD")] <- "Concord"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Dallas/Fort Worth")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Dallas/Fort Worth")] <- "DFW"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Elmira/Corning")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Elmira/Corning")] <- "Horseheads"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Greensboro/High Point")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Greensboro/High Point")] <- "Greensboro"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Gulfport/Biloxi")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Gulfport/Biloxi")] <- "Gulfport"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Hancock/Houghton")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Hancock/Houghton")] <- "Calumet"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Hattiesburg/Laurel")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Hattiesburg/Laurel")] <- "Moselle"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Iron Mountain/Kingsfd")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Iron Mountain/Kingsfd")] <- "Kingsford"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Ithaca/Cortland")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Ithaca/Cortland")] <- "Ithaca"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Jackson/Vicksburg")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Jackson/Vicksburg")] <- "Jackson"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Jacksonville/Camp Lejeune")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Jacksonville/Camp Lejeune")] <- "Richlands"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Lawton/Fort Sill")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Lawton/Fort Sill")] <- "Lawton"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Manhattan/Ft. Riley")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Manhattan/Ft. Riley")] <- "Manhattan"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Midland/Odessa")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Midland/Odessa")] <- "Midland"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Mission/McAllen/Edinburg")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Mission/McAllen/Edinburg")] <- "McAllen"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Montrose/Delta")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Montrose/Delta")] <- "Montrose"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="New Bern/Morehead/Beaufort")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="New Bern/Morehead/Beaufort")] <- "New Bern"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Newburgh/Poughkeepsie")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Newburgh/Poughkeepsie")] <- "New Windsor"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Newport News/Williamsburg")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Newport News/Williamsburg")] <- "Newport News"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="North Bend/Coos Bay")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="North Bend/Coos Bay")] <- "North Bend"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Pasco/Kennewick/Richland")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Pasco/Kennewick/Richland")] <- "Richland"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Raleigh/Durham")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Raleigh/Durham")] <- "Morrisville"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Saginaw/Bay City/Midland")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Saginaw/Bay City/Midland")] <- "Freeland"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Sarasota/Bradenton")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Sarasota/Bradenton")] <- "Sarasota"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Scranton/Wilkes-Barre")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Scranton/Wilkes-Barre")] <- "Avoca"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="Sun Valley/Hailey/Ketchum")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="Sun Valley/Hailey/Ketchum")] <- "Hailey"

flights$ORIGIN_CITY_NAME[which(flights$ORIGIN_CITY_NAME=="West Palm Beach/Palm Beach")] <- flights$DEST_CITY_NAME[which(flights$DEST_CITY_NAME=="West Palm Beach/Palm Beach")] <- "West Palm Beach"
```

We can now study the number of flights on any particular day of the week.

```{r,message=FALSE,warning=FALSE,echo=FALSE}
par(mfrow=c(1,2))
dayflight <- aggregate(flights$VALUE, by=list(flights$DAY_OF_WEEK), sum)
colnames(dayflight) <- c("Day","FlightNum")
plot(dayflight$Day,dayflight$FlightNum,xlab="Day of the Week",ylab="Number of Flights")
boxplot(dayflight$FlightNum,col="lightblue2")
```

Assuming that 1 corresponds to a Monday (we checked that January 1, 2019 is a Tuesday and corresponds to a 2 in the data), we notice that the number of flights is  lower on day 6, which is a Saturday, than it is on the rest of the days of the week.

In the boxplot, we actually see that there are no outliers - so perhaps it is the case that less people fly to and from the cities in the dataset on a Saturday. This may or may not be true, but we will leave this matter alone for now and explore the rest of the data.

Next, we can take a look at how many flights departed from each city to notice if there are any blank spaces with very few odd flights or if they are spread across the country.

```{r}
originflight <- aggregate(flights$VALUE, by=list(flights$ORIGIN_STATE_ABR,flights$ORIGIN_CITY_NAME), sum)
colnames(originflight) <- c("state","city","FlightNum")

destflight <- aggregate(flights$VALUE, by=list(flights$DEST_STATE_ABR,flights$DEST_CITY_NAME), sum)
colnames(destflight) <- c("state","city","FlightNum")
```

```{r}
uscities <- read.csv("~/Documents/GitHub/DataAnalysis-Sasha-Ced/uscities.csv")
originlatlon <- merge(x=subset(uscities,select=c("city","state_name","lat","lng","state_id")),y=originflight,
      by.x=c("city","state_id"),by.y=c("city","state"))
```

```{r,fig.align = 'center',out.width="100%",fig.cap = "Number of Flights by City: Departures",message=FALSE,warning=FALSE,echo=FALSE}
originlatlon <- originlatlon[,c(5,4,1,2,3,6)]
originlatlon.t <- usmap_transform(originlatlon)

plot_usmap(fill = "white") +
  ggrepel::geom_label_repel(data = originlatlon.t,
             aes(x = lng.1, y = lat.1, label = city),
             size = 3, alpha = 0.8,
             label.r = unit(0.5, "lines"), label.size = 0.5,
             segment.color = "red", segment.size = 0.5,
             max.overlaps=0) +
  geom_point(data = originlatlon.t,
             aes(x = lng.1, y = lat.1, size = FlightNum),
             color = "purple", alpha = 0.5) +
  scale_size_continuous(range = c(1, 16),
                        label = scales::comma) +
  labs(size = "Number of Flights") +
  theme(legend.position = "right")
```

In using the maps to help us understand the data, we notice that the origin/destination include Puerto Rico (the actual island outline is not shown on either map) as well as parts of the USA. For the origin of flights, we see that the majority depart from the East Coast, as well as a lot of departures from CA, TX, and CO.

```{r}
destlatlon <- merge(x=subset(uscities,select=c("city","state_name","lat","lng","state_id")),y=destflight,
      by.x=c("city","state_id"),by.y=c("city","state"))
```

```{r,fig.align = 'center',out.width="100%",fig.cap = "Number of Flights by City: Arrivals",message=FALSE,warning=FALSE,echo=FALSE}
destlatlon <- destlatlon[,c(5,4,1,2,3,6)]
destlatlon.t <- usmap_transform(destlatlon)

plot_usmap(fill = "white") +
  ggrepel::geom_label_repel(data = destlatlon.t,
             aes(x = lng.1, y = lat.1, label = city),
             size = 3, alpha = 0.8,
             label.r = unit(0.5, "lines"), label.size = 0.5,
             segment.color = "red", segment.size = 0.5,
             max.overlaps=0) +
  geom_point(data = destlatlon.t,
             aes(x = lng.1, y = lat.1, size = FlightNum),
             color = "purple", alpha = 0.5) +
  scale_size_continuous(range = c(1, 16),
                        label = scales::comma) +
  labs(size = "Number of Flights") +
  theme(legend.position = "right")
```

We see a similar result for the flight destination cities. There isn't any one state where the number of flights is less than 8. That, combined with the boxplot we looked at earlier, suggests that there may not be an outlier/anomaly in terms of an origin/destination of a flight. We will have to study this further, but we will move on for now.

We can now take a look at the relationship between departure and arrival delays. We would expect this relationship to be roughly linear, since if a flight departs 10 minutes late, it is likely to arrive 10 minutes late as well. However, there are conditions under which this will not be true. E.g., good weather will allow for a shorter flight, while bad weather can delay it; sometimes planes have to wait on the tarmac for their spot to open up even after the plane landed; any other unexpected situations.

```{r,fig.align = 'center',out.width="100%",fig.cap = "Departure vs Arrival Delays",message=FALSE,warning=FALSE,echo=FALSE}
ggplot(data = flights, aes(x = DEP_DELAY, y = ARR_DELAY)) +
  labs(x = "Delay in Departure", y = "Delay in Arrival") +
  theme_classic() +
  geom_point()
```

While 18,022 observations are not included in this plot (these are missing observations and they make up about 3.1\% of the observations), it is still abundantly clear that the relationship between the delay in flight departure and arrival is practically linear. There are, however some interesting patterns worth mentioning. 

While majority of the data are concentrated near the left bottom corner (low values), there are quite a few observations exceeding these small delays, even going into over a day delay (1,651 minutes = 27 hours and 31 minutes). It is plausible that some flights did indeed get delayed for over a day in a few cases, especially given that it's January and a lot of the flights originated at/arrived to the east coast, where the weather conditions could be quite poor. 

Further notice that in our earlier analyses, we saw that 3rd quartile for departure delays was 5 minutes and for arrival delays, it was 7 minutes. Thus, it is suggestive that the long delays are potentially outliers that can be explained by some other factors and not anomalies (as we see a lot of these).

Another interesting observation is that the arrival delays sometimes exceed departure delays (e.g., 231 minutes vs. 89 minutes). As we mentioned earlier, there are explainable reasons for why that may happen, so these values are conceivable. Also note that a lot of the values for both variables are negative, suggesting that the flights departed/arrived earlier than the scheduled time.

We can further analyse the departure and arrival delays, as compared to the day of the week.

```{r,fig.align = 'center',out.width="100%",fig.cap = "Departure Delays by Day of the Week",message=FALSE,warning=FALSE,echo=FALSE}
ggplot(data = flights, aes(x = DAY_OF_WEEK, y = DEP_DELAY)) +
  labs(x = "Day of Week", y = "Delay in Departure") +
  theme_classic() +
  geom_point()
```

As we noted before, majority of departure delays are relatively short (with mean of about 10 minutes), but there are quite a few flights with delays of several hours. We see that for every day of the week there are flights with departure delays lasting over a day, but the most noticeable extreme delays fall on a Friday - there are three extreme delays that stand out the most. We will have to investigate that later.

```{r,fig.align = 'center',out.width="100%",fig.cap = "Departure Delays by Date",message=FALSE,warning=FALSE,echo=FALSE}
ggplot(data = flights, aes(x = FL_DATE, y = DEP_DELAY)) +
  labs(x = "Flight Date", y = "Delay in Departure") +
  theme_classic() +
  geom_point()
```

Similar to the finding for the days of the week, we see that there are a few extreme observations that fall on specific dates. Namely, there are two observations that really stand out - the extreme delay on the 5th and the 25th - these are relatively extreme to the rest of the data. While these are the most "extreme" relative to the data, it does not mean that the other numbers we are seeing are not extreme - delays of over a few hours are questionable regardless of the day of the week/time if the month. 

These results will be very similar for the delay in arrivals, so those are not presented again.

Another variable that's worth analyzing to better understand the long delays is **arr_del15**. We suspect that this is an indicator variable equaling 1 when a flight is delayed for over 15 minutes or 0 otherwise.

```{r,fig.align = 'center',out.width="100%",fig.cap = "15+ Minute Arrival Delays Exploration",message=FALSE,warning=FALSE,echo=FALSE}
par(mfrow=c(1,2))
barplot(table(flights$ARR_DEL15),xlab="15+ Minute Delay?",ylab="Number of Flights")
cdplot(as.factor(flights$ARR_DEL15)~flights$FL_DATE,
       xlab="Flight Date",ylab="15+ Minute Delay?")
```
We see that only a relatively small fraction of flights was delayed for over 15 minutes (only ~18\% of all flights, which matches with the data on arrival delays). We can take a look at how these delays are spread out throughout the month of January. Also above we see that there are not any obvious extreme discrepancies in the delay patterns - the fraction of flights that arrive over 15 minutes late hovers around the average for every day if the month.

```{r,fig.align = 'center',out.width="100%",fig.cap = "15+ Minute Arrival Selays vs Arrival Delays",message=FALSE,warning=FALSE,echo=FALSE}
ggdensity(flights, x = "ARR_DELAY",
   add = "mean", rug = TRUE,
   color = "ARR_DEL15", fill = "ARR_DEL15",
   xlab="Arrival Delay",ylab="Density",
   legend.title="15+ Minute Delay")
```

We can also see from the density plot above that there is a visible difference between the two groups for the variable **arr_del15**. The mean for the non-delayed group for the arrival delay is around - 10 minutes (i.e., early arrivals) and for the delayed group it's 68 minutes (i.e., over an hour delay). This may be beneficial in deciding what the outliers are - instead of using the entire dataset, we may be able to break the data down by this indicator and detect outliers that way.

```{r}
summary(flights$ARR_DELAY[which(flights$ARR_DEL15==1)])
```

We see that by only considering the data where the delays are over 15 minutes, the quantiles have shifted, potentially helping us identify the "true" outliers. This may be the case because the negative values skew the data, where the negative values are simply early arrivals. Arrival delays lasting more than 15 minutes are already outliers on their own - presumably that's why the variable was created. So, we will use this variable to identify which arrival delays spanned for over 15 minutes and see how these compare to the outliers found by other methods.

```{r,fig.align = 'center',out.width="100%",fig.cap = "Clusters for Full Data (Sample)",message=FALSE,warning=FALSE,echo=FALSE,cache=TRUE}
flights <- na.omit(flights)
rownames(flights) <- NULL
numvars <- flights[,c(6,7)]
z <- numvars[sample(nrow(numvars),30000),]
hdbscanResult <- hdbscan(z, minPts=10)
hullplot(z, hdbscanResult, xlab="Departure Delays", ylab="Arrival Delays",pch=20)
```

Above is the plot for the results of HDBSCAN for the entire dataset. The black points are the outliers (as identified by the method). Note that here we only used a sample from the data due to the dataset dimensions - there are far too many observations for the package to run, so we will need to come up with a method to sample over the data and observe what's identified as an "outlier".

```{r,fig.align = 'center',out.width="100%",fig.cap = "HDBSCAN: Clusters for Full Data",message=FALSE,warning=FALSE,echo=FALSE,cache=TRUE}
n <- 100
z <- indz <- vector(mode="list", length=n)
hdbscanResult <- vector(mode="list", length=n)
for (i in 1:n){
  z[[i]] <- numvars[sample(nrow(numvars),20000),]
  hdbscanResult[[i]] <- which(hdbscan(z[[i]],minPts=10)$cluster==0)
  indz[[i]] <- as.numeric(rownames(z[[i]][hdbscanResult[[i]],]))
}

ind <- unlist(indz)

colors <- c("Inliers" = "blue", "Outliers" = "red")
ggplot() +
  labs(x = "Day of Week", y = "Delay in Departure") +
  theme_classic() +
  geom_point(data=flights[unique(ind[duplicated(ind)]),],
             aes(x=DEP_DELAY,y=ARR_DELAY,color="red"),
             alpha=0.5,shape=20) +
  geom_point(data=flights[-unique(ind[duplicated(ind)]),],
             aes(x=DEP_DELAY,y=ARR_DELAY,color="blue"),
             alpha=0.5,shape=20) +
  theme(legend.position = "top", legend.direction = "horizontal") +
  scale_color_identity(name = "Observation Type",
                          breaks = c("blue", "red"),
                          labels = c("Inliers", "Outliers"),
                          guide = "legend") + 
  coord_cartesian(xlim = c(0, 1750), ylim = c(0, 1750))
```

Rather than plotting all the different clusters the method created, we focus on only outliers vs. non-outliers. It is evident that there are quite a few observations identified as outliers (all the red points), but these are actually a small fraction of the dataset (relative to how it looks in the plot above) - (or 24.7\%) of the observations are outliers. We see that the inliers are centered around the small positive and negative values, with most of the outliers scattered around larger delay times. 

We can use a distance based method to group those observations that are similar (as we did before, but only now we'll focus on creating one group). A good start here would be mahalanobis - we will look for the distances between observations, select a reasonable cut off distance, and see how this compares to the previous results.

```{r}
mdist <- mahalanobis(numvars, as.numeric(colMeans(numvars)), cov(numvars))
plot(1:length(mdist),mdist,pch=20,xlab="Observation Number",ylab="Mahalanobis Distance Between the Points")
```

It seems that the majority of the distances between points and the calculated data center are quite small - mostly below ~200. We can begin by selecting a cut-off distance equivalent to the 99th percentile. In this case, this is 22.7 and there are 5,660 observations that are greater than the aforementioned value. We can plot these observations as outliers (in red) vs the rest of the data or inliers (in blue).

```{r,fig.align = 'center',out.width="100%",fig.cap = "Mahalanobis Distance: 99th Percentile",message=FALSE,warning=FALSE,echo=FALSE,cache=TRUE}
ggplot() +
  labs(x = "Day of Week", y = "Delay in Departure") +
  theme_classic() +
  geom_point(data=numvars[-which(mdist>as.numeric(quantile(mdist,0.99))),],
             aes(x=DEP_DELAY,y=ARR_DELAY,color="blue"),
             alpha=0.5,shape=20) +
  geom_point(data=numvars[which(mdist>as.numeric(quantile(mdist,0.99))),],
             aes(x=DEP_DELAY,y=ARR_DELAY,color="red"),
             alpha=0.5,shape=20) +
  theme(legend.position = "top", legend.direction = "horizontal") +
  scale_color_identity(name = "Observation Type",
                          breaks = c("blue", "red"),
                          labels = c("Inliers", "Outliers"),
                          guide = "legend") + 
  coord_cartesian(xlim = c(0, 1750), ylim = c(0, 1750))
```

This is quite similar to what we found previously (and what we expected to see) - those observations that have shorter delay times are considered inliers, while those exceeding a pre-specified threshold are outliers (or, potentialy, anomalies). It is also clear that those obsrvations where the delays in departures vs arrivals are not consistent (e.g., 3 vs 73 minutes) are thought of as outliers. 

Next, we can try to increase this threshold and see how the pattern changes for the inliers. We now present a plot for the delay data where 99.9% of the data are inliers (based on the calculated distances) and so 566 observations are potential outliers.

```{r,fig.align = 'center',out.width="100%",fig.cap = "Mahalanobis Distance: 99.9th Percentile",message=FALSE,warning=FALSE,echo=FALSE,cache=TRUE}
ggplot() +
  labs(x = "Day of Week", y = "Delay in Departure") +
  theme_classic() +
  geom_point(data=numvars[-which(mdist>as.numeric(quantile(mdist,0.999))),],
             aes(x=DEP_DELAY,y=ARR_DELAY,color="blue"),
             alpha=0.5,shape=20) +
  geom_point(data=numvars[which(mdist>as.numeric(quantile(mdist,0.999))),],
             aes(x=DEP_DELAY,y=ARR_DELAY,color="red"),
             alpha=0.5,shape=20) +
  theme(legend.position = "top", legend.direction = "horizontal") +
  scale_color_identity(name = "Observation Type",
                          breaks = c("blue", "red"),
                          labels = c("Inliers", "Outliers"),
                          guide = "legend") + 
  coord_cartesian(xlim = c(0, 1750), ylim = c(0, 1750))
```

It becomes evident that whatever observations are close to the calculated center (9 minutes for departure and 4 minutes for arrival delays) will be regarded as the inliers. We conclude that it makes more sense to use the 99th percentile as a frame of reference.

Next, we can use the cosine similarity distance and see how this compares to previous results. We do so by calculating the similarity between the data points and the data center - simply the means for each departure and arrival delays. We again plot the distances and see that the majority are concentrated around zero. We will again choose the 99th percentile (distance of 6.7) as the distance cutoff and treat all values above this as outliers. There are 5,660 of these.

```{r}
p <- as.matrix(numvars)
q <- as.numeric(colMeans(numvars))
d <- rowSums(p*q)/(norm(p)*(q[1]^2+q[2]^2))
plot(1:length(d),d,pch=20,xlab="Observation Number",ylab="Cosine Similarity Distance Between the Points")
```

```{r,fig.align = 'center',out.width="100%",fig.cap = "Cosine Similarity Distance: 99th Percentile",message=FALSE,warning=FALSE,echo=FALSE,cache=TRUE}
ggplot() +
  labs(x = "Day of Week", y = "Delay in Departure") +
  theme_classic() +
  geom_point(data=numvars[-which(d>as.numeric(quantile(d,0.99))),],
             aes(x=DEP_DELAY,y=ARR_DELAY,color="blue"),
             alpha=0.5,shape=20) +
  geom_point(data=numvars[which(d>as.numeric(quantile(d,0.99))),],
             aes(x=DEP_DELAY,y=ARR_DELAY,color="red"),
             alpha=0.5,shape=20) +
  theme(legend.position = "top", legend.direction = "horizontal") +
  scale_color_identity(name = "Observation Type",
                          breaks = c("blue", "red"),
                          labels = c("Inliers", "Outliers"),
                          guide = "legend") + 
  coord_cartesian(xlim = c(0, 1750), ylim = c(0, 1750))
```

Again we see a similar pattern to that found when using the Mahalanobis distance. Note that out of 5,660 identified outliers by both methods, 4,114 (or 72.7\%) overlap - this gives us some confidence in there observations being true outliers. We will compare the results for all methods later and make conclusions about anamolous data points.

```{r}
length(intersect(which(d>as.numeric(quantile(d,0.99))),which(mdist>as.numeric(quantile(mdist,0.99)))))
```

Next, we can move onto density based methods, starting out with Isolation Forest Method (IsoForest). This technique tries to isolate anomalous points by randomly selecting an attribute and a split value between that attribute’s min/max values, continuing until every point is alone in its component. Below we see that in using this method there are quite a few points in the scatterplot where their score lies far from ~0, which is the dense region. Similarly, in the density plot we note the long tail, likely containing outlying observations. Results here are shown for one tree, build in a specific way, but we would like to observe the results of using many trees to validate the presence of outliers.

```{r}
iso <- isolation.forest(numvars, ntrees = 100, nthreads=1)
iso.d <- predict(iso, numvars)
par(mfrow=c(1,2))
plot(1:length(iso.d),iso.d,pch=20,xlab="Observation Number",ylab="IsoForest Score")
plot(density(iso.d),ylab="IsoForest Density",main="",xlab="")
```

We can proceed with creating 2,220 trees with various parameters, identifying the outliers for each, and then comparing these to each other. The result is the outliers that repeatedly show up in each tree.

```{r}
pars <- list("sample_size"=c(seq(256,565760,2560),565963),
             "ntrees"=seq(10,100,10))
iso.den <- matrix(rep(list(vector(mode="list")),length(pars[[1]])*length(pars[[2]])),
                  nrow=length(pars[[1]]),ncol=length(pars[[2]])) 
for (i in 1:length(pars[[1]])){
  for (j in 1:length(pars[[2]])){
    iso.den[[i,j]] <- which(predict(isolation.forest(numvars,sample_size=pars[[1]][i],
                        ntrees=pars[[2]][j],ndim=1,nthreads=1),numvars)>
                        quantile(predict(isolation.forest(numvars,sample_size=pars[[1]][i],
                        ntrees=pars[[2]][j],ndim=1,nthreads=1),numvars),0.99))
  }
}
isoF <- unique(unlist(iso.den)[duplicated(unlist(iso.den))])
```

Below is the plot with the outlying and inlying observations. Once again, this is consistent with what we saw for the distance-based methods.

```{r,fig.align = 'center',out.width="100%",fig.cap = "IsoForest: 99th Percentile",message=FALSE,warning=FALSE,echo=FALSE,cache=TRUE}
ggplot() +
  labs(x = "Day of Week", y = "Delay in Departure") +
  theme_classic() +
  geom_point(data=numvars[isoF,],
             aes(x=DEP_DELAY,y=ARR_DELAY,color="red"),
             alpha=0.5,shape=20) +
  geom_point(data=numvars[-isoF,],
             aes(x=DEP_DELAY,y=ARR_DELAY,color="blue"),
             alpha=0.5,shape=20) +
  theme(legend.position = "top", legend.direction = "horizontal") +
  scale_color_identity(name = "Observation Type",
                          breaks = c("blue", "red"),
                          labels = c("Inliers", "Outliers"),
                          guide = "legend") + 
  coord_cartesian(xlim = c(0, 1750), ylim = c(0, 1750))
```

We can proceed with LOF (local oitlier factor). This methods works by calculating the deviations from an observation to its nearest neighbors. An observation is then considered an outlier/anomaly if this deviation is large.

```{r}
methodLOF <- function(number){
  s <- sample(nrow(numvars),number)
  numvars <- numvars[s,]
  k <- dim(numvars)[1]
  kdist <- matrix(nrow=k,ncol=k)
  for (i in 1:k){
    for (j in 1:k){
      kdist[i,j] <- as.numeric(abs(numvars[i,1]-numvars[j,1])+abs(numvars[i,2]-numvars[j,2]))
    }
  }
ind <- t(apply(kdist,1,sort))
  kind <- matrix(nrow=k,ncol=2)
  for (i in 1:k){
    kind[i,1] <- sum(ind[i,which(ind[i,]<=10)])
    kind[i,2] <- length(which(ind[i,]<=10))
  }

  lrd <- c()
  for (i in 1:k){
    lrd[i] <- (1/kind[i,1])/kind[i,2]
  }
  lrd[which(lrd==Inf)] <- max(lrd[-which(lrd==max(lrd))])
  
  lof <- c()
  for (i in 1:k){
    lof[i] <- sum(lrd[which(ind[i,]<=10)])/(kind[i,2]*lrd[i])
  }
  s[which(lof>=quantile(sort(lof,decreasing=TRUE),0.99))]
}
lofs <- replicate(2000,methodLOF(500))
lofs <- unlist(lofs)[duplicated(unlist(lofs))]
```

Below is the plot of inliers vs outliers.

```{r,fig.align = 'center',out.width="100%",fig.cap = "LOF: 99th Percentile",message=FALSE,warning=FALSE,echo=FALSE,cache=TRUE}
ggplot() +
  labs(x = "Day of Week", y = "Delay in Departure") +
  theme_classic() +
  geom_point(data=numvars[-as.numeric(lofs),],
             aes(x=DEP_DELAY,y=ARR_DELAY,color="blue"),
             alpha=0.5,shape=20) +
  geom_point(data=numvars[as.numeric(lofs),],
             aes(x=DEP_DELAY,y=ARR_DELAY,color="red"),
             alpha=0.5,shape=20) +
  theme(legend.position = "top", legend.direction = "horizontal") +
  scale_color_identity(name = "Observation Type",
                          breaks = c("blue", "red"),
                          labels = c("Inliers", "Outliers"),
                          guide = "legend") + 
  coord_cartesian(xlim = c(0, 1750), ylim = c(0, 1750))
```

We can compare the results of all methods. We won't display each outlier as there are too many here; instead, we will display how many outliers each method identified.

```{r,echo=FALSE,message=FALSE,warning=FALSE,cache=TRUE}
d.15 <- which(flights$ARR_DEL15==1)
d.hdbscan <- unique(ind[duplicated(ind)])
d.mah <- which(mdist>as.numeric(quantile(mdist,0.99)))
d.cos <- which(d>as.numeric(quantile(d,0.99)))
outs <- c(length(d.15),length(d.hdbsca),length(d.mah),
          length(d.cos),length(isoF),length(lofs))

results <- matrix(c("Number of Outliers",outs),
                  nrow=1,byrow=TRUE)
colnames(results) <- c("","15+ Minute Delay (Data)","HDBSCAN","Mahalanobis Distance",
                       "Cosine Similarity Distance","Isolation Forest","LOF")
rownames(results) <- NULL
knitr::opts_chunk$set(fig.pos = 'H')
kable(results,caption="Outliers for Each Method",
      align="lcccccc",booktabs=TRUE) %>%
  kable_styling(latex_options = c("scale_down"),font_size = 10) %>%
  row_spec(0,bold=TRUE)
```

Next, we will find what outliers overlap for each method and plot these against the inliers. This set will be our final set of outliers. As we suspected, this consists of the arrival and departures delays that last (on average) over half an hour.

```{r,fig.align = 'center',out.width="100%",fig.cap = "Outliers for Flight's Data",message=FALSE,warning=FALSE,echo=FALSE,cache=TRUE}
out <- intersect(d.15,d.hdbscan,d.mah,d.cos,isoF,lofs)
ggplot() +
  labs(x = "Day of Week", y = "Delay in Departure") +
  theme_classic() +
  geom_point(data=numvars[-as.numeric(out),],
             aes(x=DEP_DELAY,y=ARR_DELAY,color="blue"),
             alpha=0.5,shape=20) +
  geom_point(data=numvars[as.numeric(out),],
             aes(x=DEP_DELAY,y=ARR_DELAY,color="red"),
             alpha=0.5,shape=20) +
  theme(legend.position = "top", legend.direction = "horizontal") +
  scale_color_identity(name = "Observation Type",
                          breaks = c("blue", "red"),
                          labels = c("Inliers", "Outliers"),
                          guide = "legend") + 
  coord_cartesian(xlim = c(0, 1750), ylim = c(0, 1750))
```
