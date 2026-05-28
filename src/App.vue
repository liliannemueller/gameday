<script setup>
import { ref, computed } from 'vue'

const NFL_TEAMS = [
  { id: 1, name: 'Atlanta Falcons', abbr: 'ATL', primary: '#A71930', secondary: '#000000' },
  { id: 2, name: 'Buffalo Bills', abbr: 'BUF', primary: '#00338D', secondary: '#C60C30' },
  { id: 3, name: 'Chicago Bears', abbr: 'CHI', primary: '#0B162A', secondary: '#C83803' },
  { id: 4, name: 'Cincinnati Bengals', abbr: 'CIN', primary: '#FB4F14', secondary: '#000000' },
  { id: 5, name: 'Cleveland Browns', abbr: 'CLE', primary: '#311D00', secondary: '#FF3C00' },
  { id: 6, name: 'Dallas Cowboys', abbr: 'DAL', primary: '#003594', secondary: '#869397' },
  { id: 7, name: 'Denver Broncos', abbr: 'DEN', primary: '#FB4F14', secondary: '#002244' },
  { id: 8, name: 'Detroit Lions', abbr: 'DET', primary: '#0076B6', secondary: '#B0B7BC' },
  { id: 9, name: 'Green Bay Packers', abbr: 'GB', primary: '#203731', secondary: '#FFB612' },
  { id: 10, name: 'Tennessee Titans', abbr: 'TEN', primary: '#0C2340', secondary: '#4B92DB' },
  { id: 11, name: 'Indianapolis Colts', abbr: 'IND', primary: '#002C5F', secondary: '#A2AAAD' },
  { id: 12, name: 'Kansas City Chiefs', abbr: 'KC', primary: '#E31837', secondary: '#FFB81C' },
  { id: 13, name: 'Las Vegas Raiders', abbr: 'LV', primary: '#000000', secondary: '#A5ACAF' },
  { id: 14, name: 'Los Angeles Rams', abbr: 'LAR', primary: '#003594', secondary: '#FFA300' },
  { id: 15, name: 'Miami Dolphins', abbr: 'MIA', primary: '#008E97', secondary: '#FC4C02' },
  { id: 16, name: 'Minnesota Vikings', abbr: 'MIN', primary: '#4F2683', secondary: '#FFC62F' },
  { id: 17, name: 'New England Patriots', abbr: 'NE', primary: '#002244', secondary: '#C60C30' },
  { id: 18, name: 'New Orleans Saints', abbr: 'NO', primary: '#101820', secondary: '#D3BC8D' },
  { id: 19, name: 'New York Giants', abbr: 'NYG', primary: '#0B2265', secondary: '#A71930' },
  { id: 20, name: 'New York Jets', abbr: 'NYJ', primary: '#125740', secondary: '#000000' },
  { id: 21, name: 'Philadelphia Eagles', abbr: 'PHI', primary: '#007A33', secondary: '#A5ACAF' },
  { id: 22, name: 'Arizona Cardinals', abbr: 'ARI', primary: '#97233F', secondary: '#000000' },
  { id: 23, name: 'Pittsburgh Steelers', abbr: 'PIT', primary: '#101820', secondary: '#FFB612' },
  { id: 24, name: 'Los Angeles Chargers', abbr: 'LAC', primary: '#0080C6', secondary: '#FFC20E' },
  { id: 25, name: 'San Francisco 49ers', abbr: 'SF', primary: '#AA0000', secondary: '#B3995D' },
  { id: 26, name: 'Seattle Seahawks', abbr: 'SEA', primary: '#002244', secondary: '#69BE28' },
  { id: 27, name: 'Tampa Bay Buccaneers', abbr: 'TB', primary: '#D50A0A', secondary: '#FF7900' },
  { id: 28, name: 'Washington Commanders', abbr: 'WAS', primary: '#5A1414', secondary: '#FFB612' },
  { id: 29, name: 'Carolina Panthers', abbr: 'CAR', primary: '#0085CA', secondary: '#101820' },
  { id: 30, name: 'Jacksonville Jaguars', abbr: 'JAX', primary: '#006778', secondary: '#D7A22A' },
  { id: 33, name: 'Baltimore Ravens', abbr: 'BAL', primary: '#241773', secondary: '#000000' },
  { id: 34, name: 'Houston Texans', abbr: 'HOU', primary: '#03202F', secondary: '#A71930' },
]

