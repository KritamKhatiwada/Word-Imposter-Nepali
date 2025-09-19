<script>
//   import { page } from '$app/stores';
</script>

<nav class="fixed top-4 left-1/2 -translate-x-1/2 bg-white/70 backdrop-blur-md 
            rounded-2xl shadow-lg px-5 py-2 flex gap-4 items-center justify-center
            transition-all duration-300 z-50">
  <a href="/"
     class="flex flex-col items-center text-xs">
    <span class="material-icons text-lg">home</span>
    Home
  </a>

  <a href="/onedevice"
     class="flex flex-col items-center text-xs">
    <span class="material-icons text-lg">devices</span>
    One Device
  </a>

  <a href="/online"
     class="flex flex-col items-center text-xs">
    <span class="material-icons text-lg">cloud</span>
    Online
  </a>
</nav>

<style>
  a {
    transition: transform 0.2s, color 0.2s;
  }
  a:hover {
    transform: scale(1.1);
    color: green;
  }
</style>
