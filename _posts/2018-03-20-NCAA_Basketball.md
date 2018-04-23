---
layout: post
title: Mastering March Madness
---

> "I don't care what people think. people are stupid." - Charles Barkley

### Predicting NCAA D1 Men's College Basketball Games

Based on statistics that I've completely made up in my head at this very moment, 46% of today's data scientists were first drawn to the industry by the prospect of mastering their March Madness Bracket Pool. While that number might in reality, be higher or lower, the point is that trying to predict winning teams during the annual March Madness tournament is not, by any means, a new endeavor. 

The annual tournament, which began in 1939, is a single-elimination tournament, with the winners of each stage advancing to the next round, until a single championship team emerges as the undefeated winner. Since 1985, the tournament has consisted of 64 teams, yielding 6 distinct stages in the tournament (Round of 64, Round of 32, Sweet 16, Elite 8, Final 4, and Championship).

![png](/images/NCAA_Basketball/bracket.png)

Less well-documented is the history of "Bracket Pools" based on March Madness results. "Bracket Pools" are ubiquitous today, a fan-based game where entrants choose the teams they expect to advance at each stage, with points awarded to correct picks. Generally, a flat-fee is assigned to every entry in a betting pool, and the best performing brackets are awarded a cut of the total betting pool at the end.

Every team at the beginning of the tournament is assigned a ranking, or "seed", with 1 being considered the best, and 16 being considered the worst. And of course, every year, a few lower-seeded teams manage to beat the favored higher-seeded teams. While these upsets could be considered outliers attributable to random chance, diehard fans will tell you that there are true, underlying reasons for these upsets. Being able to predict these upsets, or more simply, all of the winnings teams, correctly, there is a tremendous opportunity for financial gain. (Not to mention the impending Supreme Court decision on reversing sports gambling restrictions in [New Jersey](https://nypost.com/2018/04/22/supreme-court-to-decide-on-nj-sports-gambling-this-week/) and beyond)

In the proceeding sections, I set out to build and train a model to identify the underlying advantages in NCAA D1 Men's Basketball team matchups, and try to correctly pick the winning team as often as Las Vegas sportsbooks.

If you like getting into the weeds, the coding information for this project can be found on my GitHub: [https://github.com/Tucker-Allen/NCAA_D1_Mens_Basketball_Scores](https://github.com/Tucker-Allen/NCAA_D1_Mens_Basketball_Scores)

You can also find slides from my presentation [here](https://docs.google.com/presentation/d/1ULHUyg_bc0HGAPOLS57ffJuN20BpV7kgp1hgK4PHUf8/edit?usp=sharing).

---

## Data Scraping:

The team over at [sports-reference](https://www.sports-reference.com/cbb/) have done a tremendous job compiling box scores, on an individual player basis, for every D1 basketball game that has taken place since the 2010-2011 season. To compile the box score data available, I built a script, hosted on AWS, that scraped over 90,000 html pages. Page data was parsed through BeautifulSoup, manipulated into a Pandas DataFrame, and periodically stored into csv checkpoint files. In totality, I managed to accumulate over 870,000 individual player game boxscores, spanning from the 2011 to 2018 seasons.

---

## Strategy:

It's tempting to oversimplify that "well, the team that scores the most points wins, so the team that traditionally scores a lot of points will be best". But a short review of the history, or as any good fan will tell you, there's a much more nuanced interaction between offense and defense that defines the outcome of a game. For a brief example, see Gonzaga in the 2011 season:

![png](/images/NCAA_Basketball/gonzaga.png)

Gonzaga won most, but not all of the games where it scored more points than typical. Conversely, Gonzaga also managed to win a lot of games at the "troughs", where they scored less points than typical. Teams can manage to pile up wins without necessarily putting up a lot of points.

So I set out to try and best inform my model of four larger charactersistics of any given matchup between Team A and Team B:
 1. Team A/B's long-run and short-run winningness/box-scores, independent of opponents
 2. Team A/B's long-run and short-run "holding-to"ness, i.e. defensive impact
 3. Team A/B's "Strength of Schedule", a weighting factor to #1 and #2.
 4. Team A/B's "SRS", a Simple-Rating-System score maintained by sports-reference

 Long-run metrics aim to characterize how good a team is, historically. I defined this as a 30-day rolling mean, so as to capture approximately a season's worth of games. However, since we want to capture short-term variance, due to perhaps player-injuries, or the addition of new talent, I defined short-run metrics with a few short-sighted exponentially weighted means. This is similarly applied to the alternate team's "defensive" version of those metrics. All of those previous metrics occur against teams of varying quality, and so I included Strength-of-Schedule (SOS) to inform the model how to appropriately weigh those metrics. Lastly, I included an overall score metric maintained by sports-reference for each individual team.

![png](/images/NCAA_Basketball/sos.png)

The Strength-of-Schedule histogram appears to have two peaks. This is not surprising, since some conferences are considered much more difficult than others, so there may be "buckets" of teams centered around different values

![png](/images/NCAA_Basketball/srs.png)

The SRS, determined by sports-reference, is an estimate of how many points a given team would be expected to score over an "average team". Seeing this histogram centered near 0 is reassuring, since the total summation of these estimates should zero out.

## Vegas Accuracy: 74.3%

Since 2003, Vegas Sportsbooks have correctly picked the winning team at a 74.3% clip. See [here](https://www.teamrankings.com/ncb/odds-history/win/) for history.

## My Accuracy: 72.3%

Hypertuning a Logistic Regression model after conducting Principal Component Analysis to reduce variance, yielded an overall prediction accuracy just 2% shy of Vegas bookmakers. At the 50% threshold, that's an AUC-ROC of 0.79

![png](/images/NCAA_Basketball/aucroc.png)

### Performance vs. "Difficulty"

Logically, we expect the model to perform best in matchups that would be considered "blow-outs", and worse when teams are considered more evenly matched. Sure enough, that seems to be the case, when we reference the difference in SRS as a measure of "closeness" of matchup:

![png](/images/NCAA_Basketball/diff.png)

The model performs with over 90% accuracy when there is at least a 20-pt SRS Difference, but drops to 58% when the SRS is within 5 between competing teams. And because conferences are constructed so that more evenly matched teams are often competed, these matchups tend to make up the bulk of regular season games:

![png](/images/NCAA_Basketball/matchups.png)

Thus, our model cumulatively peaks at ~72.4%:

![png](/images/NCAA_Basketball/cumulative.png)

---

# Conclusion

We can.