const currentTeam = computed(() => {
  return NFL_TEAMS.find(t => t.id === parseInt(selectedTeam.value)) || null
})

const teamColors = computed(() => {
  if (currentTeam.value) {
    return {
      primary: currentTeam.value.primary,
      secondary: currentTeam.value.secondary,
    }
  }
  return {
    primary: '#1a56db',
    secondary: '#3b82f6',
  }
})

const selectedTeam = ref('')
const minTotalScore = ref(40)
const minPassingYards = ref(250)
const minRushingYards = ref(100)
const requireOvertime = ref(false)
const requireCloseGame = ref(false)
const closeGameMargin = ref(7)

const loading = ref(false)
const error = ref('')
const games = ref([])
const analyzedGames = ref([])

async function fetchGames() {
  if (!selectedTeam.value) {
    error.value = 'Please select a team'
    return
  }

  loading.value = true
  error.value = ''
  games.value = []
  analyzedGames.value = []

  try {
    const teamId = selectedTeam.value.toString()
    const teamGames = []

    // Fetch weeks 1-18 of regular season
    for (let week = 1; week <= 18; week++) {
      try {
        const res = await fetch(
          `https://site.api.espn.com/apis/site/v2/sports/football/nfl/scoreboard?week=${week}&seasontype=2&year=2025`
        )
        const data = await res.json()

        // Filter for games involving the selected team that are completed
        const events = data.events || []
        for (const event of events) {
          const comp = event.competitions?.[0]
          if (!comp) continue

          const isCompleted = comp.status?.type?.completed
          const teams = comp.competitors || []
          const involvesTeam = teams.some(t => t.team?.id === teamId)

          if (isCompleted && involvesTeam) {
            teamGames.push(event)
          }
        }
      } catch (e) {
        console.error(`Error fetching week ${week}:`, e)
      }
    }

    // Analyze each game (get detailed stats)
    for (const game of teamGames.slice(0, 17)) {
      const gameId = game.id
      try {
        const summaryRes = await fetch(
          `https://site.api.espn.com/apis/site/v2/sports/football/nfl/summary?event=${gameId}`
        )
        const summary = await summaryRes.json()

        const analyzed = analyzeGame(summary, game)
        if (analyzed) {
          analyzedGames.value.push(analyzed)
        }
      } catch (e) {
        console.error(`Error fetching game ${gameId}:`, e)
      }
    }

    if (analyzedGames.value.length === 0) {
      error.value = 'No completed games found for this team'
    }
  } catch (e) {
    error.value = 'Failed to fetch games. Please try again.'
    console.error(e)
  } finally {
    loading.value = false
  }
}

function analyzeGame(summary, game) {
  try {
    const boxscore = summary.boxscore
    const competition = game.competitions[0]
    const teams = competition.competitors

    const homeTeam = teams.find(t => t.homeAway === 'home')
    const awayTeam = teams.find(t => t.homeAway === 'away')

    // Scores come as strings from the scoreboard API
    const homeScore = parseInt(homeTeam?.score) || 0
    const awayScore = parseInt(awayTeam?.score) || 0
    const totalScore = homeScore + awayScore
    const scoreDiff = Math.abs(homeScore - awayScore)

    // Extract stats from boxscore
    let totalPassingYards = 0
    let totalRushingYards = 0

    if (boxscore?.teams) {
      for (const team of boxscore.teams) {
        const stats = team.statistics || []
        for (const stat of stats) {
          if (stat.name === 'netPassingYards') {
            totalPassingYards += parseInt(stat.displayValue) || 0
          }
          if (stat.name === 'rushingYards') {
            totalRushingYards += parseInt(stat.displayValue) || 0
          }
        }
      }
    }

    // Check for overtime - period > 4 or check linescores length
    const linescores = homeTeam?.linescores || []
    const isOvertime = linescores.length > 4 || competition.status?.period > 4

    return {
      id: game.id,
      date: new Date(game.date).toLocaleDateString(),
      week: game.week?.number || 0,
      homeTeam: homeTeam?.team?.displayName || 'Unknown',
      awayTeam: awayTeam?.team?.displayName || 'Unknown',
      homeScore,
      awayScore,
      totalScore,
      scoreDiff,
      totalPassingYards,
      totalRushingYards,
      isOvertime,
    }
  } catch (e) {
    console.error('Error analyzing game:', e)
    return null
  }
}

