### This project was supposed to investigate the performance of "stars and scrubs" lines and normal NHL lines: where all three players are of simialr skill. The common conception is that a star will raise the skill of the line greater than a line of equal average skill (but without a star). For example:

Player X Skill - 9
Players Y & Z Skill - 3
__Average skill of line XYZ = 5__

Players A, B, C Skill - 5
__Average skill of line ABC = 5__

Most would assume that XYZ skill will be >> 5 due to the effect of having player X (a star)

```markdown
# the events log
temp = load("source-data\\nhlscrapr-20162017.RData") ## If you used option 1 (1 season)
all = get(temp)

# and the player roster
temp = load("source-data\\nhlscrapr-core.RData")
roster = get(temp)
```
