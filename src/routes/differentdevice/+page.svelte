<script lang="ts">
  import { GoogleGenerativeAI } from '@google/generative-ai';
  
  interface Player {
    id: string;
    name: string;
    role: 'imposter' | 'civilian' | 'dumb';
    word: string;
    isHost: boolean;
    hasRevealed: boolean;
  }
  
  interface GameRoom {
    roomCode: string;
    players: Player[];
    gameStarted: boolean;
    gameComplete: boolean;
    civilianWord: string;
    imposterWord: string;
    hostId: string;
    settings: {
      language: 'english' | 'nepali';
      showRoles: boolean;
      includeDumbRole: boolean;
    };
  }
  
  interface WordPair {
    civilian: string;
    imposter: string;
  }
  
  let gameState: 'home' | 'create' | 'join' | 'lobby' | 'game' | 'complete' | 'reveal' = 'home';
  let roomCode: string = '';
  let playerName: string = '';
  let currentPlayer: Player | null = null;
  let gameRoom: GameRoom | null = null;
  let joinRoomCode: string = '';
  let selectedLanguage: 'nepali' | 'english' = 'english';
  let showRoleInitially: boolean = false;
  let includeDumbRole: boolean = false;
  let isGeneratingWords: boolean = false;
  let wordRevealed: boolean = false;
  let playerId: string = '';
  
  // Simple in-memory storage simulation (in real app, this would be a backend service)
  let gameRooms: { [key: string]: GameRoom } = {};
  
  const PlayIcon = `<svg viewBox="0 0 24 24" fill="currentColor">
    <path d="M8 5v14l11-7z"/>
  </svg>`;
  const UsersIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <path d="M17 21v-2a4 4 0 0 0-4-4H5a4 4 0 0 0-4 4v2"/>
    <circle cx="9" cy="7" r="4"/>
    <path d="M23 21v-2a4 4 0 0 0-3-3.87"/>
    <circle cx="16" cy="7" r="3"/>
  </svg>`;
  const PlusIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <line x1="12" y1="5" x2="12" y2="19"/>
    <line x1="5" y1="12" x2="19" y2="12"/>
  </svg>`;
  const LoginIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <path d="M15 3h4a2 2 0 0 1 2 2v14a2 2 0 0 1-2 2h-4"/>
    <polyline points="10,17 15,12 10,7"/>
    <line x1="15" y1="12" x2="3" y2="12"/>
  </svg>`;
  const ArrowLeftIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <path d="m12 19-7-7 7-7"/>
    <path d="M19 12H5"/>
  </svg>`;
  const CopyIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <rect x="9" y="9" width="13" height="13" rx="2" ry="2"/>
    <path d="M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1"/>
  </svg>`;
  const EyeIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <path d="M1 12s4-8 11-8 11 8 11 8-4 8-11 8-11-8-11-8z"/>
    <circle cx="12" cy="12" r="3"/>
  </svg>`;
  const EyeOffIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <path d="M17.94 17.94A10.07 10.07 0 0 1 12 20c-7 0-11-8-11-8a18.45 18.45 0 0 1 5.06-5.94M9.9 4.24A9.12 9.12 0 0 1 12 4c7 0 11 8 11 8a18.5 18.5 0 0 1-2.16 3.19m-6.72-1.07a3 3 0 1 1-4.24-4.24"/>
    <path d="m1 1 22 22"/>
  </svg>`;
  const CheckIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <polyline points="20,6 9,17 4,12"/>
  </svg>`;
  const RefreshIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <path d="M3 12a9 9 0 0 1 9-9 9.75 9.75 0 0 1 6.74 2.74L21 8"/>
    <path d="M21 3v5h-5"/>
    <path d="M21 12a9 9 0 0 1-9 9 9.75 9.75 0 0 1-6.74-2.74L3 16"/>
    <path d="M3 21v-5h5"/>
  </svg>`;

  // Generate a random room code
  function generateRoomCode(): string {
    return Math.random().toString(36).substring(2, 8).toUpperCase();
  }
  
  // Generate unique player ID
  function generatePlayerId(): string {
    return Math.random().toString(36).substring(2, 15);
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
  
  // Create a new room
  function createRoom(): void {
    if (!playerName.trim()) {
      alert('Please enter your name!');
      return;
    }
    
    roomCode = generateRoomCode();
    playerId = generatePlayerId();
    
    currentPlayer = {
      id: playerId,
      name: playerName.trim(),
      role: 'civilian',
      word: '',
      isHost: true,
      hasRevealed: false
    };
    
    gameRoom = {
      roomCode,
      players: [currentPlayer],
      gameStarted: false,
      gameComplete: false,
      civilianWord: '',
      imposterWord: '',
      hostId: playerId,
      settings: {
        language: selectedLanguage,
        showRoles: showRoleInitially,
        includeDumbRole: includeDumbRole
      }
    };
    
    gameRooms[roomCode] = gameRoom;
    gameState = 'lobby';
  }
  
  // Join an existing room
  function joinRoom(): void {
    if (!playerName.trim()) {
      alert('Please enter your name!');
      return;
    }
    
    if (!joinRoomCode.trim()) {
      alert('Please enter a room code!');
      return;
    }
    
    const code = joinRoomCode.trim().toUpperCase();
    const room = gameRooms[code];
    
    if (!room) {
      alert('Room not found!');
      return;
    }
    
    if (room.gameStarted) {
      alert('Game has already started!');
      return;
    }
    
    if (room.players.some(p => p.name.toLowerCase() === playerName.trim().toLowerCase())) {
      alert('A player with this name already exists in the room!');
      return;
    }
    
    playerId = generatePlayerId();
    roomCode = code;
    
    currentPlayer = {
      id: playerId,
      name: playerName.trim(),
      role: 'civilian',
      word: '',
      isHost: false,
      hasRevealed: false
    };
    
    room.players.push(currentPlayer);
    gameRoom = room;
    gameState = 'lobby';
  }
  
  // Start the game (host only)
  async function startGame(): Promise<void> {
    if (!gameRoom || !currentPlayer?.isHost) return;
    
    if (gameRoom.players.length < 3) {
      alert('At least 3 players are required!');
      return;
    }
    
    isGeneratingWords = true;
    
    const prompt = gameRoom.settings.language === 'nepali'
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
   
    selectedLanguage = gameRoom.settings.language;
    const wordPair = await generateWithGemini(prompt);
    
    // Assign roles
    const playerCount = gameRoom.players.length;
    const imposterIndex = Math.floor(Math.random() * playerCount);
    
    let dumbIndex = -1;
    if (gameRoom.settings.includeDumbRole) {
      const availableIndices = Array.from({length: playerCount}, (_, i) => i)
        .filter(i => i !== imposterIndex);
      if (availableIndices.length > 0) {
        dumbIndex = availableIndices[Math.floor(Math.random() * availableIndices.length)];
      }
    }
    
    // Update all players with their roles and words
    gameRoom.players.forEach((player, index) => {
      if (index === imposterIndex) {
        player.role = 'imposter';
        player.word = wordPair.imposter;
      } else if (index === dumbIndex) {
        player.role = 'dumb';
        player.word = '???';
      } else {
        player.role = 'civilian';
        player.word = wordPair.civilian;
      }
      player.hasRevealed = false;
    });
    
    gameRoom.civilianWord = wordPair.civilian;
    gameRoom.imposterWord = wordPair.imposter;
    gameRoom.gameStarted = true;
    
    gameState = 'game';
    wordRevealed = false;
    isGeneratingWords = false;
  }
  
  function copyRoomCode(): void {
    navigator.clipboard.writeText(roomCode).then(() => {
      // Could show a toast notification here
    });
  }
  
  function revealWord(): void {
    wordRevealed = true;
    if (currentPlayer) {
      currentPlayer.hasRevealed = true;
    }
  }
  
  function showResults(): void {
    if (!gameRoom) return;
    gameRoom.gameComplete = true;
    gameState = 'reveal';
  }
  
  function resetGame(): void {
    gameState = 'home';
    gameRoom = null;
    currentPlayer = null;
    roomCode = '';
    playerId = '';
    playerName = '';
    joinRoomCode = '';
    wordRevealed = false;
    selectedLanguage = 'english';
    showRoleInitially = false;
    includeDumbRole = false;
  }
  
  function startNewGame(): void {
    if (!gameRoom || !currentPlayer?.isHost) return;
    
    // Reset game state but keep players
    gameRoom.gameStarted = false;
    gameRoom.gameComplete = false;
    gameRoom.players.forEach(player => {
      player.role = 'civilian';
      player.word = '';
      player.hasRevealed = false;
    });
    
    gameState = 'lobby';
    wordRevealed = false;
  }
  
  function goToCreateRoom(): void {
    gameState = 'create';
  }
  
  function goToJoinRoom(): void {
    gameState = 'join';
  }
  
  function backToHome(): void {
    gameState = 'home';
  }
  
  // Get current player's data
  $: {
    if (gameRoom && playerId) {
      currentPlayer = gameRoom.players.find(p => p.id === playerId) || null;
    }
  }
  
  // Get game statistics
  $: playersRevealed = gameRoom?.players.filter(p => p.hasRevealed).length || 0;
  $: totalPlayers = gameRoom?.players.length || 0;
  $: imposterPlayer = gameRoom?.players.find(p => p.role === 'imposter');
  $: dumbPlayer = gameRoom?.players.find(p => p.role === 'dumb');
</script>

<div class="min-h-screen bg-gradient-to-br from-gray-50 via-white to-gray-100">

  <!-- Home Screen -->
  {#if gameState === 'home'}
    <div class="container mx-auto px-4 py-8 md:py-16">
      <div class="max-w-md mx-auto">
        
        <!-- Header -->
        <div class="text-center mb-12">
          <h1 class="text-4xl md:text-5xl font-black text-gray-900 mb-3">
            Multi-Device Spy
          </h1>
          <p class="text-gray-600">Everyone plays on their own device!</p>
        </div>
        
        <!-- Game Options -->
        <div class="space-y-4">
          
          <!-- Create Room Button -->
          <button
            on:click={goToCreateRoom}
            class="w-full bg-gradient-to-r from-green-500 to-teal-600 hover:from-green-600 hover:to-teal-700 text-white font-black py-8 px-8 rounded-2xl transition-all duration-200 shadow-xl hover:shadow-green-500/25 flex items-center justify-center space-x-4 hover:scale-105 transform hover:cursor-pointer"
          >
            <div class="w-16 h-16 bg-white/20 rounded-2xl flex items-center justify-center">
              <div class="w-10 h-10 text-white">{@html PlusIcon}</div>
            </div>
            <div class="text-left">
              <div class="text-2xl font-black">CREATE ROOM</div>
              <div class="text-green-100 text-sm font-medium">Start a new game and invite friends</div>
            </div>
          </button>
          
          <!-- Join Room Button -->
          <button
            on:click={goToJoinRoom}
            class="w-full bg-gradient-to-r from-blue-500 to-purple-600 hover:from-blue-600 hover:to-purple-700 text-white font-black py-8 px-8 rounded-2xl transition-all duration-200 shadow-xl hover:shadow-blue-500/25 flex items-center justify-center space-x-4 hover:scale-105 transform hover:cursor-pointer"
          >
            <div class="w-16 h-16 bg-white/20 rounded-2xl flex items-center justify-center">
              <div class="w-10 h-10 text-white">{@html LoginIcon}</div>
            </div>
            <div class="text-left">
              <div class="text-2xl font-black">JOIN ROOM</div>
              <div class="text-blue-100 text-sm font-medium">Enter a room code to join a game</div>
            </div>
          </button>
        </div>
        
        <!-- Back Button -->
        <div class="mt-8">
          <a 
            href="/"
            class="w-full bg-gray-100 hover:bg-gray-200 text-gray-700 font-bold py-4 px-4 rounded-xl transition-all duration-200 hover:scale-105 block text-center hover:cursor-pointer"
          >
            ← Back to Home
          </a>
        </div>
      </div>
    </div>
  {/if}

  <!-- Create Room Screen -->
  {#if gameState === 'create'}
    <div class="container mx-auto px-4 py-8 md:py-16">
      <div class="max-w-md mx-auto">
        
        <div class="bg-white rounded-2xl shadow-xl border border-gray-100 p-8">
          <div class="text-center mb-6">
            <div class="w-16 h-16 bg-gradient-to-br from-green-500 to-teal-600 rounded-2xl mx-auto mb-4 flex items-center justify-center shadow-xl">
              <div class="w-8 h-8 text-white">{@html PlusIcon}</div>
            </div>
            <h2 class="text-2xl font-black text-gray-900 mb-2">Create Room</h2>
            <p class="text-gray-600 text-sm">Start a new game and invite friends</p>
          </div>
          
          <div class="space-y-4">
            <div>
              <label class="block text-sm font-bold text-gray-700 mb-2">Your Name</label>
              <input
                type="text"
                bind:value={playerName}
                placeholder="Enter your name..."
                class="w-full p-4 border-2 border-gray-200 rounded-xl focus:border-green-500 focus:outline-none transition-all bg-gray-50 hover:bg-gray-100"
              />
            </div>
            
            <!-- Language Selection -->
            <div>
              <label class="block text-sm font-bold text-gray-700 mb-3">🌍 Language</label>
              <div class="grid grid-cols-2 gap-2 p-1 bg-gray-100 rounded-xl">
                <button
                  type="button"
                  on:click={() => selectedLanguage = 'english'}
                  class="flex items-center justify-center py-3 px-4 rounded-lg text-sm font-semibold transition-all duration-200 hover:cursor-pointer
                    {selectedLanguage === 'english' ? 'bg-white shadow-lg text-gray-900 border-2 border-gray-200' : 'text-gray-600 hover:text-gray-800'}"
                >
                  English
                </button>
                <button
                  type="button"
                  on:click={() => selectedLanguage = 'nepali'}
                  class="flex items-center justify-center py-3 px-4 rounded-lg text-sm font-semibold transition-all duration-200 hover:cursor-pointer
                    {selectedLanguage === 'nepali' ? 'bg-white shadow-lg text-gray-900 border-2 border-gray-200' : 'text-gray-600 hover:text-gray-800'}"
                >
                  नेपाली
                </button>
              </div>
            </div>
            
            <!-- Game Options -->
            <div class="space-y-3">
              <div class="flex items-center justify-between p-4 bg-gray-50 rounded-xl border border-gray-100">
                <div>
                  <div class="font-bold text-gray-800 text-sm">🎭 Show roles</div>
                  <div class="text-xs text-gray-500">Reveal player roles immediately</div>
                </div>
                <button
                  type="button"
                  on:click={() => showRoleInitially = !showRoleInitially}
                  class="relative w-14 h-8 rounded-full transition-all duration-300 focus:outline-none hover:scale-110 hover:cursor-pointer
                    {showRoleInitially ? 'bg-red-500 shadow-lg' : 'bg-gray-300'}"
                >
                  <div class="absolute w-6 h-6 bg-white rounded-full shadow-sm transition-transform duration-300 top-1 flex items-center justify-center
                    {showRoleInitially ? 'translate-x-7' : 'translate-x-1'}">
                    {#if showRoleInitially}
                      <div class="w-3 h-3 text-red-500">⭐</div>
                    {/if}
                  </div>
                </button>
              </div>
              
              <div class="flex items-center justify-between p-4 bg-gray-50 rounded-xl border border-gray-100">
                <div>
                  <div class="font-bold text-gray-800 text-sm">🤔 Add Mystery Player</div>
                  <div class="text-xs text-gray-500">Someone gets no word at all</div>
                </div>
                <button
                  type="button"
                  on:click={() => includeDumbRole = !includeDumbRole}
                  class="relative w-14 h-8 rounded-full transition-all duration-300 focus:outline-none hover:scale-110 hover:cursor-pointer
                    {includeDumbRole ? 'bg-green-500 shadow-lg' : 'bg-gray-300'}"
                >
                  <div class="absolute w-6 h-6 bg-white rounded-full shadow-sm transition-transform duration-300 top-1 flex items-center justify-center
                    {includeDumbRole ? 'translate-x-7' : 'translate-x-1'}">
                    {#if includeDumbRole}
                      <div class="text-green-500 text-xs font-bold">?</div>
                    {/if}
                  </div>
                </button>
              </div>
            </div>
            
            <button
              on:click={createRoom}
              class="w-full bg-gradient-to-r from-green-500 to-teal-600 hover:from-green-600 hover:to-teal-700 text-white font-black py-5 px-6 rounded-xl transition-all duration-200 shadow-xl hover:shadow-green-500/25 flex items-center justify-center space-x-3 hover:scale-105 transform hover:cursor-pointer"
            >
              <div class="w-6 h-6">{@html PlusIcon}</div>
              <span class="text-lg">CREATE ROOM</span>
            </button>

            <button
              on:click={backToHome}
              class="w-full bg-gray-100 hover:bg-gray-200 text-gray-700 font-bold py-4 px-4 rounded-xl transition-all duration-200 hover:scale-105 flex items-center justify-center space-x-2 hover:cursor-pointer"
            >
              <div class="w-5 h-5">{@html ArrowLeftIcon}</div>
              <span>Back</span>
            </button>
          </div>
        </div>
      </div>
    </div>
  {/if}

  <!-- Join Room Screen -->
  {#if gameState === 'join'}
    <div class="container mx-auto px-4 py-8 md:py-16">
      <div class="max-w-md mx-auto">
        
        <div class="bg-white rounded-2xl shadow-xl border border-gray-100 p-8">
          <div class="text-center mb-6">
            <div class="w-16 h-16 bg-gradient-to-br from-blue-500 to-purple-600 rounded-2xl mx-auto mb-4 flex items-center justify-center shadow-xl">
              <div class="w-8 h-8 text-white">{@html LoginIcon}</div>
            </div>
            <h2 class="text-2xl font-black text-gray-900 mb-2">Join Room</h2>
            <p class="text-gray-600 text-sm">Enter a room code to join a game</p>
          </div>
          
          <div class="space-y-4">
            <div>
              <label class="block text-sm font-bold text-gray-700 mb-2">Your Name</label>
              <input
                type="text"
                bind:value={playerName}
                placeholder="Enter your name..."
                class="w-full p-4 border-2 border-gray-200 rounded-xl focus:border-blue-500 focus:outline-none transition-all bg-gray-50 hover:bg-gray-100"
              />
            </div>
            
            <div>
              <label class="block text-sm font-bold text-gray-700 mb-2">Room Code</label>
              <input
                type="text"
                bind:value={joinRoomCode}
                placeholder="Enter 6-letter code..."
                class="w-full p-4 border-2 border-gray-200 rounded-xl focus:border-blue-500 focus:outline-none transition-all bg-gray-50 hover:bg-gray-100 uppercase text-center text-2xl font-black"
                maxlength="6"
              />
            </div>
            
            <button
              on:click={joinRoom}
              class="w-full bg-gradient-to-r from-blue-500 to-purple-600 hover:from-blue-600 hover:to-purple-700 text-white font-black py-5 px-6 rounded-xl transition-all duration-200 shadow-xl hover:shadow-blue-500/25 flex items-center justify-center space-x-3 hover:scale-105 transform hover:cursor-pointer"
            >
              <div class="w-6 h-6">{@html LoginIcon}</div>
              <span class="text-lg">JOIN ROOM</span>
            </button>

            <button
              on:click={backToHome}
              class="w-full bg-gray-100 hover:bg-gray-200 text-gray-700 font-bold py-4 px-4 rounded-xl transition-all duration-200 hover:scale-105 flex items-center justify-center space-x-2 hover:cursor-pointer"
            >
              <div class="w-5 h-5">{@html ArrowLeftIcon}</div>
              <span>Back</span>
            </button>
          </div>
        </div>
      </div>
    </div>
  {/if}

  <!-- Lobby Screen -->
  {#if gameState === 'lobby'}
    <div class="container mx-auto px-4 py-8 md:py-16">
      <div class="max-w-md mx-auto">
        
        <div class="bg-white rounded-2xl shadow-xl border border-gray-100 p-8">
          <!-- Room Info -->
          <div class="text-center mb-8">
            <div class="w-20 h-20 bg-gradient-to-br from-green-500 to-teal-600 rounded-2xl mx-auto mb-4 flex items-center justify-center shadow-xl">
              <div class="w-10 h-10 text-white">{@html UsersIcon}</div>
            </div>
            <h2 class="text-3xl font-black text-gray-900 mb-2">Game Lobby</h2>
            <div class="flex items-center justify-center space-x-4">
              <div class="bg-gray-100 px-4 py-2 rounded-xl">
                <span class="text-sm font-bold text-gray-700">Room:</span>
                <span class="text-xl font-black text-gray-900 ml-2">{roomCode}</span>
              </div>
              <button
                on:click={copyRoomCode}
                class="p-2 bg-gray-100 hover:bg-gray-200 rounded-lg transition-all hover:cursor-pointer"
                title="Copy room code"
              >
                <div class="w-5 h-5 text-gray-600">{@html CopyIcon}</div>
              </button>
            </div>
          </div>
          
          <!-- Players List -->
          <div class="mb-8">
            <h3 class="text-lg font-bold text-gray-900 mb-4">Players ({gameRoom?.players.length})</h3>
            <div class="space-y-3 max-h-64 overflow-y-auto">
              {#each gameRoom?.players || [] as player}
                <div class="flex items-center justify-between p-4 bg-gray-50 rounded-xl border border-gray-100">
                  <div class="flex items-center space-x-3">
                    <div class="w-10 h-10 bg-gradient-to-br from-gray-600 to-gray-800 rounded-full flex items-center justify-center text-white font-bold">
                      {player.name.charAt(0).toUpperCase()}
                    </div>
                    <div>
                      <div class="font-bold text-gray-900">{player.name}</div>
                      {#if player.isHost}
                        <div class="text-xs text-green-600 font-bold">HOST</div>
                      {/if}
                    </div>
                  </div>
                  {#if player.id === playerId}
                    <div class="text-blue-600 font-bold text-sm">You</div>
                  {/if}
                </div>
              {/each}
            </div>
          </div>
          
          <!-- Game Settings (Host only) -->
          {#if currentPlayer?.isHost}
            <div class="mb-8 p-6 bg-gray-50 rounded-xl border border-gray-100">
              <h3 class="font-bold text-gray-900 mb-4">Game Settings</h3>
              <div class="space-y-3 text-sm">
                <div class="flex justify-between">
                  <span class="text-gray-600">Language:</span>
                  <span class="font-bold text-gray-900">{gameRoom?.settings.language === 'nepali' ? 'नेपाली' : 'English'}</span>
                </div>
                <div class="flex justify-between">
                  <span class="text-gray-600">Show Roles:</span>
                  <span class="font-bold text-gray-900">{gameRoom?.settings.showRoles ? 'Yes' : 'No'}</span>
                </div>
                <div class="flex justify-between">
                  <span class="text-gray-600">Mystery Player:</span>
                  <span class="font-bold text-gray-900">{gameRoom?.settings.includeDumbRole ? 'Yes' : 'No'}</span>
                </div>
              </div>
            </div>
          {/if}
          
          <!-- Actions -->
          <div class="space-y-4">
            {#if currentPlayer?.isHost}
              <button
                on:click={startGame}
                disabled={isGeneratingWords || (gameRoom?.players.length || 0) < 3}
                class="w-full bg-gradient-to-r from-red-500 to-pink-600 hover:from-red-600 hover:to-pink-700 text-white font-black py-5 px-6 rounded-xl transition-all duration-200 shadow-xl hover:shadow-red-500/25 flex items-center justify-center space-x-3 hover:scale-105 transform disabled:opacity-50 disabled:cursor-not-allowed disabled:hover:scale-100 hover:cursor-pointer"
              >
                {#if isGeneratingWords}
                  <div class="flex items-center justify-center space-x-2">
                    <div class="animate-spin rounded-full h-5 w-5 border-2 border-white border-t-transparent"></div>
                    <span>Creating Words</span>
                  </div>
                {:else}
                  <div class="w-6 h-6">{@html PlayIcon}</div>
                  <span class="text-lg">START GAME</span>
                {/if}
              </button>
              
              {#if (gameRoom?.players.length || 0) < 3}
                <p class="text-sm text-gray-500 text-center">Need at least 3 players to start</p>
              {/if}
            {:else}
              <div class="text-center p-6 bg-blue-50 rounded-xl border border-blue-200">
                <div class="text-2xl mb-2">⏳</div>
                <p class="text-blue-800 font-bold">Waiting for host to start the game...</p>
              </div>
            {/if}
            
            <button
              on:click={resetGame}
              class="w-full bg-gray-100 hover:bg-gray-200 text-gray-700 font-bold py-4 px-4 rounded-xl transition-all duration-200 hover:scale-105 hover:cursor-pointer"
            >
              ← Leave Room
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
        
        <div class="bg-white rounded-2xl shadow-xl border border-gray-100 p-8">
          <!-- Player Info -->
          <div class="text-center mb-8">
            <div class="w-20 h-20 bg-gray-900 rounded-full mx-auto mb-4 flex items-center justify-center text-2xl font-black text-white shadow-lg">
              {currentPlayer?.name.charAt(0).toUpperCase()}
            </div>
            <h2 class="text-3xl font-black text-gray-800 mb-2">
              {currentPlayer?.name}
            </h2>
            <div class="inline-flex items-center bg-gray-100 text-gray-700 px-4 py-2 rounded-full text-sm font-bold">
              Room: {roomCode}
            </div>
          </div>
          
          {#if !wordRevealed}
            <!-- Word Hidden State -->
            <div class="text-center space-y-6">
              <div class="bg-gray-50 p-10 rounded-2xl border-2 border-dashed border-gray-300">
                <div class="w-16 h-16 mx-auto mb-6 text-gray-400">{@html EyeOffIcon}</div>
                <p class="text-gray-500 font-medium">Tap to see your secret word</p>
              </div>
              <button
                on:click={revealWord}
                class="w-full bg-red-500 text-white font-black py-5 px-6 rounded-xl hover:bg-red-600 transition-all duration-200 shadow-xl hover:shadow-red-500/25 flex items-center justify-center space-x-3 hover:scale-105 transform hover:cursor-pointer"
              >
                <div class="w-6 h-6">{@html EyeIcon}</div>
                <span class="text-lg">REVEAL MY WORD</span>
              </button>
            </div>
          {:else}
            <!-- Word Revealed State -->
            <div class="space-y-6">
              {#if gameRoom?.settings.showRoles}
                <!-- Show Role and Word -->
                <div class="p-8 rounded-2xl text-center border-3 transform hover:scale-105 transition-all duration-200
                  {currentPlayer?.role === 'imposter' ? 'bg-red-50 border-red-200 shadow-red-500/20' :
                   currentPlayer?.role === 'dumb' ? 'bg-yellow-50 border-yellow-200 shadow-yellow-500/20' :
                   'bg-green-50 border-green-200 shadow-green-500/20'} shadow-2xl">
                  
                  <div class="text-4xl mb-4">
                    {currentPlayer?.role === 'imposter' ? '🕵️' :
                     currentPlayer?.role === 'dumb' ? '🤔' : '👥'}
                  </div>
                  
                  <h3 class="text-xl font-black mb-4
                    {currentPlayer?.role === 'imposter' ? 'text-red-600' :
                     currentPlayer?.role === 'dumb' ? 'text-yellow-600' :
                     'text-green-600'}">
                    {currentPlayer?.role === 'imposter' ? 'You are the SPY!' :
                     currentPlayer?.role === 'dumb' ? 'You are the MYSTERY!' :
                     'You are a CIVILIAN'}
                  </h3>
                  
                  <div class="text-3xl font-black text-gray-900 py-6 px-4 bg-white rounded-xl shadow-lg border-2 border-gray-100">
                    {currentPlayer?.word}
                  </div>
                  
                  <p class="text-gray-600 text-sm mt-4 font-medium">
                    {currentPlayer?.role === 'imposter' ? 'Your word is different from others!' :
                     currentPlayer?.role === 'dumb' ? 'Figure out what everyone else has!' :
                     'Remember this word for discussion!'}
                  </p>
                </div>
              {:else}
                <!-- Show Only Word -->
                <div class="p-8 rounded-2xl text-center border-3 border-gray-200 bg-gray-50 shadow-2xl shadow-gray-500/20 transform hover:scale-105 transition-all duration-200">
                  <div class="text-4xl mb-4">🎭</div>
                  <h3 class="text-xl font-black mb-4 text-gray-700">Your Secret Word</h3>
                  <div class="text-3xl font-black text-gray-900 py-6 px-4 bg-white rounded-xl shadow-lg border-2 border-gray-100">
                    {currentPlayer?.word}
                  </div>
                  <p class="text-gray-600 text-sm mt-4 font-medium">
                    {currentPlayer?.role === 'dumb' ? 'Hmm... what could this mean?' : 'Keep this secret until the discussion!'}
                  </p>
                </div>
              {/if}
              
              <!-- Player Status -->
              <div class="bg-blue-50 border border-blue-200 p-4 rounded-xl text-center">
                <p class="text-blue-800 font-bold text-sm">
                  {playersRevealed} of {totalPlayers} players have seen their words
                </p>
                <div class="w-full bg-blue-200 rounded-full h-2 mt-2">
                  <div 
                    class="bg-blue-500 h-2 rounded-full transition-all duration-500"
                    style="width: {(playersRevealed / totalPlayers) * 100}%"
                  ></div>
                </div>
              </div>
              
              <!-- Host Controls -->
              {#if currentPlayer?.isHost}
                <div class="pt-4 border-t border-gray-100">
                  <button
                    on:click={showResults}
                    class="w-full bg-gradient-to-r from-purple-500 to-indigo-600 hover:from-purple-600 hover:to-indigo-700 text-white font-black py-4 px-6 rounded-xl transition-all duration-200 shadow-xl flex items-center justify-center space-x-3 hover:scale-105 transform hover:cursor-pointer"
                  >
                    <span class="text-lg">🔍 REVEAL RESULTS</span>
                  </button>
                  <p class="text-xs text-gray-500 text-center mt-2">Only you can see this button</p>
                </div>
              {:else}
                <div class="bg-yellow-50 border border-yellow-200 p-4 rounded-xl text-center">
                  <p class="text-yellow-800 font-bold text-sm">
                    ⏳ Discuss with other players. The host will reveal results when ready.
                  </p>
                </div>
              {/if}
            </div>
          {/if}
        </div>
      </div>
    </div>
  {/if}

  <!-- Reveal Screen -->
  {#if gameState === 'reveal'}
    <div class="container mx-auto px-4 py-8 md:py-16">
      <div class="max-w-md mx-auto">
        
        <div class="bg-white rounded-2xl shadow-xl border border-gray-100 p-8">
          <div class="text-center mb-8">
            <div class="text-6xl mb-4">🎉</div>
            <h2 class="text-3xl font-black text-gray-900 mb-6">Game Results!</h2>
           
            <!-- Imposter Result -->
            <div class="bg-red-50 border-3 border-red-200 p-8 rounded-2xl mb-4 shadow-xl shadow-red-500/20 transform hover:scale-105 transition-all duration-200">
              <div class="text-4xl mb-3">🕵️</div>
              <h3 class="text-2xl font-black text-red-700 mb-2">{imposterPlayer?.name}</h3>
              <p class="text-red-600 font-bold text-lg">was the SPY!</p>
              <div class="mt-4 inline-block bg-red-100 text-red-700 px-4 py-2 rounded-full text-sm font-bold">
                Word: {gameRoom?.imposterWord}
              </div>
            </div>
            
            <!-- Dumb Player Result -->
            {#if dumbPlayer}
              <div class="bg-yellow-50 border-3 border-yellow-200 p-8 rounded-2xl mb-6 shadow-xl shadow-yellow-500/20 transform hover:scale-105 transition-all duration-200">
                <div class="text-4xl mb-3">🤔</div>
                <h3 class="text-2xl font-black text-yellow-700 mb-2">{dumbPlayer?.name}</h3>
                <p class="text-yellow-600 font-bold text-lg">was the MYSTERY!</p>
                <div class="mt-4 inline-block bg-yellow-100 text-yellow-700 px-4 py-2 rounded-full text-sm font-bold">
                  Had no word!
                </div>
              </div>
            {/if}
            
            <!-- Civilian Word -->
            <div class="bg-green-50 border-3 border-green-200 p-6 rounded-xl shadow-xl shadow-green-500/20 mb-6">
              <div class="text-2xl mb-2">👥</div>
              <p class="text-green-700 font-bold">
                <span class="text-sm">Civilian word:</span><br>
                <span class="text-2xl font-black">{gameRoom?.civilianWord}</span>
              </p>
            </div>
            
            <!-- All Players -->
            <div class="bg-gray-50 border border-gray-200 p-4 rounded-xl">
              <h4 class="font-bold text-gray-900 mb-3 text-sm">All Players:</h4>
              <div class="space-y-2 text-sm">
                {#each gameRoom?.players || [] as player}
                  <div class="flex justify-between items-center">
                    <span class="font-medium">{player.name}</span>
                    <div class="flex items-center space-x-2">
                      <span class="text-xs px-2 py-1 rounded-full
                        {player.role === 'imposter' ? 'bg-red-100 text-red-700' :
                         player.role === 'dumb' ? 'bg-yellow-100 text-yellow-700' :
                         'bg-green-100 text-green-700'}">
                        {player.role === 'imposter' ? 'Spy' :
                         player.role === 'dumb' ? 'Mystery' : 'Civilian'}
                      </span>
                    </div>
                  </div>
                {/each}
              </div>
            </div>
          </div>
         
          <div class="space-y-4">
            <!-- Host Controls -->
            {#if currentPlayer?.isHost}
              <button
                on:click={startNewGame}
                class="w-full bg-gradient-to-r from-green-500 to-teal-600 hover:from-green-600 hover:to-teal-700 text-white font-black py-5 px-6 rounded-xl transition-all duration-200 shadow-xl flex items-center justify-center space-x-3 hover:scale-105 transform hover:cursor-pointer"
              >
                <div class="w-6 h-6">{@html RefreshIcon}</div>
                <span class="text-lg">🔄 NEW ROUND</span>
              </button>
            {:else}
              <div class="bg-blue-50 border border-blue-200 p-4 rounded-xl text-center">
                <p class="text-blue-800 font-bold text-sm">
                  ⏳ Waiting for host to start a new round...
                </p>
              </div>
            {/if}
            
            <!-- Leave Room Button -->
            <button
              on:click={resetGame}
              class="w-full bg-gray-100 hover:bg-gray-200 text-gray-700 font-black py-5 px-6 rounded-xl transition-all duration-200 flex items-center justify-center space-x-3 hover:scale-105 transform hover:cursor-pointer"
            >
              <span class="text-lg">← LEAVE ROOM</span>
            </button>
          </div>
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
    background: linear-gradient(to bottom, #ef4444, #dc2626);
    border-radius: 10px;
  }
 
  :global(.overflow-y-auto::-webkit-scrollbar-thumb:hover) {
    background: linear-gradient(to bottom, #dc2626, #b91c1c);
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