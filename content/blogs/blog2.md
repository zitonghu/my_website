---
title: "Magna"
date: '2017-10-31T22:26:09-05:00'
description: Lorem Etiam Nullam
draft: no
image: pic09.jpg
keywords: ''
slug: magna
categories:
- ''
- ''
---


```{r, get_tfl_data, cache=TRUE}
url <- "https://data.london.gov.uk/download/number-bicycle-hires/ac29363e-e0cb-47cc-a97a-e216d900a6b0/tfl-daily-cycle-hires.xlsx"

# Download TFL data to temporary file
httr::GET(url, write_disk(bike.temp <- tempfile(fileext = ".xlsx")))

# Use read_excel to read it as dataframe
bike0 <- read_excel(bike.temp,
                   sheet = "Data",
                   range = cell_cols("A:B"))

# change dates to get year, month, and week
bike <- bike0 %>% 
  clean_names() %>% 
  rename (bikes_hired = number_of_bicycle_hires) %>% 
  mutate (year = year(day),
          month = lubridate::month(day, label = TRUE),
          week = isoweek(day))
```


## Monthly Changes in in TfL bike rentals

```{r tfl_absolute_monthly_change, echo=FALSE, out.width="100%"}

bike1<- bike %>% 
  filter (year > 2016, year < 2023) %>% # to obtain years 2017-2022
  group_by(year, month) %>% # group by year, then month
  mutate(month=match(month,month.abb)) %>% # converting month into numbers (to be able to draw)
  summarize(mon_year_avg = mean(bikes_hired)) # find mean of each year of each month

bike2 <- bike %>% 
  filter(year>2015, year <2020) %>% # second line, draw average 2016-2019
  group_by(month) %>%  #
  mutate(month = match(month,month.abb)) %>%  # converting month into numbers (to be able to draw)
  summarize(mean_months = mean(bikes_hired)) # fund mean of each month throughout the years

#mon_year_avg>mean_months #green
#mean_months>mon_year_avg #red
bike3 <- merge(x=bike1,y=bike2,by ='month') #
ggplot(bike3, aes(x=month))+ #
  geom_line(aes(y=mon_year_avg))+ #plot yearly month average
  geom_line(aes(y=mean_months),colour='blue') + #mean of months
  geom_ribbon(data=bike3,aes(ymax=pmin(mean_months,mon_year_avg),ymin=mon_year_avg),fill="green",alpha=0.5) + #green ribbon when average is more then 2016-2019
  geom_ribbon(data=bike3,aes(ymax=pmin(mean_months,mon_year_avg), ymin=mean_months),fill="red",alpha=0.5)+
  #geom_ribbon(data = bike6, aes(ymax = pmax(per_change, 0), ymin = 0), fill = "green", alpha = 0.5)+
  #geom_ribbon(data = bike6, aes(ymax = 0, ymin = pmin(per_change, 0)), fill = "red", alpha = 0.5)
  facet_wrap(vars(year))+ 
  ylim(10000,45000)+ #limit on y axis
  labs(x="",y="Bike rentals",title="Monthly changes in TfL bike rentals",subtitle="Change from monthly average shown in blue and calculated between 2016-2019") +
  theme_minimal()

```

## Weekly Changes in TfL bike rentals
The second one looks at percentage changes from the expected level of
weekly rentals. The two grey shaded rectangles correspond to Q2 (weeks
14-26) and Q4 (weeks 40-52).

```{r tfl_percent_change, echo=FALSE, out.width="100%"}
#process data to get the percentage difference between expected and actual bike rented per week
bike4<- bike %>% filter (year > 2016, year < 2023) %>% group_by(year, week) %>%   summarize(week_year_avg = mean(bikes_hired))

bike5 <- bike %>% filter(year>2015, year <2020) %>% group_by(week)%>% summarize(mean_weeks = mean(bikes_hired), base_line = 0)

bike6 <- merge(x=bike4,y=bike5,by ='week')
bike6<- bike6 %>% filter() %>% mutate(per_change = (week_year_avg - mean_weeks)/mean_weeks, if_exceed = ifelse(per_change >=0, TRUE, FALSE))

#plot expected and actual bike rental per year 
ggplot(bike6, aes(x=week))+geom_line(aes(group = 1, y=per_change),colour='black')+geom_line(aes(y = base_line )) +facet_wrap(vars(year))+xlim(0, 53)+
#setting different colors for above/below 0
geom_ribbon(data = bike6, aes(ymax = pmax(per_change, 0), ymin = 0), fill = "green", alpha = 0.5)+
geom_ribbon(data = bike6, aes(ymax = 0, ymin = pmin(per_change, 0)), fill = "red", alpha = 0.5)+ scale_y_continuous(labels = scales::percent)+
#add geom_rug
geom_rug(mapping = aes(color = if_exceed), sides = "b")+
#add titles
labs(title = "Weekly changes in Tfl bike rental",
              subtitle = "% change from weekly averages calculated between 2016-2019",
              caption = "Data source: TfL, London Data Store")

```