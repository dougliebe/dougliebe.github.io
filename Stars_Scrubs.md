```markdown
#the events log
temp = load("source-data\\nhlscrapr-20162017.RData") ## If you used option 1 (1 season)
all = get(temp)

#and the player roster
temp = load("source-data\\nhlscrapr-core.RData")
roster = get(temp)
```
