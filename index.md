# Who’s Hot and Who’s Not: Using Python to Track NBA Players

**Have you ever wondered which NBA players are truly “on fire” 🔥 and which ones are in a slump ❄️?**
Fans argue about it every day, but we can settle it with data. In this tutorial, we’ll use Python and the [nba_api](https://github.com/swar/nba_api) library to compare *recent performance* against *season averages*. By the end, you’ll be able to generate your own hot-or-not charts for any player.

![Line chart showing LeBron James points vs. rolling average](lebron.jpg "Hot streak visualization")


---

## Why This Matters

Sports conversations often rely on gut feeling. A player hits a few threes in a row, and suddenly Twitter declares them “unstoppable.” But without context, those judgments are misleading. By calculating **rolling averages** and comparing them to season performance, we get a fairer picture of trends.

For STAT-386 students, this tutorial is a great practice in:

* pulling real data from an API,
* cleaning and summarizing with pandas, and
* visualizing results with matplotlib.

---

## Step 1: Set Up Your Tools

First, make sure you have Python installed (3.9 or higher). Then install the following libraries:

```bash
pip install nba_api pandas matplotlib
```

If you run into any problems it might help to run the following code directly into your pyhon terminal.

```python
# Install nba_api if not already installed
%pip install nba_api pandas
import sys
!{sys.executable} -m pip install nba_api
!{sys.executable} -m pip install matplotlib
```

You can work in a Jupyter notebook (`pip install notebook` then `jupyter notebook`) or any Python IDE.

---

## Step 2: Pull NBA Game Data

The `nba_api` package connects directly to NBA Stats endpoints. Let’s fetch LeBron James’s game logs for the current season, just to see how the data is formatted. Additionally we'll grab all active players.

```python
from nba_api.stats.static import players
from nba_api.stats.endpoints import playergamelog
import pandas as pd
import matplotlib.pyplot as plt

# Get all active players
all_players = players.get_active_players()
# print(all_players[0])  # show example structure

# Build dictionary of active players
player_dict = {p['full_name']: p['id'] for p in all_players}
len(player_dict)

# LeBron James has player_id = 2544
gamelog = playergamelog.PlayerGameLog(player_id=2544, season='2024-25')
df = gamelog.get_data_frames()[0]

# Quick peek for an example
print(df.head())
```

This gives us game-by-game stats like points (`PTS`), rebounds, assists, and shooting percentages.

---

## Step 3: Define “Hot” vs. “Not”

We’ll focus on points scored, but you could adapt this to shooting %, assists, etc.

We could also include multiple different categories and assign them specific values to calculate how **Hot** a player is. We will not be doing this because I am somewhat lazy.

The key idea is the **rolling average**:

$$
MA_t = \frac{1}{k} \sum_{i=0}^{k-1} x_{t-i}
$$

where ( k ) is the number of games in the window (we’ll use 5).

We'll also put some limits on what players we'll look at, such as setting the minimum amount of games played as 40,and the minimum minutes as 20.

```python
# Let's filter Players so they meet certain criteria
season = "2024-25"
# Minimum games played this season
min_games = 40
# Minimum minutes per game
min_mins = 20
# Game amount to determine hotness
window = 5

#playergamelog.PlayerGameLog(player_id=1630173, season='2024-25').get_data_frames()[0]

# Player list
player_list = []
```
### For Loop

1. Player Filtering
- Iterates through a dictionary of players and their IDs.  
- For each player, retrieves their 2025–26 game log.  
- Excludes players who:
  - Have too few games played  
  - Average fewer than 20 minutes per game  

2. Performance Calculation
- Computes each player’s **season average points per game**.  
- Computes the **recent average points** over a defined window of games (e.g., last 5 games).  
- Calculates a **Hotness Ratio**:  
  \[
  \text{Ratio} = \frac{\text{Recent Average}}{\text{Season Average}}
  \]

3. Status Assignment
- If ratio > **1.05** → Player is **Hot**  
- If ratio < **0.95** → Player is **Cold**  
- Otherwise → Player is **Neutral**  

4. Results Compilation
- Stores results (Player name, Season Avg, Recent Avg, Ratio, Status) in a list.  
- Converts the list into a Pandas DataFrame.  
- Displays the first 15 players for review.  

---

**Summary:**  
The code identifies active players with significant minutes, evaluates their recent scoring trends compared to their season average, and provides a quick snapshot of which players are currently **overperforming, underperforming, or consistent**. 

```python
# Shrink the dictionary to filter for all players that have 
# over 20 minutes for the 2024-25 season
for name, pid in player_dict.items():
    
    # Get dataframe of games for player
    gamelog = playergamelog.PlayerGameLog(player_id=pid, season='2025-26').get_data_frames()[0]
    
    # If the gamelog is empty or their are not enough minimum games
    if gamelog.empty or len(gamelog) < min_games:
        # Do not include this player
        continue

    # Get the season average of minutes per players
    season_avg_mins = gamelog["MIN"].mean()
    # If the season average is under 20 minutes
    if season_avg_mins < min_mins:
        # Do not include this player
        continue

    # Get season average and recent average
    season_avg = gamelog["PTS"].mean()
    recent_avg = gamelog.head(window)["PTS"].mean()

    # Get Hottness ratio
    ratio = recent_avg / season_avg

    status = ""
    # Get the Status
    if ratio > 1.05:
        status = "Hot"
    elif ratio < 0.95:
        status = "Cold"
    else:
        status = "Neutral"
    
    # Append to list
    player_list.append({
        "Player": name,
        "Season Avg": round(season_avg, 1),
        "Last 5 Avg": round(recent_avg, 1),
        "Ratio": round(ratio, 1),
        "Status": status
    })

# Change to Data Frame
df_pl = pd.DataFrame(player_list)

# Grab 15 players
df_pl.head(15)
```

Now we can say whether LeBron is **hot** (recent_avg > 1.05 * season_avg) or **cold** (recent_avg < 0.95 season_avg) or **neutral** (1.05 * season_avg < recent_avg < 0.95 season_avg).

---

## Step 4: Compare Players with a Table

Let’s pretend we ran the same analysis for a few stars. A **Markdown table** makes the comparison tidy:

| Player                    | Season Avg PPG | Last 5 Games PPG | Status |
| ------------------------- | -------------- | ---------------- | ------ |
| Shai Gilgeous-Alexander   | 32.7           | 29.8             | ❄️ Cold |
| Giannis Antetokounmpo     | 30.4           | 33.0             | 🔥 Hot |
| Nikola Jovik              | 29.6           | 26.2             | ❄️ Cold |
| Jayson Tatum              | 26.8           | 28.1             | 🔥 Hot |

---

## Step 5: Visualize Hotness

Numbers tell part of the story, but a chart shows trends. Let’s plot players points per game along with their rolling average:

```python
plt.figure(figsize=(10,8))

# Bar colors based on status
colors = df_pl["Status"].map({"Hot": "red", "Neutral": "gray", "Not": "blue"})

# Scatter plot
plt.scatter(df_pl["Season Avg"], df_pl["Last 5 Avg"], color=colors, s=100)

# Diagonal line y = x
max_val = max(df_pl["Season Avg"].max(), df_pl["Last 5 Avg"].max()) + 5
plt.plot([0, max_val], [0, max_val], color="black", linestyle="--", label="y = Season Avg")

# Labels
plt.xlabel("Season Average Points")
plt.ylabel("Recent 5 Games Average Points")
plt.title("NBA Players: Season vs Recent Average Points")
plt.legend(["y = Season Avg", "Players"], loc="upper left")
plt.grid(True, linestyle='--', alpha=0.5)
plt.tight_layout()
plt.show()
```

## Conclusion

By using a publicly available NBA stats API, this code demonstrates how to gather real game data, filter players by meaningful criteria, and analyze performance trends over time. The process shows how to combine Python with libraries like `pandas` to clean, calculate, and visualize player statistics. Through this project, you learned not only practiced working with APIs and data frames but also learned how to translate raw sports data into insights such as identifying players who are currently "Hot," "Cold," or "Neutral."

## Extend the Analysis

Want to go further? Here are some ideas:

* **Multiple Metrics:** Add shooting percentage or assists to the hot/not criteria.
* **Different Windows:** Use 3-game or 10-game averages instead of 5.
* **Team Analysis:** Check whether a whole team is hot or not by averaging 
