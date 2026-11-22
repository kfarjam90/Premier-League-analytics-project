# Premier League Analytics & Fantasy Squad Prediction

An interactive football analytics project built with **Power BI, DAX, and Python**, combining Premier League reporting with a model that forecasts fantasy points and selects a squad for the next supplied gameweek.

**Author — Parham Farjam**  
**Report snapshot — 22 November 2026**

## Project overview

Explore league standings, match results, player rankings, club form, and head-to-head records across seasons. The Fantasy Prediction page presents a squad selected using predicted player points, availability, formation rules, club limits, and a budget.

The current report includes historical seasons from **2000/01** and the ongoing **2026/27** campaign. Its 2026/27 match results contain **107 completed matches through 21 November 2026**.

## Dashboard pages

| Page | Purpose |
|---|---|
| Home | Introduces the project and provides navigation. |
| Overview | Summarizes matches, goals, leading clubs, and individual leaders for the selected season. |
| League Table | Compares points, wins, draws, losses, goals, goal difference, and recent form. |
| Results | Explores scorelines and match highlights using matchweek controls. |
| Players | Ranks goals, assists, goalkeeper clean sheets, and FPL points, with player detail cards. |
| Club Form | Tracks cumulative points and results over time. |
| Head to Head | Compares two clubs through season statistics and recent league meetings. |
| Fantasy Prediction | Displays the predicted starting XI, captain, and ordered substitutes on a football pitch. |

Season slicers support historical comparisons. HTML measures provide custom cards, rankings, comparison panels, and player hover details.

## Fantasy prediction

### Target and features

The target is **fantasy points for one player in one match**.

The script builds rolling form features from each player's previous matches:

- Number of available matches in the form window.
- Mean, standard deviation, and most recent fantasy points.
- Mean minutes played.
- Share of matches with an appearance.
- Share of matches with at least 60 minutes played.

The default form window is **five matches**, configurable with `--form-games`. Players without history receive empty-form features.

### Models and validation

Two regressors independently predict match points:

1. `HistGradientBoostingRegressor`.
2. `MLPRegressor`, using standardized features.

Their predictions receive equal weight.

Training uses the earliest approximately **80% of unique kickoff timestamps**, with the later timestamps reserved for validation. All players at the same kickoff stay on the same side of the split. Rolling features are calculated before recording that match's outcomes.

Validation simulates sequential match prediction. Earlier validation results can become form inputs for later validation matches once those results are known. The scaler and models fit on training rows only during evaluation. After evaluation, the models refit on all historical rows before the prediction cutoff.

### Validation results

The following results come from the saved November 22 run:

| Model | MAE — fantasy points per player-match |
|---|---:|
| Training-mean baseline | 1.667 |
| Gradient boosting | **1.144** |
| Neural network | 1.169 |
| Equal-weight ensemble | 1.147 |

Lower MAE indicates smaller average prediction error. In this run, the ensemble reduced MAE by approximately **31%** against the baseline. Gradient boosting performed slightly better than the ensemble.

The neural network reached its 250-iteration limit without fully converging. These results describe one chronological holdout split. Repeated gameweek backtesting would provide a broader assessment.

### Availability and fixtures

- Injured, suspended, and unavailable players are excluded.
- Doubtful players receive an availability multiplier capped at 50%.
- A supplied availability probability further adjusts projected points.
- Players with no fixture receive zero projected gameweek points.
- Double-gameweek projections sum the forecast across scheduled fixtures.

The current model uses recent points and minutes. Opponent strength, home advantage, expected goals, and explicit starting-probability models are potential extensions.

### Squad optimization

The optimizer uses `scipy.optimize.milp` with binary starter and bench decisions.

| Constraint | Rule |
|---|---|
| Squad size | 15 players |
| Position quotas | 2 GK, 5 DEF, 5 MID, 3 FWD |
| Starting XI | 11 players |
| Starting formation | 1 GK, 3–5 DEF, 2–5 MID, 1–3 FWD |
| Club limit | Maximum 3 players per club |
| Budget | £100.0m by default when prices are supplied |
| Objective | Predicted starter points + 10% of predicted bench points |

The highest-projected starter is suggested as captain. Outfield substitutes are ordered by projected points, with the reserve goalkeeper identified separately.

Projected XI totals are shown **before the captain bonus**. Automatic substitutions, transfer costs, chips, existing squad ownership, and selling prices are outside the current optimizer.

## Running the Python model

Use Python 3.10 or later and install the dependencies:

```bash
python -m pip install pandas numpy scipy scikit-learn threadpoolctl
```

Place `fantasy_squad.py`, `history.csv`, `players.csv`, and `fixtures.csv` in the same directory.

```bash
python fantasy_squad.py --history history.csv --players players.csv --fixtures fixtures.csv --form-games 5 --budget 100 --output squad.csv
```

To reproduce the saved November 22 snapshot, add the prediction cutoff:

```bash
python fantasy_squad.py --history history.csv --players players.csv --fixtures fixtures.csv --form-games 5 --budget 100 --as-of 2026-11-22 --output squad.csv
```

