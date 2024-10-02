---
layout: default
title: Model Major League Baseball
permalink: /mlb/
---
{:refdef: style="text-align: center;"}
![Baseeball on Clay](/assets/baseball-on-clay.jpg)
{:refdef }

I have always loved baseball as a game, and I have many fond memories of playing it while growing up.  When I went to MIT for college, I lived in the Back Bay of Boston within walking distance of Fenway park, and I've been a Red Sox fan at heart ever since.  I've found it tough to stay invested in the team without being able to watch the games regularly, but I rediscovered my passion for the game when my son recently started T-ball.  This seemed like a perfect opportunity to channel this interest into build a data project with the main goal of predicting MLB game outcomes.  You can inspect the Github project [here](https://github.com/mikebrasher/baseball).

## Problem Statement

For any major league game, the starting lineup is published ahead of time.  Based on the past performance of these players, a model should be able to make a prediction of the outcome of the game.  There are no ties in baseball, so this is simply a binary classification problem, will the home team win or not?

My first goal in building the model was finding a reliable data source.  I found a dataset on [kaggle](https://www.kaggle.com/datasets/jraddick/baseball-events-from-retrosheetorg/discussion?sort=hotness) which includes play-by-play description of every regular season game from 1914-2022.  This is based on a massive effort by volunteers at [retrosheet](https://www.retrosheet.org/) to create a centralized database of game records.  Stored as .csv files, each at-bat is recorded as an event with pitcher, batter, fielders, pitch sequence, and play recorded among other data.  As an example play, consider the following:

`the_play='46(1)3/GDP/G4', base_running='3-4'`

Separated by '**/**' characters, the first sequence '**46(1)3**' represents a ground double play: ball hit to second base (position 4), thrown to the short stop (position 6), runner from first out (1), and then thrown to first base to get the runner out (position 3).  This allows us to assign putouts to the short stop and first baseman and an assist to the second baseman.  The second sequence contains a modifier describing the play; in this case '**GDP**' stands for ground double play.  The final '**G4**' records where and how the ball was hit, i.e. ground ball to second base.  Additionally, the base running '**3-4**' indicates that there was a runner on third base who scored.


## ETL Pipeline

Based on the reference at retrosheet, I developed a parser in Python to decode the plays in this format and fill a series of data structures with batting, base running, pitching and fielding statistics.  I created over 120 unit tests, and was able to closely track game and season statistics for individual players and teams.  In practice, I found the retrosheet data to be more reliable than some online sourcees I compared to.  To track down discrepancies for individual games, I was able to search the internet for video highlights of modern games and verify that the retrosheet data was indeed more accurate.

The next step was to utilize this parser in an extract transform load (ETL) pipeline.  To eliminate redundant computation and improve efficiency in data loading, the general flow that I settled on is:
1. **Extract** - Read plays from .csv file for specific season, create a dictionary of all game events keyed by a unique game ID
2. **Transform** - For each game, determine the game result and starting lineup, and create performance features for every player listed *specific to that game*
3. **Load** - Ensuring that games are processed sequentially by date, an exclusive prefix sum of features is built for every player encountered.  Then, the feature sums for the starting lineup are concatenated and stored along with the game result in a SQLite database.

Each play is parsed exactly once, and since the prefix sum stored in the database is exclusive, then only information available prior to the game is associated with the actual result of the game.  I then wrapped this database in a Pytorch data loader which queries the database for some list of game IDs.  I found the SQLite SELECT query to be the slowest operation, so aggregating games to be pulled at once greatly sped up data retrieval.

As a baseline, this data was used to train an XGBoost model with the following training split
* Train: seasons 1914-2020
* Validation: season 2021
* Test: season 2022


## XGBoost Model Performance

The baseline XGBoost model trained on the above data achieves an accuracy of 55% on the test data.  This is promising in that it predicts the winner better than chance.  Based on the discussion here, online sports betting predicts games correctly 60% of the time, so there is obviously room for improvement.  The two most promising areas of improvement both involve including more data in the features:
1. Include designated hitter (DH) features - Pitchers are almost always weak batters, so a designated hitter is substituted for them
2. The current model uses the pitching features of the starting pitcher for the entire game.  It is quite rare that a single pitcher throws the entire game, and there are often several substitutions.  I plan on approximating the bullpen based on previously seen pitchers and including their features as well.

Additionally, baseball parks are not standardized in their dimensions, and there are location specific effects (like altitude) that impart a variable home field advantage.  The XGBoost model predicts a probability that the home team wins, and I currently use a uniform 0.5 threshold to predict the winner.  I plan on augmenting the model with per park prediction cutoffs that favor the home team in prediction.
