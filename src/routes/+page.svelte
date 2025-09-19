<script lang="ts">
  import { onMount } from 'svelte';
  import { GoogleGenerativeAI } from '@google/generative-ai';
 
  interface Player {
    name: string;
    role: 'imposter' | 'civilian' | 'dumb';
    word: string;
  }
  interface GameData {
    players: Player[];
    civilianWord: string;
    imposterWord: string;
  }
  interface WordPair {
    civilian: string;
    imposter: string;
  }
  let gameState: 'home' | 'setup' | 'game' | 'complete' | 'reveal' = 'home';
  let numPlayers: number = 4;
  let playerNames: string[] = [''];
  let currentPlayerIndex: number = 0;
  let gameData: GameData | null = null;
  let showRoleInitially: boolean = false;
  let wordRevealed: boolean = false;
  let isGeneratingWords: boolean = false;
  let selectedLanguage: 'nepali' | 'english' = 'english';
  let includeDumbRole: boolean = false;
  const PlayIcon = `<svg viewBox="0 0 24 24" fill="currentColor">
    <path d="M8 5v14l11-7z"/>
  </svg>`;
  const EyeIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <path d="M1 12s4-8 11-8 11 8 11 8-4 8-11 8-11-8-11-8z"/>
    <circle cx="12" cy="12" r="3"/>
  </svg>`;
  const EyeOffIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <path d="M17.94 17.94A10.07 10.07 0 0 1 12 20c-7 0-11-8-11-8a18.45 18.45 0 0 1 5.06-5.94M9.9 4.24A9.12 9.12 0 0 1 12 4c7 0 11 8 11 8a18.5 18.5 0 0 1-2.16 3.19m-6.72-1.07a3 3 0 1 1-4.24-4.24"/>
    <path d="m1 1 22 22"/>
  </svg>`;
  const ArrowRightIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <path d="M5 12h14"/>
    <path d="m12 5 7 7-7 7"/>
  </svg>`;
  const SearchIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <circle cx="11" cy="11" r="8"/>
    <path d="m21 21-4.35-4.35"/>
  </svg>`;
  const RefreshIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <path d="M3 12a9 9 0 0 1 9-9 9.75 9.75 0 0 1 6.74 2.74L21 8"/>
    <path d="M21 3v5h-5"/>
    <path d="M21 12a9 9 0 0 1-9 9 9.75 9.75 0 0 1-6.74-2.74L3 16"/>
    <path d="M3 21v-5h5"/>
  </svg>`;
  const GlobeIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <circle cx="12" cy="12" r="10"/>
    <line x1="2" x2="22" y1="12" y2="12"/>
    <path d="M12 2a15.3 15.3 0 0 1 4 10 15.3 15.3 0 0 1-4 10 15.3 15.3 0 0 1-4-10 15.3 15.3 0 0 1 4-10z"/>
  </svg>`;
  const UserIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <path d="M20 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2"/>
    <circle cx="12" cy="7" r="4"/>
  </svg>`;

  const StarIcon = `<svg viewBox="0 0 24 24" fill="currentColor">
    <path d="M12 2l3.09 6.26L22 9.27l-5 4.87 1.18 6.88L12 17.77l-6.18 3.25L7 14.14 2 9.27l6.91-1.01L12 2z"/>
  </svg>`;

  const SparklesIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <path d="M9.937 15.5A2 2 0 0 0 8.5 14.063l-6.135-1.582a.5.5 0 0 1 0-.962L8.5 9.936A2 2 0 0 0 9.937 8.5l1.582-6.135a.5.5 0 0 1 .963 0L14.063 8.5A2 2 0 0 0 15.5 9.937l6.135 1.582a.5.5 0 0 1 0 .963L15.5 14.063a2 2 0 0 0-1.437 1.437l-1.582 6.135a.5.5 0 0 1-.963 0L9.937 15.5z"/>
  </svg>`;
  
  // Initialize player names array when numPlayers changes
  $: {
    if (numPlayers > 0) {
      playerNames = Array(numPlayers).fill('').map((_, i: number) => playerNames[i] || '');
    }
  }
  async function generateWithGemini(prompt: string): Promise<WordPair> {
    try {
      const genAI = new GoogleGenerativeAI(
        import.meta.env.VITE_REACT_APP_GEMINI_API_KEY || 'your-api-key-here'
      );
      const model = genAI.getGenerativeModel({ model: "gemini-2.5-flash" });
      const result = await model.generateContent(prompt);
      const response = await result.response;
      const text = await response.text();
      return parseJSON(text);
    } catch (error) {
      console.error('Error generating with Gemini:', error);
      // Fallback to predefined words if API fails
      return selectedLanguage === 'nepali'
        ? { civilian: 'कुकुर', imposter: 'बिरालो' }
        : { civilian: 'Dog', imposter: 'Cat' };
    }
  }
  function parseJSON(text: string): WordPair {
    try {
      const jsonMatch = text.match(/\{[\s\S]*\}/);
      if (jsonMatch) {
        return JSON.parse(jsonMatch[0]);
      }
     
      const lines = text.split('\n').filter((line: string) => line.trim());
      const civilianMatch = lines.find((line: string) => line.toLowerCase().includes('civilian') || line.includes('सिभिलियन'));
      const imposterMatch = lines.find((line: string) => line.toLowerCase().includes('imposter') || line.includes('इम्पोस्टर'));
     
      if (civilianMatch && imposterMatch) {
        return {
          civilian: civilianMatch.split(':').pop()?.trim().replace(/['"]/g, '') || (selectedLanguage === 'nepali' ? 'कुकुर' : 'Dog'),
          imposter: imposterMatch.split(':').pop()?.trim().replace(/['"]/g, '') || (selectedLanguage === 'nepali' ? 'बिरालो' : 'Cat')
        };
      }
     
      return selectedLanguage === 'nepali'
        ? { civilian: 'कुकुर', imposter: 'बिरालो' }
        : { civilian: 'Dog', imposter: 'Cat' };
    } catch (error) {
      console.error('Error parsing response:', error);
      return selectedLanguage === 'nepali'
        ? { civilian: 'कुकुर', imposter: 'बिरालो' }
        : { civilian: 'Dog', imposter: 'Cat' };
    }
  }
  async function startGame(): Promise<void> {
    if (numPlayers < 3) {
      alert('At least 3 players are required!');
      return;
    }
   
    isGeneratingWords = true;
   
    const prompt = selectedLanguage === 'nepali'
      ? `Generate two similar meaning but different Nepali words for a "Who is the Imposter" game.
    One word for civilians and one slightly different word for the imposter.
    The words should be related but distinct enough to create confusion.
   
    Respond in this exact JSON format:
    {
      "civilian": "nepali_word_here",
      "imposter": "similar_but_different_nepali_word_here"
    }
   
    Example categories: animals, fruits, objects, places, food, etc.
    Make sure both words are simple and commonly known Nepali words.`
      : `Generate two similar meaning but different English words for a "Who is the Imposter" game.
    One word for civilians and one slightly different word for the imposter.
    The words should be related but distinct enough to create confusion.
   
    Respond in this exact JSON format:
    {
      "civilian": "english_word_here",
      "imposter": "similar_but_different_english_word_here"
    }
   
    Example categories: animals, fruits, objects, places, food, etc.
    Make sure both words are simple and commonly known English words.`;
   
    const wordPair = await generateWithGemini(prompt);
   
    // Randomly select imposter (never first player if DUMB role is included)
    const imposterIndex = Math.floor(Math.random() * numPlayers);
   
    // If DUMB role is included, randomly select someone who isn't the imposter (and not first player)
    let dumbIndex = -1;
    if (includeDumbRole) {
      const availableIndices = Array.from({length: numPlayers}, (_, i) => i)
        .filter(i => i !== imposterIndex && i !== 0); // Exclude imposter and first player
      if (availableIndices.length > 0) {
        dumbIndex = availableIndices[Math.floor(Math.random() * availableIndices.length)];
      }
    }
   
    // Create game data
    const players: Player[] = playerNames.map((name, index) => {
      if (index === imposterIndex) {
        return {
          name: name || `Player ${index + 1}`,
          role: 'imposter',
          word: wordPair.imposter
        };
      } else if (index === dumbIndex) {
        return {
          name: name || `Player ${index + 1}`,
          role: 'dumb',
          word: '???'
        };
      } else {
        return {
          name: name || `Player ${index + 1}`,
          role: 'civilian',
          word: wordPair.civilian
        };
      }
    });
    gameData = {
      players,
      civilianWord: wordPair.civilian,
      imposterWord: wordPair.imposter
    };
    currentPlayerIndex = 0;
    wordRevealed = false;
    gameState = 'game';
    isGeneratingWords = false;
  }
  function nextPlayer(): void {
    wordRevealed = false;
    if (currentPlayerIndex < numPlayers - 1) {
      currentPlayerIndex++;
    } else {
      gameState = 'complete';
    }
  }
  function revealWord(): void {
    wordRevealed = true;
  }
  function resetGame(): void {
    gameState = 'home';
    currentPlayerIndex = 0;
    gameData = null;
    wordRevealed = false;
    playerNames = [''];
  }
  function showImposter(): void {
    gameState = 'reveal';
  }
  $: currentPlayer = gameData?.players[currentPlayerIndex];
  $: imposter = gameData?.players.find((p: Player) => p.role === 'imposter');
  $: dumbPlayer = gameData?.players.find((p: Player) => p.role === 'dumb');
</script>

<div class="min-h-screen bg-gradient-to-br from-purple-100 via-pink-50 to-indigo-100">
 
  <!-- Home Screen -->
  {#if gameState === 'home'}
    <div class="container mx-auto px-4 py-8 md:py-16">
      <div class="max-w-md mx-auto">
       
        <!-- Header -->
        <div class="text-center mb-12">
          <div class="w-16 h-16 mx-auto mb-4 text-purple-500">{@html SparklesIcon}</div>
          <h1 class="text-4xl md:text-5xl font-black bg-gradient-to-r from-purple-600 to-pink-600 bg-clip-text text-transparent mb-3">
            Who's the Spy?
          </h1>
          <p class="text-slate-600">Find the imposter in this fun party game!</p>
        </div>
        
        <!-- Game Settings -->
        <div class="bg-white/80 backdrop-blur-sm rounded-3xl shadow-2xl border border-white/20 p-8 space-y-6">
         
          <!-- Language Selection -->
          <div>
            <label class="block text-sm font-bold text-slate-700 mb-3">🌍 Language</label>
            <div class="grid grid-cols-2 gap-2 p-1 bg-slate-100/70 rounded-2xl">
              <button
                type="button"
                on:click={() => selectedLanguage = 'english'}
                class="flex hover:cursor-pointer items-center justify-center py-3 px-4 rounded-xl text-sm font-semibold transition-all duration-200 hover:scale-105
                  {selectedLanguage === 'english' ? 'bg-white shadow-lg text-purple-700 border-2 border-purple-200' : 'text-slate-600 hover:text-slate-800'}"
              >
                English
              </button>
              <button
                type="button"
                on:click={() => selectedLanguage = 'nepali'}
                class="flex items-center hover:cursor-pointer justify-center py-3 px-4 rounded-xl text-sm font-semibold transition-all duration-200 hover:scale-105
                  {selectedLanguage === 'nepali' ? 'bg-white shadow-lg text-purple-700 border-2 border-purple-200' : 'text-slate-600 hover:text-slate-800'}"
              >
                नेपाली
              </button>
            </div>
          </div>
          
          <!-- Number of Players -->
          <div>
            <label class="block text-sm font-bold text-slate-700 mb-3">👥 Players</label>
            <input
              type="number"
              min="3"
              max="12"
              bind:value={numPlayers}
              class="w-full p-4 text-center text-2xl font-black border-3 border-purple-200 rounded-2xl focus:border-purple-500 focus:outline-none transition-all bg-purple-50/50 hover:bg-purple-50"
            />
          </div>
          
          <!-- Game Options -->
          <div class="space-y-3">
            <div class="flex items-center justify-between p-4 bg-purple-50/50 rounded-2xl border border-purple-100">
              <div>
                <div class="font-bold text-slate-800 text-sm">🎭 Show roles</div>
                <div class="text-xs text-slate-500">Reveal player roles immediately</div>
              </div>
              <button
                type="button"
                on:click={() => showRoleInitially = !showRoleInitially}
                class="relative hover:cursor-pointer w-14 h-8 rounded-full transition-all duration-300 focus:outline-none hover:scale-110
                  {showRoleInitially ? 'bg-gradient-to-r from-purple-500 to-pink-500 shadow-lg' : 'bg-slate-300'}"
              >
                <div class="absolute w-6 h-6 bg-white rounded-full shadow-sm transition-transform duration-300 top-1 flex items-center justify-center
                  {showRoleInitially ? 'translate-x-7' : 'translate-x-1'}">
                  {#if showRoleInitially}
                    <div class="w-3 h-3 text-purple-500">{@html StarIcon}</div>
                  {/if}
                </div>
              </button>
            </div>
            
            <div class="flex items-center justify-between p-4 bg-amber-50/50 rounded-2xl border border-amber-100">
              <div>
                <div class="font-bold text-slate-800 text-sm">🤔 Add Mystery Player</div>
                <div class="text-xs text-slate-500">Someone gets no word at all</div>
              </div>
              <button
                type="button"
                on:click={() => includeDumbRole = !includeDumbRole}
                class="relative w-14 hover:cursor-pointer h-8 rounded-full transition-all duration-300 focus:outline-none hover:scale-110
                  {includeDumbRole ? 'bg-gradient-to-r from-amber-400 to-orange-500 shadow-lg' : 'bg-slate-300'}"
              >
                <div class="absolute w-6 h-6 bg-white rounded-full shadow-sm transition-transform duration-300 top-1 flex items-center justify-center
                  {includeDumbRole ? 'translate-x-7' : 'translate-x-1'}">
                  {#if includeDumbRole}
                    <div class="text-amber-500 text-xs font-bold">?</div>
                  {/if}
                </div>
              </button>
            </div>
          </div>
          
          <!-- Start Button -->
          <button
            on:click={() => gameState = 'setup'}
            class="w-full bg-gradient-to-r hover:cursor-pointer from-purple-500 via-pink-500 to-indigo-500 text-white font-black py-5 px-8 rounded-2xl hover:from-purple-600 hover:via-pink-600 hover:to-indigo-600 transition-all duration-300 shadow-2xl hover:shadow-purple-500/25 flex items-center justify-center space-x-3 hover:scale-105 transform"
          >
            <div class="w-6 h-6">{@html PlayIcon}</div>
            <span class="text-lg">LET'S PLAY!</span>
          </button>
        </div>
      </div>
    </div>
  {/if}
  
  <!-- Setup Screen -->
  {#if gameState === 'setup'}
    <div class="container mx-auto px-4 py-8 md:py-16">
      <div class="max-w-md mx-auto">
       
        <div class="text-center mb-8">
          <div class="w-12 h-12 mx-auto mb-4 text-purple-500">{@html UserIcon}</div>
          <h2 class="text-3xl font-black bg-gradient-to-r from-purple-600 to-pink-600 bg-clip-text text-transparent mb-2">Player Names</h2>
          <p class="text-slate-600">Who's joining the fun?</p>
        </div>
        
        <div class="bg-white/80 backdrop-blur-sm rounded-3xl shadow-2xl border border-white/20 p-8">
          <div class="space-y-4 max-h-96 overflow-y-auto mb-8">
            {#each Array(numPlayers) as _, index}
              <div class="group">
                <label class="block text-xs font-bold text-slate-600 mb-2 group-focus-within:text-purple-600 transition-colors">
                  🎮 Player {index + 1}
                </label>
                <input
                  type="text"
                  bind:value={playerNames[index]}
                  placeholder="Enter name..."
                  class="w-full p-4 border-2 border-purple-100 rounded-xl focus:border-purple-500 focus:outline-none transition-all bg-purple-50/30 hover:bg-purple-50/50 font-medium placeholder:text-slate-400"
                />
              </div>
            {/each}
          </div>
          
          <div class="flex space-x-4">
            <button
              on:click={() => gameState = 'home'}
              class="flex-1 hover:cursor-pointer bg-slate-100 hover:bg-slate-200 text-slate-700 font-bold py-4 px-4 rounded-xl transition-all duration-200 hover:scale-105"
            >
              ← Back
            </button>
            <button
              on:click={startGame}
              disabled={isGeneratingWords}
              class="flex-2 bg-gradient-to-r hover:cursor-pointer from-purple-500 to-pink-500 text-white font-bold py-4 px-6 rounded-xl hover:from-purple-600 hover:to-pink-600 transition-all duration-200 disabled:opacity-50 disabled:cursor-not-allowed hover:scale-105 disabled:hover:scale-100"
            >
              {#if isGeneratingWords}
                <div class="flex items-center justify-center space-x-2">
                  <div class="animate-spin rounded-full h-5 w-5 border-2 border-white border-t-transparent"></div>
                  <span>Creating Magic...</span>
                </div>
              {:else}
                🚀 Start Game
              {/if}
            </button>
          </div>
        </div>
      </div>
    </div>
  {/if}
  
  <!-- Game Screen -->
  {#if gameState === 'game'}
    <div class="container mx-auto px-4 py-8 md:py-16">
      <div class="max-w-md mx-auto">
       
        <div class="bg-white/80 backdrop-blur-sm rounded-3xl shadow-2xl border border-white/20 p-8">
          <!-- Player Info -->
          <div class="text-center mb-8">
            <div class="w-20 h-20 bg-gradient-to-br from-purple-400 to-pink-400 rounded-full mx-auto mb-4 flex items-center justify-center text-2xl font-black text-white shadow-lg">
              {currentPlayerIndex + 1}
            </div>
            <h2 class="text-3xl font-black text-slate-800 mb-2">
              {currentPlayer?.name || `Player ${currentPlayerIndex + 1}`}
            </h2>
            <div class="inline-flex items-center bg-purple-100 text-purple-700 px-4 py-2 rounded-full text-sm font-bold">
              Player {currentPlayerIndex + 1} of {numPlayers}
            </div>
          </div>
          
          {#if !wordRevealed}
            <!-- Word Hidden State -->
            <div class="text-center space-y-6">
              <div class="bg-slate-50/70 p-10 rounded-3xl border-2 border-dashed border-slate-300">
                <div class="w-20 h-20 mx-auto mb-6 text-slate-400">{@html EyeOffIcon}</div>
                <p class="text-slate-500 font-medium">Tap to see your secret word</p>
              </div>
              <button
                on:click={revealWord}
                class="w-full bg-gradient-to-r hover:cursor-pointer from-purple-500 to-pink-500 text-white font-black py-5 px-6 rounded-2xl hover:from-purple-600 hover:to-pink-600 transition-all duration-200 shadow-xl hover:shadow-purple-500/25 flex items-center justify-center space-x-3 hover:scale-105 transform"
              >
                <div class="w-6 h-6">{@html EyeIcon}</div>
                <span class="text-lg">REVEAL MY WORD</span>
              </button>
            </div>
          {:else}
            <!-- Word Revealed State -->
            <div class="space-y-6">
              {#if showRoleInitially}
                <!-- Show Role and Word -->
                <div class="p-8 rounded-3xl text-center border-3 transform hover:scale-105 transition-all duration-200
                  {currentPlayer?.role === 'imposter' ? 'bg-gradient-to-br from-red-50 to-pink-50 border-red-200 shadow-red-500/20' :
                   currentPlayer?.role === 'dumb' ? 'bg-gradient-to-br from-amber-50 to-orange-50 border-amber-200 shadow-amber-500/20' :
                   'bg-gradient-to-br from-emerald-50 to-green-50 border-emerald-200 shadow-emerald-500/20'} shadow-2xl">
                  
                  <div class="text-4xl mb-4">
                    {currentPlayer?.role === 'imposter' ? '🕵️' :
                     currentPlayer?.role === 'dumb' ? '🤔' : '👥'}
                  </div>
                  
                  <h3 class="text-xl font-black mb-4
                    {currentPlayer?.role === 'imposter' ? 'text-red-600' :
                     currentPlayer?.role === 'dumb' ? 'text-amber-600' :
                     'text-emerald-600'}">
                    {currentPlayer?.role === 'imposter' ? 'You are the SPY!' :
                     currentPlayer?.role === 'dumb' ? 'You are the MYSTERY!' :
                     'You are a CIVILIAN'}
                  </h3>
                  
                  <div class="text-3xl font-black text-slate-900 py-6 px-4 bg-white/70 rounded-2xl shadow-lg border-2 border-white/50 backdrop-blur-sm">
                    {currentPlayer?.word}
                  </div>
                  
                  <p class="text-slate-600 text-sm mt-4 font-medium">
                    {currentPlayer?.role === 'imposter' ? 'Your word is different from others!' :
                     currentPlayer?.role === 'dumb' ? 'Figure out what everyone else has!' :
                     'Remember this word for discussion!'}
                  </p>
                </div>
              {:else}
                <!-- Show Only Word -->
                <div class="p-8 rounded-3xl text-center border-3 border-purple-200 bg-gradient-to-br from-purple-50 to-pink-50 shadow-2xl shadow-purple-500/20 transform hover:scale-105 transition-all duration-200">
                  <div class="text-4xl mb-4">🎭</div>
                  <h3 class="text-xl font-black mb-4 text-purple-700">Your Secret Word</h3>
                  <div class="text-3xl font-black text-slate-900 py-6 px-4 bg-white/70 rounded-2xl shadow-lg border-2 border-white/50 backdrop-blur-sm">
                    {currentPlayer?.word}
                  </div>
                  <p class="text-slate-600 text-sm mt-4 font-medium">
                    {currentPlayer?.role === 'dumb' ? 'Hmm... what could this mean?' : 'Keep this secret until the discussion!'}
                  </p>
                </div>
              {/if}
              
              <button
                on:click={nextPlayer}
                class="w-full bg-gradient-to-r hover:cursor-pointer from-slate-500 to-slate-600 text-white font-black py-5 px-6 rounded-2xl hover:from-slate-600 hover:to-slate-700 transition-all duration-200 flex items-center justify-center space-x-3 hover:scale-105 transform shadow-xl"
              >
                <span class="text-lg">NEXT PLAYER</span>
                <div class="w-6 h-6">{@html ArrowRightIcon}</div>
              </button>
            </div>
          {/if}
        </div>
      </div>
    </div>
  {/if}
  
  <!-- Complete Screen -->
  {#if gameState === 'complete'}
    <div class="container mx-auto px-4 py-8 md:py-16">
      <div class="max-w-md mx-auto">
       
        <div class="bg-white/80 backdrop-blur-sm rounded-3xl shadow-2xl border border-white/20 p-8">
          <div class="text-center mb-8">
            <div class="w-24 h-24 bg-gradient-to-br from-purple-500 to-pink-500 rounded-3xl mx-auto mb-6 flex items-center justify-center text-white text-4xl shadow-xl">
              🕵️
            </div>
            <h2 class="text-3xl font-black bg-gradient-to-r from-purple-600 to-pink-600 bg-clip-text text-transparent mb-3">Ready to Hunt!</h2>
            <p class="text-slate-600 font-medium">Time to discuss and find the spy{includeDumbRole ? ' and mystery player' : ''}!</p>
          </div>
          
          <div class="space-y-4">
            <button
              on:click={showImposter}
              class="w-full bg-gradient-to-r hover:cursor-pointer from-red-500 to-pink-500 text-white font-black py-5 px-6 rounded-2xl hover:from-red-600 hover:to-pink-600 transition-all duration-200 shadow-xl hover:shadow-red-500/25 flex items-center justify-center space-x-3 hover:scale-105 transform"
            >
              <div class="w-6 h-6">{@html SearchIcon}</div>
              <span class="text-lg">🔍 REVEAL RESULTS</span>
            </button>
            <button
              on:click={resetGame}
              class="w-full bg-slate-100 hover:cursor-pointer hover:bg-slate-200 text-slate-700 font-black py-5 px-6 rounded-2xl transition-all duration-200 flex items-center justify-center space-x-3 hover:scale-105 transform"
            >
              <div class="w-6 h-6">{@html RefreshIcon}</div>
              <span class="text-lg">🎮 NEW GAME</span>
            </button>
          </div>
        </div>
      </div>
    </div>
  {/if}
  
  <!-- Reveal Screen -->
  {#if gameState === 'reveal'}
    <div class="container mx-auto px-4 py-8 md:py-16">
      <div class="max-w-md mx-auto">
       
        <div class="bg-white/80 backdrop-blur-sm rounded-3xl shadow-2xl border border-white/20 p-8">
          <div class="text-center mb-8">
            <div class="text-6xl mb-4">🎉</div>
            <h2 class="text-3xl font-black bg-gradient-to-r from-red-600 to-pink-600 bg-clip-text text-transparent mb-6">Game Results!</h2>
           
            <!-- Imposter Result -->
            <div class="bg-gradient-to-br from-red-50 to-pink-50 border-3 border-red-200 p-8 rounded-3xl mb-4 shadow-xl shadow-red-500/20 transform hover:scale-105 transition-all duration-200">
              <div class="text-4xl mb-3">🕵️</div>
              <h3 class="text-2xl font-black text-red-700 mb-2">{imposter?.name}</h3>
              <p class="text-red-600 font-bold text-lg">was the SPY!</p>
              <div class="mt-4 inline-block bg-red-100 text-red-700 px-4 py-2 rounded-full text-sm font-bold">
                Word: {gameData?.imposterWord}
              </div>
            </div>
            
            <!-- Dumb Player Result -->
            {#if dumbPlayer}
              <div class="bg-gradient-to-br from-amber-50 to-orange-50 border-3 border-amber-200 p-8 rounded-3xl mb-6 shadow-xl shadow-amber-500/20 transform hover:scale-105 transition-all duration-200">
                <div class="text-4xl mb-3">🤔</div>
                <h3 class="text-2xl font-black text-amber-700 mb-2">{dumbPlayer?.name}</h3>
                <p class="text-amber-600 font-bold text-lg">was the MYSTERY!</p>
                <div class="mt-4 inline-block bg-amber-100 text-amber-700 px-4 py-2 rounded-full text-sm font-bold">
                  Had no word!
                </div>
              </div>
            {/if}
            
            <!-- Civilian Word -->
            <div class="bg-gradient-to-br from-emerald-50 to-green-50 border-3 border-emerald-200 p-6 rounded-2xl shadow-xl shadow-emerald-500/20">
              <div class="text-2xl mb-2">👥</div>
              <p class="text-emerald-700 font-bold">
                <span class="text-sm">Civilian word:</span><br>
                <span class="text-2xl font-black">{gameData?.civilianWord}</span>
              </p>
            </div>
          </div>
         
          <button
            on:click={resetGame}
            class="w-full hover:cursor-pointer bg-gradient-to-r from-green-500 to-emerald-500 text-white font-black py-5 px-6 rounded-2xl hover:from-green-600 hover:to-emerald-600 transition-all duration-200 flex items-center justify-center space-x-3 hover:scale-105 transform shadow-xl"
          >
            <div class="w-6 h-6">{@html RefreshIcon}</div>
            <span class="text-lg">🎮 PLAY AGAIN</span>
          </button>
        </div>
      </div>
    </div>
  {/if}
</div>

<style>
  /* Custom scrollbar styling */
  :global(.overflow-y-auto::-webkit-scrollbar) {
    width: 6px;
  }
 
  :global(.overflow-y-auto::-webkit-scrollbar-track) {
    background: rgba(243, 244, 246, 0.5);
    border-radius: 10px;
  }
 
  :global(.overflow-y-auto::-webkit-scrollbar-thumb) {
    background: linear-gradient(to bottom, #a855f7, #ec4899);
    border-radius: 10px;
  }
 
  :global(.overflow-y-auto::-webkit-scrollbar-thumb:hover) {
    background: linear-gradient(to bottom, #9333ea, #db2777);
  }

  /* Add some fun animations */
  @keyframes float {
    0%, 100% { transform: translateY(0px); }
    50% { transform: translateY(-10px); }
  }
  
  :global(.float) {
    animation: float 3s ease-in-out infinite;
  }
</style>