Use `--no-budget` to disable the budget constraint. If the roster has no price column, the script also runs without a budget constraint.

The script prints model MAE, the selected XI, substitutes, captain, opponents, and projections, then writes the squad to CSV.

### Input files

| File | Required columns | Optional columns |
|---|---|---|
| `history.csv` | `player_id`, `kickoff`, `total_points` | `minutes` |
| `players.csv` | `player_id`, `name`, `club`, `position` | `price`, `status`, `availability` |
| `fixtures.csv` | `kickoff`, `home_club`, `away_club` | `gameweek` |

Use ISO dates or timestamps, prices in millions, and availability from **0 to 1**. Player positions are `GK`, `DEF`, `MID`, or `FWD`. Club labels must match between the roster and fixtures.

History should contain one row per eligible player per completed fixture, including zero points and zero minutes for non-appearances. Use only outcomes available before the prediction cutoff.

By default, player IDs must be stable across the input files. The script also accepts FPL `merged_gw.csv` aliases. For multiple files with season-local IDs, use `--history-mode fpl-season`, provide one complete history CSV per season with repeated `--history` arguments, and place the current roster's season last.

When gameweek labels are supplied, the script selects remaining fixtures in the earliest future gameweek. Otherwise, it uses a seven-day window beginning at the first future kickoff.

### Google Colab

Upload the Python script and the three CSVs to the Colab Files panel, then run:

```python
!pip -q install pandas numpy scipy scikit-learn threadpoolctl
!python /content/fantasy_squad.py --history /content/history.csv --players /content/players.csv --fixtures /content/fixtures.csv --form-games 5 --budget 100 --as-of 2026-11-22 --output /content/squad.csv
```

Download the result:

```python
from google.colab import files
files.download("/content/squad.csv")
```

## Opening the Power BI project

1. Extract the complete project ZIP.
2. Keep `PremierLeague.pbip`, `PremierLeague.Report`, and `PremierLeague.SemanticModel` together.
3. Open `PremierLeague.pbip` in Power BI Desktop and select **Refresh**.
4. Select **2026/27** in the season slicer.

The updated project uses embedded data snapshots. External image URLs require network access. The files and embedded rows were checked structurally. Power BI Desktop rendering and refresh still require verification.

The **Predict Next Week** button reveals the saved `squad` data already loaded into Power BI. For a new forecast, refresh the input files, rerun Python, and replace the model's `squad` source or embedded snapshot.

## Data model

| Table | Role |
|---|---|
| `Matches` | Completed match results and available match statistics. |
| `Standings` | Club totals and ranks for each season. |
| `SeasonTopFive` | Top five clubs for each season. |
| `FormData` | Club match history and cumulative points. |
| `Seasons` | Main season slicer dimension. |
| `CompareSeason`, `TeamA`, `TeamB` | Head-to-head selectors. |
| `Meetings` | Recent league meetings used by legacy comparison visuals. |
| `PlayerDetails` | Player-season goals, assists, minutes, FPL points, and goalkeeper records. |
| `ClubTopScorers`, `GoalkeeperLeaders` | Supporting season leader tables. |
| `squad` | Saved Python prediction output. |

Season values use the text format `YYYY/YY`. Main season relationships filter fact tables from `Seasons`. Custom head-to-head measures explicitly apply the selected clubs and comparison season.

## Data sources and coverage

- Football-Data provides downloadable match results and statistics. [Football-Data](https://www.football-data.co.uk/englandm.php)
- Official FPL feeds provide fixtures, match points, prices, and availability. [Fantasy Premier League](https://fantasy.premierleague.com/)
- FotMob provides the current-season league assist snapshot. League assists and FPL assists use different definitions. [FotMob Premier League statistics](https://www.fotmob.com/leagues/47/stats/premier-league)
- Historical player-season data was assembled from the Premier League history dataset and the FPL archive. [Premier League history dataset](https://github.com/ethankaufman/PL-History-Dashboard) · [FPL archive](https://github.com/vaastav/Fantasy-Premier-League)

Coverage varies by metric:

- Goals, league assists, minutes, and goalkeeper data cover the historical report seasons.
- FPL points and bonus coverage begins in **2016/17**.
- The seven newly added Gameweek 11 matches have blank card, shot, and referee fields pending publication in the downloadable match source.
- Current league assists may include live play at retrieval time. FPL points and minutes in the updated report were aggregated from completed fixtures.

Third-party data, badges, and player images remain subject to their source terms.

## Weekly publication workflow

1. Refresh completed match history, current roster, prices, and availability.
2. Check the upcoming gameweek and fixture schedule.
3. Run the model and inspect validation metrics and squad constraints.
4. Load the resulting squad into Power BI and check slicer behavior.
5. Publish the gameweek walkthrough with the snapshot date and projected-point labels.
6. After matches finish, compare the saved forecasts with actual points.

Weekly FPL squad selections, captain suggestions, and performance reviews are the intended focus of the accompanying video series.
