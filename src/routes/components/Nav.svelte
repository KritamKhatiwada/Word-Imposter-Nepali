<script>
  import { page } from '$app/stores';
  
  // Reactive statement to get current path
  $: currentPath = $page.url.pathname;
  
  // Helper function to check if path is active
  function isActive(path) {
    return currentPath === path;
  }
</script>

<nav class="fixed top-6 left-1/2 transform -translate-x-1/2 z-50 transition-all duration-300 hover:scale-105">
  <div class="nav-container backdrop-blur-xl border border-white/20 rounded-2xl shadow-2xl px-6 py-3">
    <div class="flex items-center gap-6">
      
      <a href="/" 
         class="nav-link group relative flex flex-col items-center gap-1 px-4 py-2 rounded-xl transition-all duration-300 hover:bg-white/10"
         class:active={isActive('/')}>
        <div class="relative">
          <svg class="w-5 h-5 transition-transform group-hover:scale-110" fill="currentColor" viewBox="0 0 24 24">
            <path d="M10 20v-6h4v6h5v-8h3L12 3 2 12h3v8z"/>
          </svg>
          {#if isActive('/')}
            <div class="absolute -bottom-1 left-1/2 transform -translate-x-1/2 w-1 h-1 bg-purple-400 rounded-full"></div>
          {/if}
        </div>
        <span class="text-xs font-medium">Home</span>
        
        <!-- Hover glow effect -->
        <div class="absolute inset-0 rounded-xl bg-gradient-to-r from-purple-500/20 to-pink-500/20 opacity-0 group-hover:opacity-100 transition-opacity -z-10"></div>
      </a>

      <a href="/onedevice" 
         class="nav-link group relative flex flex-col items-center gap-1 px-4 py-2 rounded-xl transition-all duration-300 hover:bg-white/10"
         class:active={isActive('/onedevice')}>
        <div class="relative">
          <svg class="w-5 h-5 transition-transform group-hover:scale-110" fill="currentColor" viewBox="0 0 24 24">
            <path d="M4 6h18V4H4c-1.1 0-2 .9-2 2v11H0v3h14v-3H4V6zm19 2h-6c-.55 0-1 .45-1 1v10c0 .55.45 1 1 1h6c.55 0 1-.45 1-1V9c0-.55-.45-1-1-1zm-1 9h-4v-7h4v7z"/>
          </svg>
          {#if isActive('/onedevice')}
            <div class="absolute -bottom-1 left-1/2 transform -translate-x-1/2 w-1 h-1 bg-purple-400 rounded-full"></div>
          {/if}
        </div>
        <span class="text-xs font-medium">Offline</span>
        
        <!-- Hover glow effect -->
        <div class="absolute inset-0 rounded-xl bg-gradient-to-r from-blue-500/20 to-cyan-500/20 opacity-0 group-hover:opacity-100 transition-opacity -z-10"></div>
      </a>

      <a href="/online" 
         class="nav-link group relative flex flex-col items-center gap-1 px-4 py-2 rounded-xl transition-all duration-300 hover:bg-white/10"
         class:active={isActive('/online')}>
        <div class="relative">
          <svg class="w-5 h-5 transition-transform group-hover:scale-110" fill="currentColor" viewBox="0 0 24 24">
            <path d="M19.35 10.04A7.49 7.49 0 0 0 12 4C9.11 4 6.6 5.64 5.35 8.04A5.994 5.994 0 0 0 0 14c0 3.31 2.69 6 6 6h13c2.76 0 5-2.24 5-5 0-2.64-2.05-4.78-4.65-4.96z"/>
          </svg>
          {#if isActive('/online')}
            <div class="absolute -bottom-1 left-1/2 transform -translate-x-1/2 w-1 h-1 bg-purple-400 rounded-full"></div>
          {/if}
        </div>
        <span class="text-xs font-medium">Online</span>
        
        <!-- Hover glow effect -->
        <div class="absolute inset-0 rounded-xl bg-gradient-to-r from-green-500/20 to-emerald-500/20 opacity-0 group-hover:opacity-100 transition-opacity -z-10"></div>
      </a>
    </div>
  </div>

  <!-- Subtle animated background -->
  <div class="absolute inset-0 bg-gradient-to-r from-purple-500/5 via-pink-500/5 to-blue-500/5 rounded-2xl blur-xl -z-20"></div>
</nav>

<style>
  /* Base nav container styles - adapts to page background */
  .nav-container {
    background: rgba(255, 255, 255, 0.1);
    backdrop-filter: blur(20px);
    -webkit-backdrop-filter: blur(20px);
  }
  
  /* Default nav link styles for dark backgrounds */
  .nav-link {
    color: rgba(255, 255, 255, 0.7);
    transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  }
  
  .nav-link:hover {
    color: rgba(255, 255, 255, 1);
    transform: translateY(-1px);
  }
  
  .nav-link.active {
    background: rgba(255, 255, 255, 0.2);
    color: rgba(255, 255, 255, 1);
  }

  /* Light theme / white background adaptation */
  @media (prefers-color-scheme: light) {
    .nav-container {
      background: rgba(255, 255, 255, 0.9);
      border-color: rgba(0, 0, 0, 0.1);
      box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.25);
    }
    
    .nav-link {
      color: rgba(0, 0, 0, 0.7);
    }
    
    .nav-link:hover {
      color: rgba(0, 0, 0, 1);
      background: rgba(0, 0, 0, 0.05);
    }
    
    .nav-link.active {
      background: rgba(0, 0, 0, 0.1);
      color: rgba(0, 0, 0, 1);
    }
  }

  /* Force light theme for white page backgrounds */
  :global(.page-white) .nav-container {
    background: rgba(255, 255, 255, 0.9);
    border-color: rgba(0, 0, 0, 0.1);
    box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.25);
  }
  
  :global(.page-white) .nav-link {
    color: rgba(0, 0, 0, 0.7);
  }
  
  :global(.page-white) .nav-link:hover {
    color: rgba(0, 0, 0, 1);
    background: rgba(0, 0, 0, 0.05);
  }
  
  :global(.page-white) .nav-link.active {
    background: rgba(0, 0, 0, 0.1);
    color: rgba(0, 0, 0, 1);
  }

  /* Custom focus styles */
  .nav-link:focus-visible {
    outline: 2px solid rgba(139, 92, 246, 0.8);
    outline-offset: 2px;
    border-radius: 0.75rem;
  }

  /* Active state */
  .nav-link:active {
    transform: translateY(0);
    transition-duration: 0.1s;
  }

  /* Ensure text is always readable */
  .nav-link span {
    text-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
  }

  /* Responsive design */
  @media (max-width: 640px) {
    nav {
      top: 1rem;
    }
    
    .nav-container {
      padding: 0.5rem 1rem;
    }
    
    .nav-link {
      padding: 0.5rem 0.75rem;
      gap: 0.25rem;
    }
    
    .nav-link svg {
      width: 1.125rem;
      height: 1.125rem;
    }
    
    .nav-link span {
      font-size: 0.6875rem;
    }
  }
</style>