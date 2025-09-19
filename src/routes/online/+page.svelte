<!-- Home Screen -->
<div class="min-h-screen flex flex-col items-center justify-center bg-gradient-to-br from-green-500 to-red-500 text-white p-6">
  
  <!-- Title -->
  <h1 class="text-4xl md:text-5xl font-extrabold mb-10 tracking-tight drop-shadow-lg">
    Who is the Spy? 👀
  </h1>

  <!-- Buttons -->
  <div class="flex flex-col gap-4 w-full max-w-xs">
    <button
      class="w-full bg-green-500 hover:bg-green-600 text-white font-semibold py-3 rounded-xl shadow-lg transition transform hover:scale-105">
      Create Room
    </button>

    <button
      class="w-full bg-red-500 hover:bg-red-600 text-white font-semibold py-3 rounded-xl shadow-lg transition transform hover:scale-105">
      Join Room
    </button>
  </div>

  <!-- Footer -->
  <p class="mt-16 text-sm opacity-75">Gather your friends & start playing 🎉</p>
</div>


<!-- Create Room Screen -->
<div class="min-h-screen flex flex-col items-center justify-center bg-gradient-to-br from-green-500 to-red-500 text-white p-6">
  
  <!-- Title -->
  <h1 class="text-3xl md:text-4xl font-extrabold mb-8 drop-shadow-lg">
    Create a Room 🛠️
  </h1>

  <!-- Card -->
  <div class="bg-white/10 backdrop-blur-lg rounded-2xl shadow-2xl p-8 w-full max-w-md space-y-6">

    <!-- Player Name -->
    <div>
      <label class="block text-left text-sm font-medium mb-2">Your Name</label>
      <input
        type="text"
        placeholder="Enter your name"
        class="w-full px-4 py-3 rounded-xl bg-white/20 text-white placeholder-white/60 focus:outline-none focus:ring-2 focus:ring-green-400"
      />
    </div>

    <!-- Language Select -->
    <div>
      <label class="block text-left text-sm font-medium mb-2">Language</label>
      <select
        class="w-full px-4 py-3 rounded-xl bg-white/20 text-white placeholder-white/60 focus:outline-none focus:ring-2 focus:ring-green-400">
        <option>English</option>
        <option>Nepali</option>
      </select>
    </div>

    <!-- Options -->
    <div class="flex items-center justify-between">
      <label class="flex items-center space-x-2 cursor-pointer">
        <input type="checkbox" class="w-5 h-5 rounded accent-green-500" />
        <span class="text-sm">Show Role Initially</span>
      </label>
    </div>
    <div class="flex items-center justify-between">
      <label class="flex items-center space-x-2 cursor-pointer">
        <input type="checkbox" class="w-5 h-5 rounded accent-red-500" />
        <span class="text-sm">Include Dumb Role</span>
      </label>
    </div>

    <!-- Action Buttons -->
    <div class="flex justify-between gap-4">
      <button
        class="flex-1 bg-gray-500 hover:bg-gray-600 text-white font-semibold py-3 rounded-xl shadow-lg transition">
        Back
      </button>
      <button
        class="flex-1 bg-green-500 hover:bg-green-600 text-white font-semibold py-3 rounded-xl shadow-lg transition">
        Create
      </button>
    </div>
  </div>
</div>


<!-- Join Room Screen -->
<div class="min-h-screen flex flex-col items-center justify-center bg-gradient-to-br from-green-500 to-red-500 text-white p-6">
  
  <!-- Title -->
  <h1 class="text-3xl md:text-4xl font-extrabold mb-8 drop-shadow-lg">
    Join a Room 🔎
  </h1>

  <!-- Card -->
  <div class="bg-white/10 backdrop-blur-lg rounded-2xl shadow-2xl p-8 w-full max-w-md space-y-6">

    <!-- Player Name -->
    <div>
      <label class="block text-left text-sm font-medium mb-2">Your Name</label>
      <input
        type="text"
        placeholder="Enter your name"
        class="w-full px-4 py-3 rounded-xl bg-white/20 text-white placeholder-white/60 focus:outline-none focus:ring-2 focus:ring-green-400"
      />
    </div>

    <!-- Room Code -->
    <div>
      <label class="block text-left text-sm font-medium mb-2">Room Code</label>
      <input
        type="text"
        placeholder="Enter code"
        class="w-full px-4 py-3 rounded-xl bg-white/20 text-white placeholder-white/60 focus:outline-none focus:ring-2 focus:ring-red-400 uppercase tracking-widest"
      />
    </div>

    <!-- Action Buttons -->
    <div class="flex justify-between gap-4">
      <button
        class="flex-1 bg-gray-500 hover:bg-gray-600 text-white font-semibold py-3 rounded-xl shadow-lg transition">
        Back
      </button>
      <button
        class="flex-1 bg-green-500 hover:bg-green-600 text-white font-semibold py-3 rounded-xl shadow-lg transition">
        Join
      </button>
    </div>
  </div>