const recommendations = computed(() => {
  return analyzedGames.value.map(game => {
    const reasons = []
    let shouldWatch = true

    // Check total score
    if (game.totalScore < minTotalScore.value) {
      reasons.push(`Low scoring (${game.totalScore} pts)`)
      shouldWatch = false
    } else {
      reasons.push(`High scoring (${game.totalScore} pts)`)
    }

    // Check passing yards
    if (game.totalPassingYards < minPassingYards.value) {
      reasons.push(`Low passing (${game.totalPassingYards} yds)`)
      shouldWatch = false
    } else {
      reasons.push(`Good passing (${game.totalPassingYards} yds)`)
    }

    // Check rushing yards
    if (game.totalRushingYards < minRushingYards.value) {
      reasons.push(`Low rushing (${game.totalRushingYards} yds)`)
      shouldWatch = false
    } else {
      reasons.push(`Good rushing (${game.totalRushingYards} yds)`)
    }

    // Check overtime requirement
    if (requireOvertime.value && !game.isOvertime) {
      reasons.push('No overtime')
      shouldWatch = false
    } else if (game.isOvertime) {
      reasons.push('Went to OT!')
    }

    // Check close game requirement
    if (requireCloseGame.value && game.scoreDiff > closeGameMargin.value) {
      reasons.push(`Not close (${game.scoreDiff} pt margin)`)
      shouldWatch = false
    } else if (game.scoreDiff <= closeGameMargin.value) {
      reasons.push(`Close game (${game.scoreDiff} pt margin)`)
    }

    return {
      ...game,
      shouldWatch,
      reasons,
    }
  })
})
</script>

<template>
  <div
    class="app"
    :style="{
      '--primary-color': teamColors.primary,
      '--secondary-color': teamColors.secondary,
    }"
  >
    <header>
      <h1>GameDay</h1>
      <p class="tagline">Should you watch the game?</p>
    </header>

    <main>
      <div class="controls">
        <div class="form-group">
          <label for="team">Select Your Team</label>
          <select id="team" v-model="selectedTeam">
            <option value="" disabled>Choose a team...</option>
            <option v-for="team in NFL_TEAMS" :key="team.id" :value="team.id">
              {{ team.name }}
            </option>
          </select>
        </div>

        <div class="params">
          <h3>Your Preferences</h3>

          <div class="param-row">
            <label>Min Total Score</label>
            <input type="number" v-model="minTotalScore" min="0" />
          </div>

          <div class="param-row">
            <label>Min Passing Yards (combined)</label>
            <input type="number" v-model="minPassingYards" min="0" />
          </div>

          <div class="param-row">
            <label>Min Rushing Yards (combined)</label>
            <input type="number" v-model="minRushingYards" min="0" />
          </div>

          <div class="param-row checkbox">
            <input type="checkbox" id="overtime" v-model="requireOvertime" />
            <label for="overtime">Only show overtime games</label>
          </div>

          <div class="param-row checkbox">
            <input type="checkbox" id="closeGame" v-model="requireCloseGame" />
            <label for="closeGame">Only show close games</label>
          </div>

          <div v-if="requireCloseGame" class="param-row">
            <label>Max point margin</label>
            <input type="number" v-model="closeGameMargin" min="1" max="50" />
          </div>
        </div>

        <button class="btn" @click="fetchGames" :disabled="loading">
          {{ loading ? 'Loading games... (this may take a moment)' : 'Check Games' }}
        </button>

        <p v-if="error" class="error">{{ error }}</p>
      </div>

      <div v-if="recommendations.length > 0" class="results">
        <h2>Game Analysis</h2>

        <div
          v-for="game in recommendations"
          :key="game.id"
          class="game-card"
          :class="{ watch: game.shouldWatch, skip: !game.shouldWatch }"
        >
          <div class="game-header">
            <span class="verdict">{{ game.shouldWatch ? '👍' : '👎' }}</span>
            <span class="date">Week {{ game.week }} - {{ game.date }}</span>
          </div>

          <div class="matchup">
            <span>{{ game.awayTeam }}</span>
            <span class="vs">@</span>
            <span>{{ game.homeTeam }}</span>
          </div>

          <div class="reasons">
            <span
              v-for="(reason, i) in game.reasons"
              :key="i"
              class="reason-tag"
            >
              {{ reason }}
            </span>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<style>
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  min-height: 100vh;
  color: white;
}

