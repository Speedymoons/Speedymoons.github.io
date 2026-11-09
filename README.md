# Speedymoons.github.io
Testing simulator
Here is the complete source code for the **Reality Show Simulator** React application. It incorporates all setup options, twist mechanics (Double Eliminations, Team Swaps, Purge Events, Non-Elimination Weeks, and Redemption Buybacks), step-by-step 8-page episodic simulation flow, single-vote nomination rules, and the dynamic Wikia-style Progress Track Table with Gold/Silver placements.

You can save this as `App.jsx` inside any React project (such as standard Create React App or Vite setup with Tailwind CSS):

```jsx
import React, { useState, useEffect } from 'react';
import { 
  Users, Play, Shield, Award, RotateCcw, Flame, Sparkles, 
  Settings, ChevronRight, Skull, ArrowRight, UserPlus, CheckCircle2,
  TrendingUp, Eye, Dices, RefreshCw, Layers
} from 'lucide-react';

// Pre-configured default archetype templates
const ARCHETYPES = [
  { name: 'The Mastermind', desc: 'High strategy and physical awareness. Prone to orchestrating blindsides.', physical: 7, strategic: 9, social: 6, temper: 5 },
  { name: 'The Challenge Beast', desc: 'Dominates physical arenas but can struggle with social nuances.', physical: 9, strategic: 5, social: 5, temper: 6 },
  { name: 'The Social Butterfly', desc: 'Well-liked by everyone. Great at evading nominations.', physical: 5, strategic: 6, social: 9, temper: 4 },
  { name: 'The Wildcard', desc: 'Erratic and high temper. Keeps everyone guessing.', physical: 6, strategic: 7, social: 4, temper: 9 },
  { name: 'The Underdog', desc: 'Resilient and balanced. Excels under pressure.', physical: 6, strategic: 6, social: 7, temper: 3 },
];

export default function App() {
  // --- STATE MANAGEMENT ---
  const [phase, setPhase] = useState('SETUP'); // SETUP, CASTING, SIMULATION, FINALE
  const [currentPage, setCurrentPage] = useState(1); // 1-8 for weekly episodic pages

  // Season Configuration Settings
  const [settings, setSettings] = useState({
    seasonName: "Challenge: Battle for Supremacy",
    contestantCount: 12,
    finaleSize: 3, // 2, 3, 4, 5
    finaleFormat: "JURY", // JURY, CHALLENGE, BOTH
    seasonType: "SOLO", // SOLO, TEAMS
    twists: {
      doubleElim: { active: true, week: 3 },
      teamSwap: { active: false },
      purge: { active: true, week: 2 },
      nonElim: { active: true, week: 4 },
      redemption: { active: false, week: 5 }
    }
  });

  // Cast Data State
  const [cast, setCast] = useState([]);
  
  // Weekly Game Engine State
  const [currentWeek, setCurrentWeek] = useState(1);
  const [weeklyData, setWeeklyData] = useState(null);
  const [weeklyHistory, setWeeklyHistory] = useState([]); // Stores snapshot history per week for track rendering

  // --- INITIALIZE CAST ---
  const initializeCast = (count) => {
    const defaultNames = [
      "Mitzy", "Colin", "Yoru", "Conut", "Jet", "Miguel", 
      "Louisa", "Zero", "Carson", "Kai", "Arden", "Holy",
      "Maddy", "Mads", "Junebug", "Raymond", "Blake", "Sofi",
      "Tate", "Nico", "Sierra", "Kira", "Jax", "Elena"
    ];

    const newCast = Array.from({ length: count }, (_, i) => {
      const name = defaultNames[i] || `Contestant ${i + 1}`;
      const archetype = ARCHETYPES[i % ARCHETYPES.length];
      return {
        id: `player_${i + 1}`,
        name: name,
        gender: i % 2 === 0 ? "Female" : "Male",
        photo: `https://api.dicebear.com/7.x/bottts/svg?seed=${name}`,
        archetype: archetype.name,
        stats: {
          physical: archetype.physical,
          strategic: archetype.strategic,
          social: archetype.social,
          temper: archetype.temper
        },
        status: "ACTIVE", // ACTIVE, ELIMINATED
        eliminatedWeek: null,
        placement: null,
        team: settings.seasonType === "TEAMS" ? (i % 2 === 0 ? "Red Team" : "Blue Team") : null
      };
    });
    setCast(newCast);
  };

  useEffect(() => {
    initializeCast(settings.contestantCount);
  }, [settings.contestantCount, settings.seasonType]);

  // --- GAME ENGINE: SIMULATE WEEK ---
  const startNextWeek = () => {
    const activePlayers = cast.filter(p => p.status === "ACTIVE");
    const eliminatedPlayers = cast.filter(p => p.status === "ELIMINATED");

    // Check if Finale reached
    if (activePlayers.length <= settings.finaleSize) {
      runFinaleSequence();
      return;
    }

    // Check Twists for current week
    const isDoubleElim = settings.twists.doubleElim.active && settings.twists.doubleElim.week === currentWeek;
    const isPurge = settings.twists.purge.active && settings.twists.purge.week === currentWeek && activePlayers.length > 6;
    const isNonElim = settings.twists.nonElim.active && settings.twists.nonElim.week === currentWeek;
    const isRedemptionWeek = settings.twists.redemption.active && settings.twists.redemption.week === currentWeek && eliminatedPlayers.length > 0;

    // PAGE 1: Camp Life & Events
    const events = generateCampEvents(activePlayers);

    // PAGE 2: Challenge Phase
    let purgeEliminated = null;
    let commander = null;
    let winningTeam = null;

    if (isPurge) {
      // Last person to 3 points in Purge is ousted instantly
      const purgeScores = {};
      activePlayers.forEach(p => purgeScores[p.id] = 0);
      let purgeLoser = null;
      while (!purgeLoser) {
        const scorer = activePlayers[Math.floor(Math.random() * activePlayers.length)];
        purgeScores[scorer.id] = (purgeScores[scorer.id] || 0) + 1;
        const reached = Object.values(purgeScores).filter(s => s >= 3).length;
        if (reached === activePlayers.length - 1) {
          purgeLoser = activePlayers.find(p => purgeScores[p.id] < 3);
        }
      }
      purgeEliminated = purgeLoser;
    }

    // Standard Immunity/Reward Comp
    if (settings.seasonType === "TEAMS") {
      winningTeam = Math.random() > 0.5 ? "Red Team" : "Blue Team";
    } else {
      commander = activePlayers[Math.floor(Math.random() * activePlayers.length)];
    }

    // PAGE 3: Strategy & Risk
    const vulnerable = activePlayers.filter(p => p.id !== commander?.id);
    const targetA = vulnerable[Math.floor(Math.random() * vulnerable.length)];
    let targetB = vulnerable[Math.floor(Math.random() * vulnerable.length)];
    while (targetB.id === targetA.id && vulnerable.length > 1) {
      targetB = vulnerable[Math.floor(Math.random() * vulnerable.length)];
    }

    // PAGE 4 & 5: Single Vote Nominations Engine (Top 2 or Top 3 into Arena)
    const votes = [];
    const voteCounts = {};
    activePlayers.forEach(p => voteCounts[p.id] = 0);

    const poolToVote = vulnerable;
    activePlayers.forEach(voter => {
      // SINGLE VOTE PER PERSON
      const validTargets = poolToVote.filter(p => p.id !== voter.id);
      const chosen = validTargets[Math.floor(Math.random() * validTargets.length)];
      votes.push({ voter: voter.name, target: chosen.name, targetId: chosen.id });
      voteCounts[chosen.id] = (voteCounts[chosen.id] || 0) + 1;
    });

    // Rank players by vote counts
    const sortedByVotes = Object.entries(voteCounts)
      .map(([id, count]) => ({ id, count, player: activePlayers.find(p => p.id === id) }))
      .sort((a, b) => b.count - a.count);

    // Number of duelists sent to Ring/Duel (Double Elim sends 3, Standard sends 2)
    const duelistsCount = isDoubleElim ? 3 : 2;
    const nominatedDuelists = sortedByVotes.slice(0, duelistsCount).map(item => item.player);

    // PAGE 6: Elimination Duel Arena
    let duelLosers = [];
    let duelWinners = [];

    if (isNonElim) {
      duelWinners = nominatedDuelists; // Both remain safe
    } else {
      // Simulate Duel Comp (First to 3 points)
      const duelScores = {};
      nominatedDuelists.forEach(d => duelScores[d.id] = 0);
      
      let duelComplete = false;
      while (!duelComplete) {
        // Weighted by Physical stat
        const weights = nominatedDuelists.map(d => d.stats.physical);
        const totalWeight = weights.reduce((a, b) => a + b, 0);
        let rnd = Math.random() * totalWeight;
        let chosenDuelist = nominatedDuelists[0];
        
        for (let i = 0; i < nominatedDuelists.length; i++) {
          if (rnd < weights[i]) {
            chosenDuelist = nominatedDuelists[i];
            break;
          }
          rnd -= weights[i];
        }

        duelScores[chosenDuelist.id] += 1;
        
        // Winner declared when reaching 3 points
        if (duelScores[chosenDuelist.id] >= 3) {
          duelWinners.push(chosenDuelist);
          duelLosers = nominatedDuelists.filter(d => d.id !== chosenDuelist.id);
          duelComplete = true;
        }
      }
    }

    // Process Eliminations
    const weeklyEliminations = [];
    if (purgeEliminated) {
      purgeEliminated.status = "ELIMINATED";
      purgeEliminated.eliminatedWeek = currentWeek;
      weeklyEliminations.push(purgeEliminated);
    }

    if (!isNonElim) {
      duelLosers.forEach(loser => {
        loser.status = "ELIMINATED";
        loser.eliminatedWeek = currentWeek;
        weeklyEliminations.push(loser);
      });
    }

    // Save Week State Summary
    const weekSnapshot = {
      week: currentWeek,
      commander,
      winningTeam,
      events,
      votes,
      voteCounts,
      nominatedDuelists,
      duelWinners,
      duelLosers,
      isNonElim,
      isDoubleElim,
      purgeEliminated,
      eliminatedThisWeek: weeklyEliminations
    };

    setWeeklyData(weekSnapshot);
    setWeeklyHistory(prev => [...prev, weekSnapshot]);
    setCurrentPage(1);
    setPhase('SIMULATION');
  };

  // --- FINALE ENGINE ---
  const runFinaleSequence = () => {
    const finalists = cast.filter(p => p.status === "ACTIVE");
    const jury = cast.filter(p => p.status === "ELIMINATED");

    const juryVotes = {};
    finalists.forEach(f => juryVotes[f.id] = 0);

    const individualJuryVotes = [];
    jury.forEach(juror => {
      // Jury members vote based on Social + Strategic preference
      const favorite = finalists[Math.floor(Math.random() * finalists.length)];
      juryVotes[favorite.id] += 1;
      individualJuryVotes.push({ jurorName: juror.name, votedFor: favorite.name });
    });

    // Challenge Finale scoring
    const challengeScores = {};
    finalists.forEach(f => challengeScores[f.id] = Math.floor(Math.random() * 5) + f.stats.physical);

    // Winner Calculation based on Format
    let winner = null;
    let totalScores = {};

    finalists.forEach(f => {
      if (settings.finaleFormat === "JURY") {
        totalScores[f.id] = juryVotes[f.id];
      } else if (settings.finaleFormat === "CHALLENGE") {
        totalScores[f.id] = challengeScores[f.id];
      } else {
        // BOTH (Jury vote = 1 pt, Challenge win = 2 pts)
        totalScores[f.id] = (juryVotes[f.id] * 1) + (challengeScores[f.id] * 2);
      }
    });

    const sortedFinalists = [...finalists].sort((a, b) => totalScores[b.id] - totalScores[a.id]);
    winner = sortedFinalists[0];

    // Assign final placements
    sortedFinalists.forEach((player, idx) => {
      player.placement = idx === 0 ? "WINNER" : idx === 1 ? "RUNNER UP" : `${idx + 1}ND RUNNER UP`;
    });

    // Assign placement to eliminated players based on elimination order (reverse week)
    const eliminatedSorted = cast
      .filter(p => p.status === "ELIMINATED")
      .sort((a, b) => b.eliminatedWeek - a.eliminatedWeek);

    eliminatedSorted.forEach((player, idx) => {
      player.placement = `ELIMINATED (W${player.eliminatedWeek})`;
    });

    setWeeklyData({
      isFinale: true,
      finalists: sortedFinalists,
      winner,
      juryVotes,
      individualJuryVotes,
      challengeScores,
      totalScores
    });

    setPhase('FINALE');
  };

  // Dynamic Camp Event Generator
  const generateCampEvents = (players) => {
    const p1 = players[Math.floor(Math.random() * players.length)];
    let p2 = players[Math.floor(Math.random() * players.length)];
    while (p2.id === p1.id && players.length > 1) {
      p2 = players[Math.floor(Math.random() * players.length)];
    }

    return [
      {
        type: "SPOTLIGHT",
        title: "Spotlight",
        players: [p1],
        desc: `${p1.name} gets openly written off by half the villa this week and says nothing. Just gets on with it. It reads well.`,
        statChange: "+2 Popularity"
      },
      {
        type: "ROMANCE",
        title: "Drama & Friction",
        players: [p1, p2],
        desc: `${p1.name} and ${p2.name} get into a heated debate over camp duties and voting strategies.`,
        statChange: "-1 Relationship (all pairs)"
      },
      {
        type: "BONDING",
        title: "Bonding",
        players: [p1, p2],
        desc: `${p1.name} and ${p2.name} fall into a 6am workout routine that nobody else has the energy to join. An unspoken duo forms.`,
        statChange: "+1 Relationship · +2 Popularity"
      }
    ];
  };

  // Next Page Handler inside simulation
  const handleNextPage = () => {
    if (currentPage < 8) {
      setCurrentPage(prev => prev + 1);
    } else {
      // Advance Week
      setCurrentWeek(prev => prev + 1);
      startNextWeek();
    }
  };

  // --- DYNAMIC TRACK TABLE PLAYER SORTING ---
  // Ranks players so active/winners sit at top, down to 1st eliminated at bottom
  const getSortedCastForTrack = () => {
    return [...cast].sort((a, b) => {
      if (a.status === "ACTIVE" && b.status === "ELIMINATED") return -1;
      if (a.status === "ELIMINATED" && b.status === "ACTIVE") return 1;
      
      if (a.placement === "WINNER") return -1;
      if (b.placement === "WINNER") return 1;
      if (a.placement === "RUNNER UP") return -1;
      if (b.placement === "RUNNER UP") return 1;

      // Both eliminated: higher week stays higher up
      if (a.eliminatedWeek !== b.eliminatedWeek) {
        return (b.eliminatedWeek || 0) - (a.eliminatedWeek || 0);
      }
      return 0;
    });
  };

  return (
    <div className="min-h-screen bg-slate-950 text-slate-100 font-sans p-4 md:p-8">
      {/* HEADER NAVBAR */}
      <header className="max-w-7xl mx-auto flex justify-between items-center pb-6 border-b border-slate-800 mb-8">
        <div className="flex items-center gap-3">
          <Flame className="w-8 h-8 text-pink-500 animate-pulse" />
          <h1 className="text-2xl font-black tracking-wider bg-gradient-to-r from-pink-500 via-purple-500 to-indigo-500 bg-clip-text text-transparent">
            {settings.seasonName.toUpperCase()}
          </h1>
        </div>
        <div className="flex gap-4">
          <span className="bg-slate-800 px-3 py-1 rounded-full text-xs font-semibold text-slate-300 flex items-center gap-1">
            <Users className="w-4 h-4 text-pink-400" /> {cast.filter(c => c.status === "ACTIVE").length} Active
          </span>
          <span className="bg-slate-800 px-3 py-1 rounded-full text-xs font-semibold text-slate-300">
            Week {currentWeek}
          </span>
        </div>
      </header>

      {/* ==================== PHASE 1: CONFIGURATION & SETUP ==================== */}
      {phase === 'SETUP' && (
        <div className="max-w-5xl mx-auto grid grid-cols-1 md:grid-cols-2 gap-8">
          {/* Setup Inputs Panel */}
          <div className="bg-slate-900 border border-slate-800 rounded-2xl p-6 shadow-xl">
            <h2 className="text-xl font-bold mb-6 text-pink-400 flex items-center gap-2">
              <Settings className="w-5 h-5" /> Season Settings
            </h2>

            <div className="space-y-4">
              <div>
                <label className="text-xs text-slate-400 font-semibold block mb-1">SEASON NAME</label>
                <input 
                  type="text"
                  value={settings.seasonName}
                  onChange={(e) => setSettings({...settings, seasonName: e.target.value})}
                  className="w-full bg-slate-800 border border-slate-700 rounded-lg p-2.5 text-sm focus:outline-none focus:border-pink-500"
                />
              </div>

              <div className="grid grid-cols-2 gap-4">
                <div>
                  <label className="text-xs text-slate-400 font-semibold block mb-1">CONTESTANTS (4-24)</label>
                  <input 
                    type="number"
                    min="4"
                    max="24"
                    value={settings.contestantCount}
                    onChange={(e) => setSettings({...settings, contestantCount: parseInt(e.target.value) || 12})}
                    className="w-full bg-slate-800 border border-slate-700 rounded-lg p-2.5 text-sm focus:outline-none focus:border-pink-500"
                  />
                </div>

                <div>
                  <label className="text-xs text-slate-400 font-semibold block mb-1">FINALE SIZE</label>
                  <select 
                    value={settings.finaleSize}
                    onChange={(e) => setSettings({...settings, finaleSize: parseInt(e.target.value)})}
                    className="w-full bg-slate-800 border border-slate-700 rounded-lg p-2.5 text-sm focus:outline-none focus:border-pink-500"
                  >
                    <option value={2}>Top 2</option>
                    <option value={3}>Top 3</option>
                    <option value={4}>Top 4</option>
                    <option value={5}>Top 5</option>
                  </select>
                </div>
              </div>

              <div className="grid grid-cols-2 gap-4">
                <div>
                  <label className="text-xs text-slate-400 font-semibold block mb-1">FINALE FORMAT</label>
                  <select 
                    value={settings.finaleFormat}
                    onChange={(e) => setSettings({...settings, finaleFormat: e.target.value})}
                    className="w-full bg-slate-800 border border-slate-700 rounded-lg p-2.5 text-sm focus:outline-none focus:border-pink-500"
                  >
                    <option value="JURY">Jury Vote</option>
                    <option value="CHALLENGE">Challenge Finale</option>
                    <option value="BOTH">Both (Hybrid)</option>
                  </select>
                </div>

                <div>
                  <label className="text-xs text-slate-400 font-semibold block mb-1">SEASON TYPE</label>
                  <select 
                    value={settings.seasonType}
                    onChange={(e) => setSettings({...settings, seasonType: e.target.value})}
                    className="w-full bg-slate-800 border border-slate-700 rounded-lg p-2.5 text-sm focus:outline-none focus:border-pink-500"
                  >
                    <option value="SOLO">Solo (Individual)</option>
                    <option value="TEAMS">Teams / Tribes</option>
                  </select>
                </div>
              </div>
            </div>

            {/* Twist Configurations */}
            <h3 className="text-sm font-bold text-slate-300 mt-6 mb-3 flex items-center gap-2">
              <Dices className="w-4 h-4 text-pink-400" /> Active Twists Schedule
            </h3>
            <div className="space-y-3 bg-slate-950 p-4 rounded-xl border border-slate-800">
              <div className="flex items-center justify-between">
                <span className="text-xs font-medium">Double Elimination (3 in Arena)</span>
                <input 
                  type="checkbox"
                  checked={settings.twists.doubleElim.active}
                  onChange={(e) => setSettings({
                    ...settings, 
                    twists: {...settings.twists, doubleElim: {...settings.twists.doubleElim, active: e.target.checked}}
                  })}
                  className="accent-pink-500 w-4 h-4"
                />
              </div>

              <div className="flex items-center justify-between">
                <span className="text-xs font-medium">Purge Event (Instant Kill before F6)</span>
                <input 
                  type="checkbox"
                  checked={settings.twists.purge.active}
                  onChange={(e) => setSettings({
                    ...settings, 
                    twists: {...settings.twists, purge: {...settings.twists.purge, active: e.target.checked}}
                  })}
                  className="accent-pink-500 w-4 h-4"
                />
              </div>

              <div className="flex items-center justify-between">
                <span className="text-xs font-medium">Non-Elimination Week</span>
                <input 
                  type="checkbox"
                  checked={settings.twists.nonElim.active}
                  onChange={(e) => setSettings({
                    ...settings, 
                    twists: {...settings.twists, nonElim: {...settings.twists.nonElim, active: e.target.checked}}
                  })}
                  className="accent-pink-500 w-4 h-4"
                />
              </div>
            </div>

            <button 
              onClick={() => setPhase('CASTING')}
              className="mt-6 w-full bg-gradient-to-r from-pink-500 to-purple-600 hover:from-pink-600 hover:to-purple-700 py-3 rounded-xl font-bold flex items-center justify-center gap-2 shadow-lg shadow-pink-500/20 transition-all"
            >
              Continue to Cast Builder <ChevronRight className="w-5 h-5" />
            </button>
          </div>

          {/* Preset Summary Box */}
          <div className="bg-slate-900/50 border border-slate-800 rounded-2xl p-6 flex flex-col justify-between">
            <div>
              <h3 className="text-lg font-bold text-slate-200 mb-2">Simulated Ruleset Overview</h3>
              <p className="text-xs text-slate-400 mb-6">
                Contestants cast <span className="text-pink-400 font-bold">1 vote each</span> for nomination. The top 2 voted contestants face off in the Elimination Arena.
              </p>

              <div className="space-y-3 text-xs">
                <div className="bg-slate-800/60 p-3 rounded-lg border border-slate-700/50">
                  <span className="font-bold text-pink-400">Voting Rules:</span> 1 Vote per player. Top 2 overall vote-getters compete in the Arena duel.
                </div>
                <div className="bg-slate-800/60 p-3 rounded-lg border border-slate-700/50">
                  <span className="font-bold text-pink-400">Double Elim Twist:</span> Top 3 vote getters enter elimination; only 1 winner returns.
                </div>
                <div className="bg-slate-800/60 p-3 rounded-lg border border-slate-700/50">
                  <span className="font-bold text-pink-400">Finale Structure:</span> Top {settings.finaleSize} finalists enter finale ({settings.finaleFormat} format).
                </div>
              </div>
            </div>

            <div className="bg-slate-950 p-4 rounded-xl border border-slate-800 text-center text-xs text-slate-500">
              Ready to construct your player roster.
            </div>
          </div>
        </div>
      )}

      {/* ==================== PHASE 2: CAST BUILDER ==================== */}
      {phase === 'CASTING' && (
        <div className="max-w-6xl mx-auto">
          <div className="flex justify-between items-center mb-6">
            <h2 className="text-xl font-bold text-pink-400 flex items-center gap-2">
              <Users className="w-5 h-5" /> Customize Contestants ({cast.length})
            </h2>
            <button 
              onClick={() => {
                setCurrentPage(1);
                startNextWeek();
              }}
              className="bg-gradient-to-r from-pink-500 to-purple-600 px-6 py-2.5 rounded-xl font-bold flex items-center gap-2 shadow-lg shadow-pink-500/20"
            >
              Simulate Now! <Play className="w-4 h-4 fill-current" />
            </button>
          </div>

          <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
            {cast.map((player, idx) => (
              <div key={player.id} className="bg-slate-900 border border-slate-800 rounded-2xl p-5 shadow-lg relative">
                <div className="flex items-center gap-4 mb-4">
                  <img 
                    src={player.photo} 
                    alt={player.name}
                    className="w-14 h-14 rounded-full bg-slate-800 border-2 border-pink-500/30 object-cover" 
                  />
                  <div>
                    <input 
                      type="text"
                      value={player.name}
                      onChange={(e) => {
                        const updated = [...cast];
                        updated[idx].name = e.target.value;
                        setCast(updated);
                      }}
                      className="bg-slate-800 border border-slate-700 rounded px-2 py-1 text-sm font-bold w-full mb-1"
                    />
                    <span className="text-xs text-slate-400 font-medium">{player.archetype}</span>
                  </div>
                </div>

                {/* Stats Sliders */}
                <div className="space-y-2 text-xs">
                  <div>
                    <div className="flex justify-between text-slate-400 mb-1">
                      <span>Physical</span>
                      <span className="font-bold text-pink-400">{player.stats.physical}</span>
                    </div>
                    <input 
                      type="range" min="1" max="10" 
                      value={player.stats.physical}
                      onChange={(e) => {
                        const updated = [...cast];
                        updated[idx].stats.physical = parseInt(e.target.value);
                        setCast(updated);
                      }}
                      className="w-full accent-pink-500 bg-slate-800 h-1.5 rounded-lg"
                    />
                  </div>

                  <div>
                    <div className="flex justify-between text-slate-400 mb-1">
                      <span>Strategic</span>
                      <span className="font-bold text-purple-400">{player.stats.strategic}</span>
                    </div>
                    <input 
                      type="range" min="1" max="10" 
                      value={player.stats.strategic}
                      onChange={(e) => {
                        const updated = [...cast];
                        updated[idx].stats.strategic = parseInt(e.target.value);
                        setCast(updated);
                      }}
                      className="w-full accent-purple-500 bg-slate-800 h-1.5 rounded-lg"
                    />
                  </div>
                </div>
              </div>
            ))}
          </div>
        </div>
      )}

      {/* ==================== PHASE 3: 8-PAGE EPISODIC SIMULATION ==================== */}
      {phase === 'SIMULATION' && weeklyData && (
        <div className="max-w-4xl mx-auto bg-slate-900 border border-slate-800 rounded-3xl p-6 md:p-8 shadow-2xl">
          {/* Page Progress Indicator */}
          <div className="flex justify-between items-center mb-6 pb-4 border-b border-slate-800 text-xs font-semibold text-slate-400">
            <span>WEEK {currentWeek} • PAGE {currentPage} OF 8</span>
            <div className="flex gap-1.5">
              {[1,2,3,4,5,6,7,8].map(p => (
                <div 
                  key={p} 
                  className={`w-2.5 h-2.5 rounded-full ${p === currentPage ? 'bg-pink-500 scale-125' : p < currentPage ? 'bg-pink-900' : 'bg-slate-800'}`} 
                />
              ))}
            </div>
          </div>

          {/* PAGE 1: CAMP LIFE EVENTS */}
          {currentPage === 1 && (
            <div className="space-y-6">
              <h2 className="text-xl font-black text-pink-400 uppercase tracking-wider flex items-center gap-2">
                <Sparkles className="w-5 h-5" /> Page 1: Villa & Camp Life Events
              </h2>
              
              <div className="space-y-4">
                {weeklyData.events.map((evt, idx) => (
                  <div key={idx} className="bg-slate-950 border border-slate-800 rounded-2xl p-5">
                    <span className="text-xs font-bold text-pink-400 tracking-wider uppercase mb-2 block">{evt.title}</span>
                    <div className="flex items-center gap-3 mb-3">
                      {evt.players.map(p => (
                        <div key={p.id} className="flex items-center gap-2">
                          <img src={p.photo} className="w-8 h-8 rounded-full border border-pink-500/40" />
                          <span className="text-sm font-bold">{p.name}</span>
                        </div>
                      ))}
                    </div>
                    <p className="text-sm text-slate-300 mb-3">{evt.desc}</p>
                    <span className="inline-block bg-slate-900 px-2.5 py-1 rounded text-xs font-semibold text-emerald-400 border border-slate-800">
                      {evt.statChange}
                    </span>
                  </div>
                ))}
              </div>
            </div>
          )}

          {/* PAGE 2: COMPETITION & IMMUNITY */}
          {currentPage === 2 && (
            <div className="space-y-6">
              <h2 className="text-xl font-black text-pink-400 uppercase tracking-wider flex items-center gap-2">
                <Award className="w-5 h-5" /> Page 2: Daily Competition
              </h2>

              {weeklyData.purgeEliminated && (
                <div className="bg-red-950/40 border border-red-800 p-4 rounded-2xl flex items-center gap-4">
                  <Skull className="w-8 h-8 text-red-500" />
                  <div>
                    <h4 className="font-bold text-red-400 text-sm">SURPRISE PURGE EVENT!</h4>
                    <p className="text-xs text-slate-300">
                      {weeklyData.purgeEliminated.name} finished dead last in the purge comp and was instantly eliminated!
                    </p>
                  </div>
                </div>
              )}

              <div className="bg-slate-950 p-6 rounded-2xl border border-slate-800 text-center">
                <h3 className="text-sm font-bold text-slate-400 mb-2">IMMUNITY WINNER</h3>
                {weeklyData.commander ? (
                  <div className="inline-flex items-center gap-3 bg-slate-900 px-6 py-3 rounded-full border border-pink-500/40">
                    <img src={weeklyData.commander.photo} className="w-10 h-10 rounded-full" />
                    <span className="text-lg font-black text-pink-400">{weeklyData.commander.name}</span>
                  </div>
                ) : (
                  <div className="text-lg font-bold text-pink-400">{weeklyData.winningTeam} Wins Immunity!</div>
                )}
              </div>
            </div>
          )}

          {/* PAGE 3: STRATEGY & RISK */}
          {currentPage === 3 && (
            <div className="space-y-6">
              <h2 className="text-xl font-black text-pink-400 uppercase tracking-wider flex items-center gap-2">
                <Flame className="w-5 h-5" /> Page 3: Strategy & Risk Discussions
              </h2>
              <div className="bg-slate-950 p-6 rounded-2xl border border-slate-800 space-y-4">
                <p className="text-sm text-slate-300 leading-relaxed">
                  With the nomination vote approaching, vulnerable players are scrambling to form alliances and secure safety. 
                  Single-vote dynamics mean target consolidation is crucial to avoid being forced into the Elimination Arena.
                </p>
              </div>
            </div>
          )}

          {/* PAGE 4: VOTING REVEAL */}
          {currentPage === 4 && (
            <div className="space-y-6">
              <h2 className="text-xl font-black text-pink-400 uppercase tracking-wider flex items-center gap-2">
                <Eye className="w-5 h-5" /> Page 4: Voting Reveal Sequence
              </h2>

              <div className="bg-slate-950 p-6 rounded-2xl border border-slate-800">
                <h3 className="text-xs font-bold text-slate-400 mb-4">VOTE TALLY (1 VOTE PER CONTESTANT)</h3>
                <div className="grid grid-cols-2 gap-3">
                  {weeklyData.votes.map((v, i) => (
                    <div key={i} className="bg-slate-900 p-3 rounded-xl border border-slate-800 text-xs flex justify-between">
                      <span className="text-slate-400">{v.voter} voted:</span>
                      <span className="font-bold text-pink-400">{v.target}</span>
                    </div>
                  ))}
                </div>
              </div>
            </div>
          )}

          {/* PAGE 5: VOTING BREAKDOWN MATRIX */}
          {currentPage === 5 && (
            <div className="space-y-6">
              <h2 className="text-xl font-black text-pink-400 uppercase tracking-wider flex items-center gap-2">
                <Layers className="w-5 h-5" /> Page 5: Voting Breakdown & Arena Nominees
              </h2>

              <div className="bg-slate-950 p-6 rounded-2xl border border-slate-800">
                <h3 className="text-sm font-bold text-slate-300 mb-4">PLAYERS SENT TO ELIMINATION ARENA</h3>
                <div className="flex justify-center gap-6">
                  {weeklyData.nominatedDuelists.map(duelist => (
                    <div key={duelist.id} className="bg-slate-900 p-4 rounded-2xl border border-pink-500/40 text-center">
                      <img src={duelist.photo} className="w-16 h-16 rounded-full mx-auto mb-2" />
                      <span className="font-bold text-sm block">{duelist.name}</span>
                      <span className="text-xs text-pink-400 font-semibold">
                        {weeklyData.voteCounts[duelist.id]} Votes Received
                      </span>
                    </div>
                  ))}
                </div>
              </div>
            </div>
          )}

          {/* PAGE 6: ELIMINATION DUEL ARENA */}
          {currentPage === 6 && (
            <div className="space-y-6">
              <h2 className="text-xl font-black text-pink-400 uppercase tracking-wider flex items-center gap-2">
                <Shield className="w-5 h-5" /> Page 6: Elimination Arena Duel
              </h2>

              {weeklyData.isNonElim ? (
                <div className="bg-emerald-950/40 border border-emerald-800 p-6 rounded-2xl text-center">
                  <h3 className="text-lg font-black text-emerald-400 mb-2">SURPRISE NON-ELIMINATION WEEK!</h3>
                  <p className="text-xs text-slate-300">Both duelists fought bravely and both re-enter the house!</p>
                </div>
              ) : (
                <div className="bg-slate-950 p-6 rounded-2xl border border-slate-800 text-center">
                  <h3 className="text-xs font-bold text-slate-400 mb-4">ARENA SURVIVOR</h3>
                  {weeklyData.duelWinners.map(w => (
                    <div key={w.id} className="inline-flex items-center gap-3 bg-emerald-950/40 border border-emerald-500/40 px-6 py-3 rounded-full">
                      <img src={w.photo} className="w-10 h-10 rounded-full" />
                      <span className="text-base font-black text-emerald-400">{w.name} SURVIVES</span>
                    </div>
                  ))}
                </div>
              )}
            </div>
          )}

          {/* PAGE 7: ELIMINATION RESULT SUMMARY */}
          {currentPage === 7 && (
            <div className="space-y-6">
              <h2 className="text-xl font-black text-pink-400 uppercase tracking-wider flex items-center gap-2">
                <Skull className="w-5 h-5" /> Page 7: Elimination Summary
              </h2>

              <div className="bg-slate-950 p-6 rounded-2xl border border-slate-800 text-center">
                {weeklyData.eliminatedThisWeek.length > 0 ? (
                  <div className="space-y-4">
                    <h3 className="text-xs font-bold text-slate-400">ELIMINATED THIS WEEK</h3>
                    <div className="flex justify-center gap-4">
                      {weeklyData.eliminatedThisWeek.map(p => (
                        <div key={p.id} className="bg-slate-900 p-4 rounded-2xl border border-slate-800">
                          <img src={p.photo} className="w-14 h-14 rounded-full grayscale mx-auto mb-2" />
                          <span className="font-bold text-sm text-slate-400 block">{p.name}</span>
                          <span className="text-xs text-red-400 font-semibold">Eliminated</span>
                        </div>
                      ))}
                    </div>
                  </div>
                ) : (
                  <div className="text-slate-400 text-sm">No contestants were eliminated this week!</div>
                )}
              </div>
            </div>
          )}

          {/* PAGE 8: PROGRESS TRACK TABLE (WIKIA STYLE DYNAMIC TRACK) */}
          {currentPage === 8 && (
            <div className="space-y-6">
              <h2 className="text-xl font-black text-pink-400 uppercase tracking-wider flex items-center gap-2">
                <TrendingUp className="w-5 h-5" /> Page 8: Dynamic Season Track
              </h2>

              <div className="overflow-x-auto bg-slate-950 rounded-2xl border border-slate-800 p-4">
                <table className="w-full text-left border-collapse text-xs">
                  <thead>
                    <tr className="bg-rose-950 text-rose-200 uppercase font-black tracking-wider border-b border-rose-900">
                      <th className="p-2 border-r border-rose-900">Cast</th>
                      {Array.from({ length: currentWeek }, (_, i) => (
                        <th key={i} className="p-2 border-r border-rose-900 text-center">W{i + 1}</th>
                      ))}
                    </tr>
                  </thead>
                  <tbody>
                    {getSortedCastForTrack().map((player) => (
                      <tr key={player.id} className="border-b border-slate-900 hover:bg-slate-900/50">
                        {/* Player Column */}
                        <td className="p-2 flex items-center gap-2 border-r border-slate-800 font-bold">
                          <img src={player.photo} className="w-6 h-6 rounded-full" />
                          <span className={player.status === "ELIMINATED" ? "line-through text-slate-500" : ""}>
                            {player.name}
                          </span>
                        </td>

                        {/* Weekly Status Cells */}
                        {Array.from({ length: currentWeek }, (_, i) => {
                          const wNum = i + 1;
                          const weekHist = weeklyHistory.find(h => h.week === wNum);

                          if (!weekHist) return <td key={i} className="p-2 border-r border-slate-900 bg-slate-950"></td>;

                          // If player was eliminated BEFORE this week, render empty blank cell
                          if (player.eliminatedWeek && player.eliminatedWeek < wNum) {
                            return <td key={i} className="p-2 border-r border-slate-900 bg-slate-950"></td>;
                          }

                          // Check weekly status
                          const isElim = weekHist.eliminatedThisWeek?.some(e => e.id === player.id);
                          const isDuelist = weekHist.nominatedDuelists?.some(d => d.id === player.id);
                          const isCommander = weekHist.commander?.id === player.id;
                          const recVotes = weekHist.voteCounts?.[player.id] > 0;

                          let bgClass = "bg-slate-100 text-slate-900"; // SAFE (White)
                          let label = "SAFE";

                          if (isElim) {
                            bgClass = "bg-slate-600 text-slate-100 font-bold";
                            label = "ELIM";
                          } else if (isDuelist) {
                            bgClass = "bg-rose-200 text-rose-950 font-bold"; // RING/DUEL (Light Red)
                            label = "RING";
                          } else if (isCommander) {
                            bgClass = "bg-amber-200 text-amber-950 font-bold"; // IMMUNITY (Light Yellow)
                            label = "IMM";
                          } else if (recVotes) {
                            bgClass = "bg-purple-200 text-purple-950 font-bold"; // VOTED (Light Purple)
                            label = "VOTED";
                          }

                          return (
                            <td key={i} className={`p-2 border-r border-slate-900 text-center font-bold text-[10px] ${bgClass}`}>
                              {label}
                            </td>
                          );
                        })}
                      </tr>
                    ))}
                  </tbody>
                </table>

                {/* COLOR KEY LEGEND */}
                <div className="flex flex-wrap gap-3 mt-4 pt-4 border-t border-slate-900 text-[11px] font-semibold">
                  <span className="flex items-center gap-1.5"><span className="w-3 h-3 bg-amber-200 rounded"></span> Immunity</span>
                  <span className="flex items-center gap-1.5"><span className="w-3 h-3 bg-slate-100 rounded"></span> Safe</span>
                  <span className="flex items-center gap-1.5"><span className="w-3 h-3 bg-purple-200 rounded"></span> Voted (Saved)</span>
                  <span className="flex items-center gap-1.5"><span className="w-3 h-3 bg-rose-200 rounded"></span> Arena Duel</span>
                  <span className="flex items-center gap-1.5"><span className="w-3 h-3 bg-slate-600 rounded"></span> Eliminated</span>
                </div>
              </div>
            </div>
          )}

          {/* PAGE NAVIGATION BUTTON */}
          <button 
            onClick={handleNextPage}
            className="mt-8 w-full bg-gradient-to-r from-pink-500 to-purple-600 hover:from-pink-600 hover:to-purple-700 py-3.5 rounded-2xl font-bold flex items-center justify-center gap-2 shadow-lg shadow-pink-500/20 transition-all text-sm tracking-wider uppercase"
          >
            {currentPage === 8 ? "Advance to Next Week" : "Continue to Next Page"} <ArrowRight className="w-4 h-4" />
          </button>
        </div>
      )}

      {/* ==================== PHASE 4: FINALE & CROWNING SCREEN ==================== */}
      {phase === 'FINALE' && weeklyData && (
        <div className="max-w-4xl mx-auto bg-slate-900 border border-slate-800 rounded-3xl p-8 shadow-2xl text-center space-y-8">
          <Sparkles className="w-16 h-16 text-amber-400 mx-auto animate-bounce" />

          <div>
            <span className="text-xs font-bold text-pink-400 tracking-widest uppercase">SEASON FINALE COMPLETE</span>
            <h2 className="text-3xl font-black text-amber-400 mt-1">CROWNING THE WINNER</h2>
          </div>

          {/* WINNER GOLD CARD */}
          <div className="bg-gradient-to-b from-amber-400 to-amber-600 p-1 rounded-3xl max-w-sm mx-auto shadow-2xl shadow-amber-500/20">
            <div className="bg-slate-950 rounded-[23px] p-6 text-center">
              <img src={weeklyData.winner.photo} className="w-24 h-24 rounded-full mx-auto mb-3 border-4 border-amber-400" />
              <h3 className="text-2xl font-black text-amber-400">{weeklyData.winner.name}</h3>
              <span className="inline-block bg-amber-400 text-amber-950 font-black text-xs px-3 py-1 rounded-full mt-2 uppercase">
                Sole Survivor & Champion
              </span>
            </div>
          </div>

          {/* FINAL WIKIA PROGRESS TRACK TABLE WITH GOLD & SILVER */}
          <div className="overflow-x-auto bg-slate-950 rounded-2xl border border-slate-800 p-4 text-left">
            <h4 className="text-xs font-bold text-slate-400 mb-3">FINAL SEASON PROGRESSION TRACK</h4>
            <table className="w-full text-left border-collapse text-xs">
              <thead>
                <tr className="bg-rose-950 text-rose-200 uppercase font-black tracking-wider border-b border-rose-900">
                  <th className="p-2 border-r border-rose-900">Cast</th>
                  {Array.from({ length: currentWeek }, (_, i) => (
                    <th key={i} className="p-2 border-r border-rose-900 text-center">W{i + 1}</th>
                  ))}
                  <th className="p-2 text-center bg-amber-950 text-amber-300">Finale</th>
                </tr>
              </thead>
              <tbody>
                {getSortedCastForTrack().map((player) => {
                  const isWinner = player.placement === "WINNER";
                  const isRunnerUp = player.placement?.includes("RUNNER UP");

                  return (
                    <tr key={player.id} className="border-b border-slate-900">
                      {/* Contestant Name Column */}
                      <td className="p-2 flex items-center gap-2 border-r border-slate-800 font-bold">
                        <img src={player.photo} className="w-6 h-6 rounded-full" />
                        <span>{player.name}</span>
                      </td>

                      {/* Weekly Cell History */}
                      {Array.from({ length: currentWeek }, (_, i) => {
                        const wNum = i + 1;
                        const weekHist = weeklyHistory.find(h => h.week === wNum);

                        if (!weekHist || (player.eliminatedWeek && player.eliminatedWeek < wNum)) {
                          return <td key={i} className="p-2 border-r border-slate-900 bg-slate-950"></td>;
                        }

                        const isElim = weekHist.eliminatedThisWeek?.some(e => e.id === player.id);
                        const isDuelist = weekHist.nominatedDuelists?.some(d => d.id === player.id);
                        const recVotes = weekHist.voteCounts?.[player.id] > 0;

                        let bgClass = "bg-slate-100 text-slate-900";
                        let label = "SAFE";

                        if (isElim) {
                          bgClass = "bg-slate-600 text-slate-100 font-bold";
                          label = "ELIM";
                        } else if (isDuelist) {
                          bgClass = "bg-rose-200 text-rose-950 font-bold";
                          label = "RING";
                        } else if (recVotes) {
                          bgClass = "bg-purple-200 text-purple-950 font-bold";
                          label = "VOTED";
                        }

                        return (
                          <td key={i} className={`p-2 border-r border-slate-900 text-center font-bold text-[10px] ${bgClass}`}>
                            {label}
                          </td>
                        );
                      })}

                      {/* FINALE RESULT CELL */}
                      <td className={`p-2 text-center font-black text-[10px] ${
                        isWinner 
                          ? 'bg-amber-300 text-amber-950 border-2 border-amber-400 shadow-inner' 
                          : isRunnerUp 
                          ? 'bg-slate-300 text-slate-900 border-2 border-slate-400' 
                          : 'bg-slate-900 text-slate-500'
                      }`}>
                        {player.status === "ACTIVE" ? player.placement : (
                          weeklyData.individualJuryVotes?.find(j => j.jurorName === player.name) 
                            ? `Voted ${weeklyData.individualJuryVotes.find(j => j.jurorName === player.name).votedFor}`
                            : ""
                        )}
                      </td>
                    </tr>
                  );
                })}
              </tbody>
            </table>
          </div>

          <button 
            onClick={() => {
              setCurrentWeek(1);
              setWeeklyHistory([]);
              setPhase('SETUP');
            }}
            className="bg-slate-800 hover:bg-slate-700 text-slate-200 px-6 py-3 rounded-xl font-bold inline-flex items-center gap-2"
          >
            <RotateCcw className="w-4 h-4" /> Start New Season
          </button>
        </div>
      )}
    </div>
  );
}

```
