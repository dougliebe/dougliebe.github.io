#### This project was supposed to investigate the performance of "stars and scrubs" lines and normal NHL lines: where all three players are of simialr skill. The common conception is that a star will raise the skill of the line greater than a line of equal average skill (but without a star). For example:

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
#### Fpts60 = (Goals/60 *3)+(Primary Assists/60 *2)+(iCorsi/60 *0.50*0.50)


