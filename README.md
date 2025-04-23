## Step 1: Introduction
The goal of this analysis is to find interesting insights into how player positions (i.e. top lane, jungle, mid lane, etc.) are exemplified by data. The first question investigated by this report is: which position is most valuable to their team? Every League of Legends (LoL) player likes to think they carry their team, but is there really one clear MVP?

To answer this question I'm using a dataset provided by [Oracle's Elixir](https://oracleselixir.com/tools/downloads). It contains 150,589 rows of data on 2022 LoL esports matches from various professional leagues. Relevent columns in this dataset include 'kills,' 'deaths,' and 'assists' which represent total kills, deaths, and assists (respectively) by a player in a match (pretty self-explanatory). There's also a 'champion' column providing the name of which character a player selected. Last but not least is 'position,' a column describing either a player's role in the match or indicating that the row is a 'team' row. Team rows summarize the team's overall stats, but are not relevant to this analysis. For player rows, 'position' is either 'top' for top lane, 'jng' for jungle, 'mid' for mid lane, 'bot' for bottom lane, or 'sup' for support.

## Step 2: Data Cleaning and Exploratory Data Analysis

### Data Cleaning
The dataset contained both player and team rows as mentioned in the introduction. This would be an issue during all data analysis steps because team kill/death/assist and all other player stats would effectively be double counted. I removed the team rows from the data set by extracting all rows with the exception of rows with the value of 'team' in the position column, and putting them in a separate DataFrame. Much later into this analysis I noticed that the column 'datacompleteness' became relevant because NaN values in my calculations were causing errors in model fitting. To resolve this, I removed all rows which had 'partial' in the 'datacompleteness' column, which from my inspection were missing large sums of player stats.

Below are the first few rows of the cleaned dataset (subsetted to include only relevant columns):

| teamname          | playername   | position   | champion   |   kills |   deaths |   assists |
|:------------------|:-------------|:-----------|:-----------|--------:|---------:|----------:|
| BRION Challengers | Soboro       | top        | Renekton   |       2 |        3 |         2 |
| BRION Challengers | Raptor       | jng        | Xin Zhao   |       2 |        5 |         6 |
| BRION Challengers | Feisty       | mid        | LeBlanc    |       2 |        2 |         3 |
| BRION Challengers | Gamin        | bot        | Samira     |       2 |        4 |         2 |
| BRION Challengers | Loopy        | sup        | Leona      |       1 |        5 |         6 |

### Univariate Analysis
 <iframe
 src="assets/univariate-distribution.html"
 width="610"
 height="450"
 frameborder="0"
 ></iframe>
The plot above shows distribution of the 'kills' column in the dataset as a bar chart of kill count distributions. The plot is heavily right skewed meaning most players do not get more than 1 or 2 kills per game, but the tail to the right may consist of higher kills by a given position. More analysis is necessary though...

### Bivariate Analysis
 <iframe
 src="assets/bivariate-distribution.html"
 width="610"
 height="450"
 frameborder="0"
 ></iframe>
 The plot above shows the distribution of players' kill/death/assist ratios by position, calculated as (kills + assists) / deaths. The medians are all quite close together but mid and bot lane players are tied for highest, indicating that they tend to contribute more kills and assists per life than other positions making them more valuable.

### Interesting Aggregate Analysis

|   Total Kills |   Total Assists |   Total Deaths |
|--------------:|----------------:|---------------:|
|         90744 |          115420 |          54840 |
|         65951 |          146571 |          67002 |
|         75662 |          124670 |          57494 |
|         19094 |          196947 |          69156 |
|         59659 |          107754 |          63253 |

Above is a table aggregating total kills, assists, and deaths by position. It shows a significant difference in kills and assists among positions, with junglers having more kills but fewer assists, while supports have relatively few kills but the highest amount of assists.

### Can We Answer the Question?
For the purposes of this analysis, being the "most valuable" position should entail contributing the most kills and assists while minimizing deaths. The KDA ratio captures this directly, but unfortunately there is not one clear winner. Middle and bottom laners appear to be the most valuable positions thanks to their high median ratios, while top lane is clearly the worst with a low median ratio (sorry top laners).

## Step 3: Framing a Prediction Problem
### Setup
It's time to make things more challenging. Now that we have some idea of how different positions perform, we can make a prediction about a player's position. More specifically, can we answer the qustion: what was a player's position, using their post-game stats? Considering this is post-game information, any player data (except for the position column obviously) would be fair game, but this analysis will exclude the 'champion' column. LoL players tend to pick from very specific groups of champions given that they are tailor made for a certain position, essentially making the prediction pointless if they were included.

### Problem Type
This is a multiclass classification problem because the goal is to train a model to make a prediction given a discrete set of positions, of which there are 5 (making it non-binary). The response variable will be position because this column directly indicates the true answer. 

### Metric
To assess the performance of the model, this analysis will use the accuracy metric because class-imbalance is not an issue for this dataset (each player has a different position on each team, so each position is represented equally). Thus, there is no need to worry about the accuracy of one position inflating the overall model performance. For this problem, it is a straight-forward approach to seeing how often the model is correct.

## Step 4: Baseline Model
### Model Algorithm and Features
The baseline model for this analysis used the Random Forest algorithm which creates a complex series of decision trees to perform classification. The model was very simple with only two features: 'earned gpm' and 'kills.' The 'earned gpm' column represents how much gold (valuable for player upgrades) a player earned per minute of the game and the 'kills' column remains the same as before. 'earned gpm' was included with the hopes that positions that do not standout by kills might stand out in the gold that they earn from other tasks in the game (like killing minions). Both 'earned gpm' and 'kills' are quantitative variables so they could be trained on without encodings.

### Performance
After making a train/test split (25% test data), fitting the model, and testing, it had an average accuracy of 42%. Its only strong suit was classifying supports with 81% accuracy. This is fairly bad performance considering that, from a gameplay perspective, each position's playstyle is fairly distinct, especially junglers which only got an accuracy of 38%.

## Step 5: Final Model
### New Features
There were a total of 7 features added and 1 feature removed from the final model comapred to the baseline. A feature called 'kda' was created by adding a FunctionTransformer to the model pipeline which modified the dataset to include the KDA ratio for each player. This calculation uses 'kills' and keeping 'kills' was not found to provide useful information, so it was removed. The reasoning for adding 'kda' to improve the model is that some roles can be distinguished better by their assists and kills per death rather than just total kills. This was seen in the aggregate table in a previous analysis section. The second feature created for the model was 'kp' (kill participation) which was calculated using the same method as 'kda' but using the formula (kills + assists) / (team kills). It represents the ratio of kills a player participated in, meaning either getting the kill or assisting. The rationale for including this feature was that it could distinguish positions like mid lane which can be involved in a high percent of attacks on players around the map by roaming.

A existing feature of the dataset 'dpm' (damage per minute) was added to the model because mid laners also tend to have high damage output (by the design of their champions) and the model was still lacking accuracy in mid lane classification. 'damagetakenperminute' (damange taken per minute) was added to the model because it could identify positions that tend to be attacked more by the opposing team or are tankier (able to take more damage). 'damagemitigatedperminute' (damage mitigated per minute) represents preventing damage to other players on your team. It was added because the support position is meant to help other players from dying by mitigating damage. The feature 'cspm' (creep score per minute) was added because it counts killing minions, monsters, and wards which is typically common for mid lane players to do. Lastly, 'monsterkills' (monster kills) was added specifically to help identify junglers because that's where monsters spawn.

### Model Selection
4 model algorithms were tested with the final feature selection including Random Forest, a multilayer perceptron (MLP) network, Gaussian Naive Bayes, and K-Nearest Neighbors. Although the average accuracy of the Random Forest model and the MLP model were equal, the mid lane accuracy (the weakest for all of the models) was highest for the Random Forest method, making it the best option for the final model.

### Hyperparameter Tuning
The model's performance was improved using hyperparameter tuning with a 5-fold cross validated grid search. Different values for the 'n_estimators' parameter were tested because it determines the number of trees in the algorithm, which will affect how complex decisions can be. 'max_depth' was tested because it determines depth of trees, allowing for further complexity and therefore possible accuracy. Lastly, 'min_samples_split' was tested because it affects when trees split, which could prevent overfitting or underfitting.

The optimal parameter values were n_estimators=300, max_depth=20, and min_samples_split=2.

### Performance
The model's performance is visualized using the confusion matrix below:
 <iframe
 src="assets/confusion-matrix.html"
 width="620"
 height="550"
 frameborder="0"
 ></iframe>
From the matrix you can see that the mid lane was the least accurately identified, often being confused for the top and bottom lanes. 

This is an improvement over the baseline model in every way. The average accuracy increased to 81% (up 39 percentage-points), and while mid lane accuracy is still the weakest it jumped from 27% to 60%.

### Overall Evaluation
The model is a significant improvement from baseline but still not ideal. It seems mid lane is very difficult to distinguish from the top and bottom lanes which is logical considering those three positions share the similarity of playing in a lane with towers and minions. 


## Final Note
Unfortunately, I do not have the time to dedicate myself to becoming diamond tier, but this analysis was my best effort to appreciate the complexity of League of Legends, and maybe one day stop feeding in the bot lane...
