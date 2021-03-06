#### This project was supposed to investigate the performance of "stars and scrubs" lines and normal NHL lines: where all three players are of similar skill. The common conception is that a star will raise the skill of the line greater than a line of equal average skill (but without a star). For example:

Player | Skill
-------|-------
Player X | 9
Players Y & Z | 3
__Average skill of line XYZ__ | __5__

Player | Skill
-------|-------
Players A, B, C | 5
__Average skill of line ABC__ | __5__

Most would assume that XYZ skill will be >> 5 due to the effect of having player X (a star)

###### First, I loaded my play-by-play data for the previous season
```markdown

# the events log
temp = load("source-data\\nhlscrapr-20162017.RData") ## If you used option 1 (1 season)
all = get(temp)

# and the player roster
temp = load("source-data\\nhlscrapr-core.RData")
roster = get(temp)
```

##### Next, I broke the play-by-play data down by fantasy points
###### I used DK scoring
etype | Points
------|------
Goal | 3
Assist | 2
Shot | 0.5

```markdown
ev_all2$homepts <- apply(ev_all2[,c('etype','ev.team','hometeam','ev.player.2')],1,
                         function(x) { 
                           if (x[1]=="GOAL" & x[2]==x[3]) {if (as.numeric(x[4]) >1){5.5} else {3.5}
                           } else if ((x[1]=="SHOT" | x[1]=="BLOCK")&(x[2]==x[3])){0.5} else {0}
                           
                         } 
)
ev_all2$awaypts <- apply(ev_all2[,c('etype','ev.team','awayteam','ev.player.2')],1,
                         function(x) { 
                           if (x[1]=="GOAL" & x[2]==x[3]) {if (as.numeric(x[4]) >1){5.5} else {3.5}
                           } else if ((x[1]=="SHOT" | x[1]=="BLOCK")&(x[2]==x[3])){0.5} else {0}
                           
                         }
)
# Determining the time between each event
ev_all$Time <- ev_all$seconds-shift(ev_all$seconds,1)# For every set of events in which the same players were on the ice, point totals were combined
# Removing all times when there was a man advantage
ev_results <- subset(ev_all, ev_all$seconds!=0.0 & ev_all$Time > 0 &
                       ev_all$home.skaters == ev_all$away.skaters & ev_all$ev.team!="" & ev_all$ev.team!="HAN" & ev_all$ev.team!="GOA")
ev_results <- ev_results[,c(2,15,16,17,9,10,11,49,50,51)]

```

###### Now I have data for all shifts at EV and who was out there.
I have to aggregate point totals for when the same players remained out together. I am intereested in line combos so even one player changing will end the shift.
```markdown
# to get data for home and away team separately
aggh <- aggregate(list(ev_results$Time,ev_results$homepts),by = list(ev_results$gcode, ev_results$h1,ev_results$h2,ev_results$h3), sum)
names(aggh) <- c('game','h1','h2','h3','Time',"Home Pts")
aggh <- subset(aggh, aggh$Time>30)
aggh$pps <- aggh$`Home Pts`/aggh$Time
aggh$Home <- 1

agga <- aggregate(list(ev_results$Time,ev_results$awaypts),by = list(ev_results$gcode, ev_results$a1,ev_results$a2,ev_results$a3), sum)
names(agga) <- c('game','h1','h2','h3','Time',"Home Pts")
agga <- subset(agga, agga$Time>30)
agga$pps <- agga$`Home Pts`/agga$Time
agga$Home <- 0

# put them together with a variable for Home/Away
agglines <- rbind(aggh, agga)
```
At this point I matched player index numbers with data from Puckalytics.com for the 2015-2016 season
To calculate Fpts60 for each player I used the following formula:
#### Fpts60 = (Goals/60 *3)+(Primary Assists/60 *2)+(iCorsi/60 *0.50 *0.50)
I like to use iCorsi instead of shots because of its higher year-to-year correlations
Speaking of correlations... I adjusted the previous formula heavily to take into account the loss in predictability assocaited with goals, assists and corsi. 
Goals -> Goals (year+1) = ~32%
P1 Assists -> P1 Assists (year+1) = ~35%
iCorsi (year+1) = ~75%
#### Fpts60 = (Goals/60 *3) *0.32+(Primary Assists/60 *2) *0.35+(iCorsi/60 *0.50 *0.50) *0.75

Now for each shift we have: __who was out there, how many points they scored, how long they were out there, and what each player's fantasy production is__

I need to figure out who the best player on the line is and how much better he is than the worst player. I choose these two variables as proxies for determining stars and determining skill gap, respectively.

```markdown
## Make a comparison to lines with same average as crosby's, but with no superstar
agglines$max <- apply(agglines[9:11],1,max)
agglines$min <- apply(agglines[9:11],1,min)
# I turned points per second into points per hour, just to get rounder numbers
agglines$pps <- agglines$pps*3600
agglines$dif <- agglines$max - agglines$min
agglines$avg <- (agglines$h1fpts60+agglines$h2fpts60+agglines$h3fpts60)/3.0
# I removed shifts with huge scores really fast because I think they are just noise that skews the data
no_outliers <- subset(agglines, agglines$pps<150)
no_outliers$stars_and_scrubs <- ifelse(no_outliers$max > 4.8 & no_outliers$dif > 1.7,1,0)
no_outliers$stars_and_scrubs <- ifelse(no_outliers$dif < 1.7,"Low Skill Gap",ifelse(no_outliers$max > 4.8 & no_outliers$dif > 1.7,"Star with Lower Skilled",0))
```
Notice at the end of this snippet that I determined my 'stars and scrubs' lines based two criteria:
- Having one player with a max production over 4.8 Fpts/60
- Having a difference between best and worst player of at least 1.7 Fpts/60

Both of these values were chosen because they are the 3rd quartile of their respective categories

The last thing I did to tailor the data was remove lines with line averages that were outside the normal range (there were hardly any lines that had a star AS WELL AS a total line average Fpts60 below 3.8)

```markdown
o1 <- subset(no_outliers, no_outliers$stars_and_scrubs == "Low Skill Gap" & no_outliers$avg > 2.8 & no_outliers$avg <4.8)
o2 <- subset(no_outliers, no_outliers$stars_and_scrubs == "Star with Lower Skilled" & no_outliers$avg > 3.8 & no_outliers$avg < 4.8)
o3 <- rbind(o1,o2)
```


![alt text](https://user-images.githubusercontent.com/29124840/27358534-26d39a42-55e5-11e7-907e-d535e81dd3b8.png)  
It appears that stars cannot even lift the production of two lower level players to that of three players of even fantasy production. When a great player plays with only slightly below average players, their total line fantasy production sees drops of 10% or greater on avereage.