</div>

<!-- Lobby Screen -->
<div class="min-h-screen flex flex-col items-center justify-center bg-gradient-to-br from-green-500 to-red-500 text-white p-6">

  <!-- Title -->
  <h1 class="text-3xl md:text-4xl font-extrabold mb-6 drop-shadow-lg">
    Game Lobby 🎉
  </h1>

  <!-- Room Info -->
  <div class="mb-8 text-center">
    <p class="text-sm opacity-80">Room Code</p>
    <p class="text-3xl font-bold tracking-widest bg-white/20 rounded-lg px-6 py-2 mt-2 inline-block">
      ABC123
    </p>
  </div>

  <!-- Lobby Card -->
  <div class="bg-white/10 backdrop-blur-lg rounded-2xl shadow-2xl p-8 w-full max-w-lg space-y-6">

    <!-- Players List -->
    <div>
      <h2 class="text-lg font-semibold mb-4">Players</h2>
      <div class="grid grid-cols-2 gap-3">
        <div class="bg-white/20 rounded-xl py-3 text-center font-medium">Alice</div>
        <div class="bg-white/20 rounded-xl py-3 text-center font-medium">Bob</div>
        <div class="bg-white/20 rounded-xl py-3 text-center font-medium">Charlie</div>
        <div class="bg-white/20 rounded-xl py-3 text-center font-medium">David</div>
      </div>
    </div>

    <!-- Host Controls -->
    <div class="space-y-4">
      <label class="flex items-center justify-between bg-white/10 px-4 py-2 rounded-xl cursor-pointer">
        <span>Show Role Initially</span>
        <input type="checkbox" class="w-5 h-5 accent-green-500" />
      </label>
      <label class="flex items-center justify-between bg-white/10 px-4 py-2 rounded-xl cursor-pointer">
        <span>Include Dumb Role</span>
        <input type="checkbox" class="w-5 h-5 accent-red-500" />
      </label>
    </div>

    <!-- Action Buttons -->
    <div class="flex justify-between gap-4 pt-4">
      <button
        class="flex-1 bg-gray-500 hover:bg-gray-600 text-white font-semibold py-3 rounded-xl shadow-lg transition">
        Leave
      </button>
      <button
        class="flex-1 bg-green-500 hover:bg-green-600 text-white font-semibold py-3 rounded-xl shadow-lg transition">
        Start Game
      </button>
    </div>
  </div>
</div>


<!-- Game Screen -->
<div class="min-h-screen flex flex-col items-center justify-center bg-gradient-to-br from-green-500 to-red-500 text-white p-6">

  <!-- Title -->
  <h1 class="text-3xl md:text-4xl font-extrabold mb-6 drop-shadow-lg">
    Game On! 🎭
  </h1>

  <!-- Game Card -->
  <div class="bg-white/10 backdrop-blur-lg rounded-2xl shadow-2xl p-8 w-full max-w-lg text-center space-y-6">

    <!-- Player Info -->
    <div class="space-y-2">
      <p class="text-sm opacity-80">You are</p>
      <p class="text-2xl font-bold">Player: Alice</p>
    </div>

    <!-- Reveal Button -->
    <div>
      <button
        class="w-full bg-green-500 hover:bg-green-600 text-white font-semibold py-3 rounded-xl shadow-lg transition">
        Reveal Word
      </button>
    </div>

    <!-- Word Display (placeholder for reveal) -->
    <div class="bg-white/20 rounded-xl py-6 mt-4 text-xl font-semibold tracking-wide">
      🍎 Apple
    </div>

    <!-- Progress -->
    <div class="mt-6">
      <p class="text-sm opacity-80 mb-2">Players Revealed</p>
      <div class="w-full bg-white/20 rounded-full h-3">
        <div class="bg-green-500 h-3 rounded-full" style="width: 60%;"></div>
      </div>
      <p class="text-xs mt-1">3 / 5</p>
    </div>

    <!-- Host Controls -->
    <div class="flex justify-between gap-4 pt-4">
      <button
        class="flex-1 bg-red-500 hover:bg-red-600 text-white font-semibold py-2 rounded-xl shadow-lg transition">
        End Game
      </button>
      <button
        class="flex-1 bg-green-500 hover:bg-green-600 text-white font-semibold py-2 rounded-xl shadow-lg transition">
        Next Round
      </button>
    </div>
  </div>
</div>

