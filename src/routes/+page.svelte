<script lang="ts">
  import { goto } from '$app/navigation';
  import Nav from './components/Nav.svelte';
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

  const gameOptions = [
    {
      id: 'same-device',
      title: 'Same Device Game',
      description: 'Pass the device around - perfect for gatherings!',
      icon: DeviceIcon,
      route: '/onedevice',
      gradient: 'from-blue-500 to-purple-600',
      hoverGradient: 'from-blue-600 to-purple-700',
      bgAccent: 'bg-blue-50',
      borderAccent: 'border-blue-200',
      textAccent: 'text-blue-600',
      shadowColor: 'shadow-blue-500/25',
      features: ['Pass & play', 'Instant setup', 'No internet needed']
    },
    // {
    //   id: 'different-device',
    //   title: 'Different Device Game',
    //   description: 'Everyone uses their own device with room codes',
    //   icon: WifiIcon,
    //   route: '/differentdevice',
    //   gradient: 'from-yellow-500 to-orange-600',
    //   hoverGradient: 'from-yellow-600 to-orange-700',
    //   bgAccent: 'bg-green-50',
    //   borderAccent: 'border-yellow-200',
    //   textAccent: 'text-yellow-600',
    //   shadowColor: 'shadow-yellow-500/25',
    //   features: ['Room codes', 'Own devices', 'Remote play']
    // },
    {
      id: 'online-game',
      title: 'Online Game',
      description: 'Coming soon - play with friends anywhere!',
      icon: UsersIcon,
      route: '/online',
      gradient: 'from-green-400 to-teal-600',
      hoverGradient: 'from-green-400 to-teal-700',
      bgAccent: 'bg-gray-50',
      borderAccent: 'border-green-200',
      textAccent: 'text-green-500',
      shadowColor: 'shadow-green-500/25',
      features: ['Global play', 'Friend matching', 'Coming soon'],
      // disabled: true
    }
  ];

  function selectGameMode(option: any) {
    if (option.disabled) return;
    goto(option.route);
  }
</script>
<Nav/>

<div class="min-h-screen bg-gradient-to-br from-gray-50 via-white to-gray-100">
  <div class="container mx-auto px-4 py-8 md:py-16">
    <div class="max-w-4xl mx-auto">
      
      <!-- Header -->
      <div class="text-center mt-16 mb-16">
        <h1 class="text-5xl md:text-6xl font-black text-gray-900 mb-4">
          Who's the Spy?
        </h1>
        <p class="text-xl text-gray-600 max-w-2xl mx-auto leading-relaxed">
          The ultimate party game of deception and deduction. Choose how you want to play!
        </p>
      </div>
      
      <!-- Game Mode Selection -->
      <div class="grid gap-8 md:grid-cols-2">
        {#each gameOptions as option}
          <div 
            class="group relative transform transition-all duration-300 hover:scale-105 {option.disabled ? 'cursor-not-allowed opacity-75' : 'cursor-pointer'}"
            on:click={() => selectGameMode(option)}
            on:keydown={(e) => e.key === 'Enter' && selectGameMode(option)}
            role="button"
            tabindex="0"
          >
            <!-- Card -->
            <div class="bg-white rounded-2xl border-2 {option.borderAccent} shadow-xl group-hover:shadow-2xl group-hover:{option.shadowColor} transition-all duration-300 p-8 h-full flex flex-col">
              
              <!-- Icon & Title -->
              <div class="text-center mb-6">
                <div class="inline-flex items-center justify-center w-16 h-16 bg-gradient-to-br {option.gradient} group-hover:{option.hoverGradient} rounded-xl mb-4 shadow-lg transition-all duration-300">
                  <div class="w-8 h-8 text-white">{@html option.icon}</div>
                </div>
                <h3 class="text-2xl font-black text-gray-900 mb-2">
                  {option.title}
                </h3>
                <p class="text-gray-600 text-sm leading-relaxed">
                  {option.description}
                </p>
              </div>
              
              <!-- Features -->
              <div class="flex-grow">
                <div class="space-y-3">
                  {#each option.features as feature}
                    <div class="flex items-center space-x-3">
                      <div class="w-2 h-2 bg-gradient-to-r {option.gradient} rounded-full flex-shrink-0"></div>
                      <span class="text-sm text-gray-600 font-medium">{feature}</span>
                    </div>
                  {/each}
                </div>
              </div>
              
              <!-- Action Button -->
              <div class="mt-8 pt-6 border-t border-gray-100">
                {#if option.disabled}
                  <div class="w-full bg-gray-100 text-gray-500 font-bold py-4 px-6 rounded-xl text-center">
                    Coming Soon
                  </div>
                {:else}
                  <div class="w-full bg-gradient-to-r {option.gradient} group-hover:{option.hoverGradient} text-white font-bold py-4 px-6 rounded-xl text-center transition-all duration-300 shadow-lg group-hover:shadow-xl">
                    Play Now
                  </div>
                {/if}
              </div>
            </div>
            
            <!-- Hover Effect Overlay -->
            {#if !option.disabled}
              <div class="absolute inset-0 bg-gradient-to-r {option.gradient} opacity-0 group-hover:opacity-5 rounded-2xl transition-all duration-300"></div>
            {/if}
          </div>
        {/each}
      </div>
      
      <!-- Game Info -->
      <div class="mt-16 text-center">
        <div class="bg-white/80 backdrop-blur-sm rounded-2xl border border-gray-200 p-8 shadow-xl">
          <h2 class="text-2xl font-black text-gray-900 mb-4">How to Play</h2>
          <div class="grid gap-6 md:grid-cols-3 text-left">
            <div class="flex flex-col items-center text-center">
              <div class="w-12 h-12 bg-gradient-to-br from-red-500 to-pink-600 rounded-xl flex items-center justify-center text-white font-black text-xl mb-3">1</div>
              <h3 class="font-bold text-gray-900 mb-2">Get Your Words</h3>
              <p class="text-sm text-gray-600">Everyone gets a secret word. The spy gets a different word!</p>
            </div>
            <div class="flex flex-col items-center text-center">
              <div class="w-12 h-12 bg-gradient-to-br from-red-500 to-pink-600 rounded-xl flex items-center justify-center text-white font-black text-xl mb-3">2</div>
              <h3 class="font-bold text-gray-900 mb-2">Discuss & Deduce</h3>
              <p class="text-sm text-gray-600">Talk about your words without being too obvious!</p>
            </div>
            <div class="flex flex-col items-center text-center">
              <div class="w-12 h-12 bg-gradient-to-br from-red-500 to-pink-600 rounded-xl flex items-center justify-center text-white font-black text-xl mb-3">3</div>
              <h3 class="font-bold text-gray-900 mb-2">Find the Spy</h3>
              <p class="text-sm text-gray-600">Vote to eliminate who you think is the spy!</p>
            </div>
          </div>
        </div>
      </div>
      
      <!-- Footer -->
      <div class="mt-12 text-center">
        <p class="text-gray-500 text-sm">
          Made for fun times with friends • Supports Nepali & English
        </p>
      </div>
    </div>
  </div>
</div>

<style>
  /* Add some subtle animations */
  @keyframes float {
    0%, 100% { transform: translateY(0px); }
    50% { transform: translateY(-5px); }
  }
  
  .group:hover .float {
    animation: float 2s ease-in-out infinite;
  }
  
  /* Custom scrollbar if needed */
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
</style>