.app {
  --primary-color: #1a56db;
  --secondary-color: #3b82f6;
  max-width: 800px;
  margin: 0 auto;
  padding: 2rem;
  min-height: 100vh;
  background: linear-gradient(135deg, color-mix(in srgb, var(--primary-color) 80%, black) 0%, color-mix(in srgb, var(--primary-color) 40%, black) 50%, color-mix(in srgb, var(--secondary-color) 30%, black) 100%);
  transition: background 0.5s ease;
}

header {
  text-align: center;
  margin-bottom: 2rem;
}

h1 {
  font-family: 'Bebas Neue', sans-serif;
  font-size: 4rem;
  letter-spacing: 0.05em;
  background: linear-gradient(90deg, var(--primary-color), var(--secondary-color));
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  transition: background 0.5s ease;
}

.tagline {
  color: rgba(255, 255, 255, 0.7);
  font-size: 1.1rem;
}

.controls {
  background: rgba(255, 255, 255, 0.1);
  padding: 1.5rem;
  border-radius: 16px;
  backdrop-filter: blur(10px);
}

.form-group {
  margin-bottom: 1.5rem;
}

label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: 500;
}

select, input[type="number"] {
  width: 100%;
  padding: 0.75rem;
  font-size: 1rem;
  border: 2px solid rgba(255, 255, 255, 0.2);
  border-radius: 8px;
  background: rgba(255, 255, 255, 0.1);
  color: white;
  transition: border-color 0.3s ease;
}

select:focus, input[type="number"]:focus {
  outline: none;
  border-color: var(--secondary-color);
}

select option {
  background: #1a1a2e;
  color: white;
}

.params {
  margin-bottom: 1.5rem;
}

.params h3 {
  margin-bottom: 1rem;
  font-size: 1.1rem;
}

.param-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.75rem;
}

.param-row input[type="number"] {
  width: 100px;
}

.param-row.checkbox {
  justify-content: flex-start;
  gap: 0.5rem;
}

.param-row.checkbox input {
  width: auto;
}

.btn {
  width: 100%;
  padding: 1rem;
  font-size: 1.1rem;
  font-weight: 600;
  border: none;
  border-radius: 8px;
  background: linear-gradient(90deg, var(--primary-color), var(--secondary-color));
  color: white;
  cursor: pointer;
  transition: transform 0.2s, opacity 0.2s, background 0.5s ease;
}

.btn:hover:not(:disabled) {
  transform: translateY(-2px);
}

.btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.error {
  color: #ff6b6b;
  text-align: center;
  margin-top: 1rem;
}

.results {
  margin-top: 2rem;
}

.results h2 {
  margin-bottom: 1rem;
}

.game-card {
  background: rgba(255, 255, 255, 0.1);
  border-radius: 12px;
  padding: 1rem;
  margin-bottom: 1rem;
  border-left: 4px solid;
}

.game-card.watch {
  border-left-color: #4ade80;
}

.game-card.skip {
  border-left-color: #f87171;
}

.game-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 0.5rem;
}

.verdict {
  font-size: 1.5rem;
}

.date {
  color: rgba(255, 255, 255, 0.6);
  font-size: 0.9rem;
}

.matchup {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 1.1rem;
  margin-bottom: 0.75rem;
}

.vs {
  font-weight: 500;
  color: rgba(255, 255, 255, 0.5);
}

.reasons {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
}

.reason-tag {
  background: rgba(255, 255, 255, 0.15);
  padding: 0.25rem 0.5rem;
  border-radius: 4px;
  font-size: 0.8rem;
}

@media (max-width: 600px) {
  .app {
    padding: 1rem;
  }

  h1 {
    font-size: 2.75rem;
  }

  .tagline {
    font-size: 0.95rem;
  }

  .controls {
    padding: 1rem;
  }

  .param-row {
    flex-direction: column;
    align-items: flex-start;
    gap: 0.4rem;
  }

  .param-row input[type="number"] {
    width: 100%;
  }

  .param-row.checkbox {
    flex-direction: row;
    align-items: center;
  }

  .matchup {
    font-size: 0.9rem;
    gap: 0.25rem;
  }

  .matchup span:first-child,
  .matchup span:last-child {
    flex: 1;
    min-width: 0;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
}
</style>
