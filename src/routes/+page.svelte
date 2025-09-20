<script lang="ts">
  import { goto } from '$app/navigation';
  // import Nav from './components/Nav.svelte';
  import { injectSpeedInsights } from '@vercel/speed-insights/sveltekit';

  injectSpeedInsights();

  
  const DeviceIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <rect x="2" y="3" width="20" height="14" rx="2" ry="2"/>
    <line x1="8" y1="21" x2="16" y2="21"/>
    <line x1="12" y1="17" x2="12" y2="21"/>
  </svg>`;
  
  const WifiIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <path d="M12 20h.01"/>
    <path d="M8.5 16.429a5 5 0 0 1 7 0"/>
    <path d="M5 12.859a10 10 0 0 1 14 0"/>
    <path d="M1.42 9.289a15 15 0 0 1 21.16 0"/>
  </svg>`;
  
  const UsersIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <path d="M17 21v-2a4 4 0 0 0-4-4H5a4 4 0 0 0-4 4v2"/>
    <circle cx="9" cy="7" r="4"/>
    <path d="M23 21v-2a4 4 0 0 0-3-3.87"/>
    <circle cx="16" cy="7" r="3"/>
  </svg>`;

  const PlayIcon = `<svg viewBox="0 0 24 24" fill="currentColor">
    <path d="M8 5v14l11-7z"/>
  </svg>`;

const gameOptions = [
  {
    id: 'same-device',
    title: 'Pass & Play',
    subtitle: 'Same Device Game',
    description: 'Perfect for gatherings - pass one device around the group',
    icon: DeviceIcon,
    route: '/onedevice',
    gradient: 'from-blue-500 to-indigo-600',
    hoverGradient: 'hover:from-blue-400 hover:to-indigo-500',
    glowColor: 'shadow-blue-500/30',
    features: ['One device only', 'Pass around', 'Instant setup', 'Local multiplayer'],
    status: 'available',
    statusText: 'Ready to Play'
  },
  {
    id: 'online-game',
    title: 'Online Play',
    subtitle: 'Remote Game',
    description: 'Play with friends anywhere in the world',
    icon: UsersIcon,
    route: '/online',
    gradient: 'from-emerald-500 to-teal-600',
    hoverGradient: 'hover:from-emerald-400 hover:to-teal-500',
    glowColor: 'shadow-emerald-500/30',
    features: ['Room codes', 'Own devices', 'Remote friends', 'Real-time sync'],
    status: 'available',
    statusText: 'Ready to Play'
  },
  {
    id: 'team-battle',
    title: 'Team Battle',
    subtitle: 'Cooperative Mode',
    description: 'Teams work together to identify the opposing team\'s spy',
    icon: `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
      <path d="M16 21v-2a4 4 0 0 0-4-4H6a4 4 0 0 0-4 4v2"/>
      <circle cx="9" cy="7" r="4"/>
      <path d="M22 21v-2a4 4 0 0 0-3-3.87"/>
      <circle cx="16" cy="7" r="3"/>
      <path d="M12 12h.01"/>
      <path d="M8 8h.01"/>
      <path d="M16 8h.01"/>
    </svg>`,
    route: '/team-battle',
    gradient: 'from-yellow-500 to-orange-600',
    hoverGradient: 'hover:from-yellow-400 hover:to-orange-500',
    glowColor: 'shadow-yellow-500/30',
    features: ['Team vs team', 'Collaboration', '4v4 or 3v3', 'Strategy focus'],
    status: 'available',
    statusText: 'Ready to Play'
  },
  {
    id: 'speed-rounds',
    title: 'Speed Rounds',
    subtitle: 'Fast-Paced Mode',
    description: 'Quick 60-second rounds for intense, rapid-fire gameplay',
    icon: `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
      <circle cx="12" cy="12" r="10"/>
      <polyline points="12,6 12,12 16,14"/>
      <path d="M12 2v4M12 18v4M4.93 4.93l2.83 2.83M16.24 16.24l2.83 2.83M2 12h4M18 12h4M4.93 19.07l2.83-2.83M16.24 7.76l2.83-2.83"/>
    </svg>`,
    route: '/speed',
    gradient: 'from-orange-500 to-red-600',
    hoverGradient: 'hover:from-orange-400 hover:to-red-500',
    glowColor: 'shadow-orange-500/30',
    features: ['60-second timer', 'Quick decisions', 'Multiple rounds', 'High intensity'],
    status: 'coming-soon',
    statusText: 'Coming Soon'
  },
  {
    id: 'tournament',
    title: 'Tournament',
    subtitle: 'Competitive Mode',
    description: 'Bracket-style competition with scoring and elimination',
    icon: `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
      <path d="M6 9H4.5a2.5 2.5 0 0 1 0-5H6"/>
      <path d="M18 9h1.5a2.5 2.5 0 0 0 0-5H18"/>
      <path d="M4 22h16"/>
      <path d="M10 14.66V17c0 .55-.47.98-.97 1.21C7.85 18.75 7 20.24 7 22"/>
      <path d="M14 14.66V17c0 .55.47.98.97 1.21C16.15 18.75 17 20.24 17 22"/>
      <path d="M18 2H6v7a6 6 0 0 0 12 0V2Z"/>
    </svg>`,
    route: '/tournament',
    gradient: 'from-purple-500 to-pink-600',
    hoverGradient: 'hover:from-purple-400 hover:to-pink-500',
    glowColor: 'shadow-purple-500/30',
    features: ['Bracket system', 'Point scoring', 'Leaderboards', 'Multiple rounds'],
    status: 'coming-soon',
    statusText: 'Coming Soon'
  },

];

  function selectGameMode(option: any) {
    if (option.status === 'coming-soon') return;
    goto(option.route);
  }
</script>


<div class="min-h-screen bg-gradient-to-br from-slate-900 via-purple-900 to-slate-900 relative overflow-hidden">
  <!-- Animated Background Elements -->
  <div class="absolute inset-0 overflow-hidden pointer-events-none">
    <div class="absolute -top-20 -right-20 w-40 h-40 bg-purple-500/10 rounded-full blur-2xl animate-pulse"></div>
    <div class="absolute -bottom-20 -left-20 w-40 h-40 bg-blue-500/10 rounded-full blur-2xl animate-pulse" style="animation-delay: 2s;"></div>
    <div class="absolute top-1/2 left-1/2 transform -translate-x-1/2 -translate-y-1/2 w-48 h-48 bg-indigo-500/5 rounded-full blur-2xl animate-pulse" style="animation-delay: 4s;"></div>
  </div>

  <!-- Main Content -->
  <div class="relative z-10 min-h-screen">
    <div class="container mx-auto ">
      <div class="max-w-6xl mx-auto">
        
        <!-- Header -->
        <div class="text-center mt-16 mb-16">

          
          <h1 class="text-5xl md:text-6xl font-black text-white mb-6 tracking-tight">
            Who's the <span class="bg-gradient-to-r from-purple-400 to-pink-400 bg-clip-text text-transparent">Word Spy?</span>
          </h1>
          <p class="text-md md:text-lg text-slate-300 max-w-3xl mx-auto leading-relaxed mb-8">
            The ultimate party game of deception and deduction. Choose your game mode to start the fun!
          </p>
          
          <!-- Quick Stats -->
        </div>
        
        <!-- Game Mode Selection -->
        <div class="grid gap-8 md:grid-cols-2 max-w-4xl mx-auto mb-16">
          {#each gameOptions as option}
            <div 
              class="group relative transform transition-all duration-300 {option.status === 'coming-soon' ? 'cursor-not-allowed' : 'cursor-pointer hover:scale-105'}"
              on:click={() => selectGameMode(option)}
              on:keydown={(e) => e.key === 'Enter' && selectGameMode(option)}
              role="button"
              tabindex="0"
              aria-label="{option.title} - {option.description}"
              >
              <!-- Card -->
              <div class="relative bg-white/5 backdrop-blur-xl border border-white/10 rounded-2xl p-8 h-full flex flex-col transition-all duration-300 group-hover:bg-white/10 group-hover:border-white/20 {option.status === 'available' ? 'group-hover:shadow-2xl group-hover:' + option.glowColor : ''}">
                
                <!-- Status Badge -->
                <div class="absolute top-4 right-4">
                  {#if option.status === 'available'}
                  <div class="flex items-center gap-1 px-3 py-1 bg-green-500/20 border border-green-400/30 rounded-full">
                      <div class="w-1.5 h-1.5 bg-green-400 rounded-full animate-pulse"></div>
                      <span class="text-green-300 text-xs font-medium">Available</span>
                    </div>
                    {:else}
                    <div class="flex items-center gap-1 px-3 py-1 bg-amber-500/20 border border-amber-400/30 rounded-full">
                      <div class="w-1.5 h-1.5 bg-amber-400 rounded-full"></div>
                      <span class="text-amber-300 text-xs font-medium">Coming Soon</span>
                    </div>
                  {/if}
                </div>
                
                <!-- Icon & Title -->
                <div class="text-center mb-6">
                  <div class="inline-flex items-center justify-center w-20 h-20 bg-gradient-to-br {option.gradient} rounded-2xl mb-4 shadow-lg transition-all duration-300 group-hover:scale-110 {option.status === 'available' ? 'group-hover:shadow-xl' : ''}">
                    <div class="w-10 h-10 text-white">{@html option.icon}</div>
                  </div>
                  <div class="space-y-2">
                    <h3 class="text-3xl font-black text-white mb-1">
                      {option.title}
                    </h3>
                    <p class="text-lg font-medium text-slate-300">
                      {option.subtitle}
                    </p>
                    <p class="text-slate-400 leading-relaxed max-w-sm mx-auto">
                      {option.description}
                    </p>
                  </div>
                </div>
                
                <!-- Features -->
                <div class="flex-grow mb-6">
                  <div class="grid grid-cols-2 gap-3">
                    {#each option.features as feature, i}
                    <div class="flex items-center space-x-2 text-sm">
                      <div class="w-2 h-2 bg-gradient-to-r {option.gradient} rounded-full flex-shrink-0"></div>
                      <span class="text-slate-300 font-medium">{feature}</span>
                    </div>
                    {/each}
                  </div>
                </div>
                
                <!-- Action Button -->
                <div class="mt-auto">
                  {#if option.status === 'coming-soon'}
                  <div class="w-full bg-white/10 text-slate-400 font-bold py-4 px-6 rounded-xl text-center border border-white/20 cursor-not-allowed">
                    <div class="flex items-center justify-center gap-2">
                        <svg class="w-4 h-4" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                          <circle cx="12" cy="12" r="10"/>
                          <polyline points="12,6 12,12 16,14"/>
                        </svg>
                        Coming Soon
                      </div>
                    </div>
                    {:else}
                    <button class="w-full bg-gradient-to-r {option.gradient} {option.hoverGradient} text-white font-bold py-4 px-6 rounded-xl text-center transition-all duration-300 shadow-lg hover:shadow-xl group-hover:scale-105 active:scale-95 focus:outline-none focus:ring-2 focus:ring-purple-500 focus:ring-offset-2 focus:ring-offset-slate-900">
                      <div class="flex items-center justify-center gap-2">
                        <div class="w-5 h-5">{@html PlayIcon}</div>
                        <span class="text-lg">Start Playing</span>
                      </div>
                    </button>
                  {/if}
                </div>
              </div>
              
              <!-- Hover Effect Overlay -->
              {#if option.status === 'available'}
              <div class="absolute inset-0 bg-gradient-to-r {option.gradient} opacity-0 group-hover:opacity-5 rounded-2xl transition-all duration-300 pointer-events-none"></div>
              {/if}
            </div>
            {/each}
          </div>
          <div class="flex justify-center gap-8 mb-8">
            <div class="text-center">
              <div class="text-2xl font-bold text-white">3-12</div>
              <div class="text-sm text-slate-400">Players</div>
            </div>
            <div class="text-center">
              <div class="text-2xl font-bold text-white">5-15</div>
              <div class="text-sm text-slate-400">Minutes</div>
            </div>
            <div class="text-center">
              <div class="text-2xl font-bold text-white">16+</div>
              <div class="text-sm text-slate-400">Languages</div>
            </div>
          </div>
        
        <!-- Game Info -->
        <div class="text-center">
          <div class="bg-white/5 backdrop-blur-xl border border-white/10 rounded-2xl p-8 shadow-xl">
            <div class="flex items-center justify-center gap-2 mb-6">
              <svg class="w-6 h-6 text-purple-400" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M9.09 9a3 3 0 0 1 5.83 1c0 2-3 3-3 3"/>
                <circle cx="12" cy="17" r="1"/>
                <circle cx="12" cy="12" r="10"/>
              </svg>
              <h2 class="text-3xl font-black text-white">How to Play</h2>
            </div>
            
            <div class="grid gap-8 md:grid-cols-3 text-center">
              <div class="group">
                <div class="w-16 h-16 bg-gradient-to-br from-red-500 to-pink-600 rounded-2xl flex items-center justify-center text-white font-black text-2xl mb-4 mx-auto group-hover:scale-110 transition-transform shadow-lg">1</div>
                <h3 class="text-xl font-bold text-white mb-3">Get Secret Words</h3>
                <p class="text-slate-300 leading-relaxed">Everyone receives a secret word, but the spy gets a different one! Stay alert and don't reveal too much.</p>
              </div>
              
              <div class="group">
                <div class="w-16 h-16 bg-gradient-to-br from-blue-500 to-indigo-600 rounded-2xl flex items-center justify-center text-white font-black text-2xl mb-4 mx-auto group-hover:scale-110 transition-transform shadow-lg">2</div>
                <h3 class="text-xl font-bold text-white mb-3">Discuss & Deduce</h3>
                <p class="text-slate-300 leading-relaxed">Talk about your words carefully. Give enough hints to prove you're not the spy, but not too much!</p>
              </div>
              
              <div class="group">
                <div class="w-16 h-16 bg-gradient-to-br from-emerald-500 to-teal-600 rounded-2xl flex items-center justify-center text-white font-black text-2xl mb-4 mx-auto group-hover:scale-110 transition-transform shadow-lg">3</div>
                <h3 class="text-xl font-bold text-white mb-3">Find the Spy</h3>
                <p class="text-slate-300 leading-relaxed">Vote to eliminate who you think is the spy. Can you catch them before they figure out the civilian word?</p>
              </div>
            </div>
            
            <!-- Additional Game Features -->
            <div class="mt-8 pt-6 border-t border-white/20">
              <div class="flex flex-wrap justify-center gap-4 text-sm text-slate-400">
                <div class="flex items-center gap-2">
                  <div class="w-2 h-2 bg-purple-400 rounded-full"></div>
                  <span>16+ Languages Supported</span>
                </div>
                <div class="flex items-center gap-2">
                  <div class="w-2 h-2 bg-blue-400 rounded-full"></div>
                  <span>Custom Themes Available</span>
                </div>
                <div class="flex items-center gap-2">
                  <div class="w-2 h-2 bg-emerald-400 rounded-full"></div>
                  <span>Mystery Player Mode</span>
                </div>
              </div>
            </div>
          </div>
        </div>
        
        <!-- Footer -->
        <div class="mt-12 text-center">
          <div class="inline-flex items-center gap-2 text-slate-500 text-sm">
            <span>Made for fun times with friends</span>
            <div class="w-1 h-1 bg-slate-500 rounded-full"></div>
            <span>Open source party game</span>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>

<style>
  /* Enhanced animations */
  @keyframes float {
    0%, 100% { transform: translateY(0px); }
    50% { transform: translateY(-8px); }
  }
  
  .group:hover .animate-float {
    animation: float 3s ease-in-out infinite;
  }

  /* Improved focus styles for accessibility */
  button:focus-visible,
  [role="button"]:focus-visible {
    outline: 2px solid #8b5cf6;
    outline-offset: 2px;
    border-radius: 1rem;
  }

  /* Custom scrollbar matching game theme */
  :global(.overflow-y-auto::-webkit-scrollbar) {
    width: 8px;
  }
 
  :global(.overflow-y-auto::-webkit-scrollbar-track) {
    background: rgba(255, 255, 255, 0.05);
    border-radius: 10px;
  }
 
  :global(.overflow-y-auto::-webkit-scrollbar-thumb) {
    background: linear-gradient(to bottom, #8b5cf6, #ec4899);
    border-radius: 10px;
  }

  :global(.overflow-y-auto::-webkit-scrollbar-thumb:hover) {
    background: linear-gradient(to bottom, #7c3aed, #db2777);
  }

  /* Hover glow effects */
  .hover-glow:hover {
    box-shadow: 0 0 30px rgba(139, 92, 246, 0.3);
  }

  /* Ensure text contrast for accessibility */
  .text-contrast {
    text-shadow: 0 1px 2px rgba(0, 0, 0, 0.5);
  }

  /* Button press animation */
  button:active,
  [role="button"]:active {
    transform: scale(0.95);
    transition: transform 0.1s;
  }
</style>