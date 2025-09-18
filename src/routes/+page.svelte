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

  const EyeIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
    <path d="M1 12s4-8 11-8 11 8 11 8-4 8-11 8-11-8-11-8z"/>
    <circle cx="12" cy="12" r="3"/>
  </svg>`;

  const EyeOffIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
    <path d="M17.94 17.94A10.07 10.07 0 0 1 12 20c-7 0-11-8-11-8a18.45 18.45 0 0 1 5.06-5.94M9.9 4.24A9.12 9.12 0 0 1 12 4c7 0 11 8 11 8a18.5 18.5 0 0 1-2.16 3.19m-6.72-1.07a3 3 0 1 1-4.24-4.24"/>
    <path d="m1 1 22 22"/>
  </svg>`;

  const ArrowRightIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
    <path d="M5 12h14"/>
    <path d="m12 5 7 7-7 7"/>
  </svg>`;

  const SearchIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
    <circle cx="11" cy="11" r="8"/>
    <path d="m21 21-4.35-4.35"/>
  </svg>`;

  const RefreshIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
    <path d="M3 12a9 9 0 0 1 9-9 9.75 9.75 0 0 1 6.74 2.74L21 8"/>
    <path d="M21 3v5h-5"/>
    <path d="M21 12a9 9 0 0 1-9 9 9.75 9.75 0 0 1-6.74-2.74L3 16"/>
    <path d="M3 21v-5h5"/>
  </svg>`;

  const GlobeIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
    <circle cx="12" cy="12" r="10"/>
    <line x1="2" x2="22" y1="12" y2="12"/>
    <path d="M12 2a15.3 15.3 0 0 1 4 10 15.3 15.3 0 0 1-4 10 15.3 15.3 0 0 1-4-10 15.3 15.3 0 0 1 4-10z"/>
  </svg>`;

  const UserIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
    <path d="M20 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2"/>
    <circle cx="12" cy="7" r="4"/>
  </svg>`;

  // Initialize player names array when numPlayers changes
  $: {
    if (numPlayers > 0) {
      playerNames = Array(numPlayers).fill('').map((_, i: number) => playerNames[i] || `Player ${i + 1}`);
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
  }

  function showImposter(): void {
    gameState = 'reveal';
  }

  $: currentPlayer = gameData?.players[currentPlayerIndex];
  $: imposter = gameData?.players.find((p: Player) => p.role === 'imposter');
  $: dumbPlayer = gameData?.players.find((p: Player) => p.role === 'dumb');
</script>

<div class="min-h-screen bg-gradient-to-br from-slate-50 to-zinc-100">
  
  <!-- Home Screen -->
  {#if gameState === 'home'}
    <div class="container mx-auto px-4 py-8 md:py-16">
      <div class="max-w-md mx-auto">
        
        <!-- Header -->
        <div class="text-center mb-12">
          <h1 class="text-3xl md:text-4xl font-bold text-slate-800 mb-3">
            Who is the Imposter?
          </h1>
          <p class="text-slate-500">Find the odd one out in this mind game</p>
        </div>

        <!-- Game Settings -->
        <div class="bg-white rounded-3xl shadow-xl shadow-slate-200/50 p-8 space-y-8">
          
          <!-- Language Selection -->
          <div>
            <label class="block text-sm font-semibold text-slate-700 mb-3">Language</label>
            <div class="grid grid-cols-2 gap-2 p-1 bg-slate-100 rounded-xl">
              <button
                type="button"
                on:click={() => selectedLanguage = 'english'}
                class="flex hover:cursor-pointer items-center justify-center py-3 px-4 rounded-lg text-sm font-medium transition-all duration-200
                  {selectedLanguage === 'english' ? 'bg-white shadow-sm text-slate-900' : 'text-slate-600 hover:text-slate-800'}"
              >
                <div class="w-4 h-4 mr-2">{@html GlobeIcon}</div>
                English
              </button>
              <button
                type="button"
                on:click={() => selectedLanguage = 'nepali'}
                class="flex items-center hover:cursor-pointer justify-center py-3 px-4 rounded-lg text-sm font-medium transition-all duration-200
                  {selectedLanguage === 'nepali' ? 'bg-white shadow-sm text-slate-900' : 'text-slate-600 hover:text-slate-800'}"
              >
                <div class="w-4 h-4 mr-2">{@html GlobeIcon}</div>
                नेपाली
              </button>
            </div>
          </div>

          <!-- Number of Players -->
          <div>
            <label class="block text-sm font-semibold text-slate-700 mb-3">Number of Players</label>
            <input
              type="number"
              min="3"
              max="12"
              bind:value={numPlayers}
              class="w-full p-4 text-center text-xl font-bold border-2 border-slate-200 rounded-xl focus:border-indigo-500 focus:outline-none transition-colors bg-slate-50"
            />
          </div>

          <!-- Game Options -->
          <div class="space-y-4">
            <div class="flex items-center justify-between p-4 bg-slate-50 rounded-xl">
              <div>
                <div class="font-medium text-slate-800">Show roles to players</div>
                <div class="text-sm text-slate-500">Reveal if player is imposter/civilian</div>
              </div>
              <button
                type="button"
                on:click={() => showRoleInitially = !showRoleInitially}
                class="relative hover:cursor-pointer w-12 h-6 rounded-full transition-colors duration-200 focus:outline-none
                  {showRoleInitially ? 'bg-indigo-500' : 'bg-slate-300'}"
              >
                <div class="absolute w-5 h-5 bg-white rounded-full shadow-sm transition-transform duration-200 top-0.5
                  {showRoleInitially ? 'translate-x-6' : 'translate-x-0.5'}"></div>
              </button>
            </div>

            <div class="flex items-center justify-between p-4 bg-slate-50 rounded-xl">
              <div>
                <div class="font-medium text-slate-800">Add DUMB role</div>
                <div class="text-sm text-slate-500">Someone gets no word and must guess</div>
              </div>
              <button
                type="button"
                on:click={() => includeDumbRole = !includeDumbRole}
                class="relative w-12 hover:cursor-pointer h-6 rounded-full transition-colors duration-200 focus:outline-none
                  {includeDumbRole ? 'bg-amber-500' : 'bg-slate-300'}"
              >
                <div class="absolute w-5 h-5 bg-white rounded-full shadow-sm transition-transform duration-200 top-0.5
                  {includeDumbRole ? 'translate-x-6' : 'translate-x-0.5'}"></div>
              </button>
            </div>
          </div>

          <!-- Start Button -->
          <button
            on:click={() => gameState = 'setup'}
            class="w-full bg-gradient-to-r hover:cursor-pointer from-indigo-500 to-purple-600 text-white font-bold py-4 px-8 rounded-xl hover:from-indigo-600 hover:to-purple-700 transition-all duration-200 shadow-lg shadow-indigo-500/25 flex items-center justify-center space-x-3"
          >
            <div class="w-5 h-5">{@html PlayIcon}</div>
            <span>START GAME</span>
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
          <h2 class="text-2xl font-bold text-slate-800 mb-2">Player Names</h2>
          <p class="text-slate-500">Enter names for all players</p>
        </div>

        <div class="bg-white rounded-3xl shadow-xl shadow-slate-200/50 p-8">
          <div class="space-y-4 min-h-100 overflow-y-auto mb-8">
            {#each Array(numPlayers) as _, index}
              <div>
                <label class="block text-xs font-medium text-slate-600 mb-2">
                  Player {index + 1}
                </label>
                <input
                  type="text"
                  placeholder="Player {index + 1}"
                  class="w-full p-4 border-2 border-slate-200 rounded-xl focus:border-indigo-500 focus:outline-none transition-colors"
                />
              </div>
            {/each}
          </div>

          <div class="flex space-x-4">
            <button
              on:click={() => gameState = 'home'}
              class="flex-1 hover:cursor-pointer bg-slate-100 text-slate-700 font-semibold py-3 px-4 rounded-xl hover:bg-slate-200 transition-colors"
            >
              Back
            </button>
            <button
              on:click={startGame}
              disabled={isGeneratingWords}
              class="flex-2 bg-gradient-to-r hover:cursor-pointer from-indigo-500 to-purple-600 text-white font-semibold py-3 px-6 rounded-xl hover:from-indigo-600 hover:to-purple-700 transition-all duration-200 disabled:opacity-50 disabled:cursor-not-allowed"
            >
              {#if isGeneratingWords}
                <div class="flex items-center justify-center space-x-2">
                  <div class="animate-spin rounded-full h-4 w-4 border-2 border-white border-t-transparent"></div>
                  <span>Generating...</span>
                </div>
              {:else}
                Start Game
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
        
        <div class="bg-white rounded-3xl shadow-xl shadow-slate-200/50 p-8">
          <!-- Player Info -->
          <div class="text-center mb-8">
            <div class="w-16 h-16 bg-gradient-to-br from-slate-100 to-slate-200 rounded-full mx-auto mb-4 flex items-center justify-center text-xl font-bold text-slate-700">
              {currentPlayerIndex + 1}
            </div>
            <h2 class="text-2xl font-bold text-slate-800 mb-1">
              {currentPlayer?.name}'s Turn
            </h2>
            <p class="text-slate-500 text-sm">
              Player {currentPlayerIndex + 1} of {numPlayers}
            </p>
          </div>

          {#if !wordRevealed}
            <!-- Word Hidden State -->
            <div class="text-center space-y-6">
              <div class="bg-slate-50 p-8 rounded-2xl">
                <div class="w-16 h-16 mx-auto mb-4 text-slate-400">
                  {@html EyeOffIcon}
                </div>
                <p class="text-slate-500">Click the button to reveal your word</p>
              </div>

              <button
                on:click={revealWord}
                class="w-full bg-gradient-to-r hover:cursor-pointer from-indigo-500 to-purple-600 text-white font-semibold py-4 px-6 rounded-xl hover:from-indigo-600 hover:to-purple-700 transition-all duration-200 shadow-lg shadow-indigo-500/25 flex items-center justify-center space-x-3"
              >
                <div class="w-5 h-5">{@html EyeIcon}</div>
                <span>Reveal Word</span>
              </button>
            </div>
          {:else}
            <!-- Word Revealed State -->
            <div class="space-y-6">
              {#if showRoleInitially}
                <!-- Show Role and Word -->
                <div class="p-6 rounded-2xl text-center border-2
                  {currentPlayer?.role === 'imposter' ? 'bg-red-50 border-red-200' : 
                   currentPlayer?.role === 'dumb' ? 'bg-amber-50 border-amber-200' : 
                   'bg-emerald-50 border-emerald-200'}">
                  <h3 class="text-lg font-bold mb-3
                    {currentPlayer?.role === 'imposter' ? 'text-red-600' : 
                     currentPlayer?.role === 'dumb' ? 'text-amber-600' : 
                     'text-emerald-600'}">
                    {currentPlayer?.role === 'imposter' ? 'You are the Imposter' : 
                     currentPlayer?.role === 'dumb' ? 'You are the DUMB' : 
                     'You are a Civilian'}
                  </h3>
                  <div class="text-2xl font-bold text-slate-900 py-4 bg-white rounded-xl shadow-sm">
                    {currentPlayer?.word}
                  </div>
                  <p class="text-slate-500 text-sm mt-3">
                    {currentPlayer?.role === 'imposter' ? 'Your word is different!' : 
                     currentPlayer?.role === 'dumb' ? 'Figure out the word!' : 
                     'This is your word'}
                  </p>
                </div>
              {:else}
                <!-- Show Only Word -->
                <div class="p-6 rounded-2xl text-center border-2 border-slate-200 bg-slate-50">
                  <h3 class="text-lg font-bold mb-3 text-slate-700">Your Word</h3>
                  <div class="text-2xl font-bold text-slate-900 py-4 bg-white rounded-xl shadow-sm">
                    {currentPlayer?.word}
                  </div>
                  <p class="text-slate-500 text-sm mt-3">
                    {currentPlayer?.role === 'dumb' ? 'Figure out what this means!' : 'Remember this word'}
                  </p>
                </div>
              {/if}

              <button
                on:click={nextPlayer}
                class="w-full bg-gradient-to-r hover:cursor-pointer from-slate-500 to-slate-600 text-white font-semibold py-4 px-6 rounded-xl hover:from-slate-600 hover:to-slate-700 transition-all duration-200 flex items-center justify-center space-x-3"
              >
                <span>Next Player</span>
                <div class="w-5 h-5">{@html ArrowRightIcon}</div>
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
        
        <div class="bg-white rounded-3xl shadow-xl shadow-slate-200/50 p-8">
          <div class="text-center mb-8">
            <div class="w-20 h-20 bg-gradient-to-br from-indigo-500 to-purple-600 rounded-2xl mx-auto mb-6 flex items-center justify-center text-white">
              {@html SearchIcon}
            </div>
            <h2 class="text-2xl font-bold text-slate-800 mb-2">Game Ready!</h2>
            <p class="text-slate-500">Now discuss and find the imposter{includeDumbRole ? ' and dumb player' : ''}!</p>
          </div>

          <div class="space-y-4">
            <button
              on:click={showImposter}
              class="w-full bg-gradient-to-r hover:cursor-pointer from-red-500 to-pink-600 text-white font-semibold py-4 px-6 rounded-xl hover:from-red-600 hover:to-pink-700 transition-all duration-200 shadow-lg shadow-red-500/25 flex items-center justify-center space-x-3"
            >
              <div class="w-5 h-5">{@html SearchIcon}</div>
              <span>Reveal Results</span>
            </button>

            <button
              on:click={resetGame}
              class="w-full bg-slate-100 hover:cursor-pointer text-slate-700 font-semibold py-4 px-6 rounded-xl hover:bg-slate-200 transition-colors flex items-center justify-center space-x-3"
            >
              <div class="w-5 h-5">{@html RefreshIcon}</div>
              <span>New Game</span>
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
        
        <div class="bg-white rounded-3xl shadow-xl shadow-slate-200/50 p-8">
          <div class="text-center mb-8">
            <h2 class="text-2xl font-bold text-red-600 mb-6">Game Results!</h2>
            
            <!-- Imposter Result -->
            <div class="bg-red-50 border-2 border-red-200 p-6 rounded-2xl mb-4">
              <div class="w-12 h-12 mx-auto mb-3 text-red-500">
                {@html UserIcon}
              </div>
              <h3 class="text-xl font-bold text-red-700 mb-1">{imposter?.name}</h3>
              <p class="text-red-600 font-medium">was the Imposter!</p>
            </div>

            <!-- Dumb Player Result -->
            {#if dumbPlayer}
              <div class="bg-amber-50 border-2 border-amber-200 p-6 rounded-2xl mb-6">
                <div class="w-12 h-12 mx-auto mb-3 text-amber-500">
                  {@html UserIcon}
                </div>
                <h3 class="text-xl font-bold text-amber-700 mb-1">{dumbPlayer?.name}</h3>
                <p class="text-amber-600 font-medium">was the DUMB!</p>
              </div>
            {/if}

            <!-- Words Reveal -->
            <div class="space-y-3">
              <div class="bg-emerald-50 border-2 border-emerald-200 p-4 rounded-xl">
                <p class="text-emerald-700">
                  <span class="font-semibold">Civilian word:</span> 
                  <span class="font-mono text-lg font-bold">{gameData?.civilianWord}</span>
                </p>
              </div>
              <div class="bg-red-50 border-2 border-red-200 p-4 rounded-xl">
                <p class="text-red-700">
                  <span class="font-semibold">Imposter word:</span> 
                  <span class="font-mono text-lg font-bold">{gameData?.imposterWord}</span>
                </p>
              </div>
            </div>
          </div>

         
      <button
        on:click={resetGame}
        class="w-full hover:cursor-pointer bg-green-500 text-white font-medium py-3 px-4 rounded-lg hover:bg-green-600 flex items-center justify-center space-x-2 transition-colors"
      >
        <span>Play Again</span>
      </button>
    </div>
    </div>
    
</div>
{/if}
</div>


<style>
  /* Custom scrollbar styling */
  :global(.overflow-y-auto::-webkit-scrollbar) {
    width: 4px;
  }
  
  :global(.overflow-y-auto::-webkit-scrollbar-track) {
    background: #f3f4f6;
    border-radius: 10px;
  }
  
  :global(.overflow-y-auto::-webkit-scrollbar-thumb) {
    background: #d1d5db;
    border-radius: 10px;
  }
  
  :global(.overflow-y-auto::-webkit-scrollbar-thumb:hover) {
    background: #9ca3af;
  }
</style>