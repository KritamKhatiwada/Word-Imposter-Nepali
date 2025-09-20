<script lang="ts">
	import { onMount } from 'svelte';
	import { GoogleGenerativeAI } from '@google/generative-ai';
	import Nav from '../components/Nav.svelte';
	interface Player {
		name: string;
		role: 'imposter' | 'civilian';
		word: string;
		team: 'A' | 'B';
	}
	interface GameData {
		players: Player[];
		teamAWord: string;
		teamBWord: string;
		teamAImposterWord: string;
		teamBImposterWord: string;
		theme?: string;
	}
	interface WordPair {
		civilian: string;
		imposter: string;
	}
	// Store used word pairs to prevent repetition
	let usedWordPairs: Set<string> = new Set();
	let currentSessionPairs: WordPair[] = [];
	let usedThemes: Set<string> = new Set();
	let gameState: 'home' | 'setup' | 'game' | 'complete' | 'reveal' = 'home';
	let teamASize: number = 3;
	let teamBSize: number = 3;
	let teamANames: string[] = [''];
	let teamBNames: string[] = [''];
	let savedTeamANames: string[] = [];
	let savedTeamBNames: string[] = [];
	let currentPlayerIndex: number = 0;
	let currentTeam: 'A' | 'B' = 'A';
	let gameData: GameData | null = null;
	let showRoleInitially: boolean = false;
	let wordRevealed: boolean = false;
	let isGeneratingWords: boolean = false;
	let selectedLanguage: string = 'english';
	let showLanguageDropdown: boolean = false;
	let selectedTheme: string = 'random';
	let showThemeInput: boolean = false;
	// Team battle specific state
	let teamPlayOrder: Player[] = [];
	let discussionPhase: boolean = false;
	let votingPhase: boolean = false;
	let teamAScore: number = 0;
	let teamBScore: number = 0;
	let roundNumber: number = 1;
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
	const UsersIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <path d="M17 21v-2a4 4 0 0 0-4-4H5a4 4 0 0 0-4 4v2"/>
    <circle cx="9" cy="7" r="4"/>
    <path d="M23 21v-2a4 4 0 0 0-3-3.87"/>
    <circle cx="16" cy="7" r="3"/>
  </svg>`;
	const TrophyIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <path d="M6 9H4.5a2.5 2.5 0 0 1 0-5H6"/>
    <path d="M18 9h1.5a2.5 2.5 0 0 0 0-5H18"/>
    <path d="M4 22h16"/>
    <path d="M10 14.66V17c0 .55-.47.98-.97 1.21C7.85 18.75 7 20.24 7 22"/>
    <path d="M14 14.66V17c0 .55.47.98.97 1.21C16.15 18.75 17 20.24 17 22"/>
    <path d="M18 2H6v7a6 6 0 0 0 12 0V2Z"/>
  </svg>`;
	const languages = {
		english: { name: 'English', flag: '🇺🇸' },
		nepali: { name: 'नेपाली', flag: '🇳🇵' },
		hindi: { name: 'हिंदी', flag: '🇮🇳' },
		spanish: { name: 'Español', flag: '🇪🇸' },
		french: { name: 'Français', flag: '🇫🇷' },
		german: { name: 'Deutsch', flag: '🇩🇪' }
	};
	const availableThemes = {
		english: [
			'Animals',
			'Food',
			'Movies',
			'Countries',
			'Sports',
			'Colors',
			'Professions',
			'Music Genres',
			'Car Brands',
			'Board Games',
			'Fruits',
			'Vegetables'
		],
		nepali: ['जनावरहरू', 'खाना', 'चलचित्र', 'देशहरू', 'खेलहरू', 'रङहरू', 'पेशाहरू'],
		hindi: ['जानवर', 'खाना', 'फिल्में', 'देश', 'खेल', 'रंग', 'पेशे']
	};
	// Reactive statements to update team arrays
	$: {
		if (teamASize > 0) {
			teamANames = Array(teamASize)
				.fill('')
				.map((_, i: number) => teamANames[i] || '');
		}
		if (teamBSize > 0) {
			teamBNames = Array(teamBSize)
				.fill('')
				.map((_, i: number) => teamBNames[i] || '');
		}
	}
	onMount(() => {
		function handleClickOutside(event: MouseEvent) {
			if (showLanguageDropdown && !event.target?.closest('.language-dropdown')) {
				showLanguageDropdown = false;
			}
		}
		document.addEventListener('click', handleClickOutside);
		return () => document.removeEventListener('click', handleClickOutside);
	});
	function shuffleArray(array: any[]): any[] {
		const shuffled = [...array];
		for (let i = shuffled.length - 1; i > 0; i--) {
			const j = Math.floor(Math.random() * (i + 1));
			[shuffled[i], shuffled[j]] = [shuffled[j], shuffled[i]];
		}
		return shuffled;
	}
	function getRandomUnusedTheme(): string {
		const themes = availableThemes[selectedLanguage] || availableThemes.english;
		const unusedThemes = themes.filter((theme) => !usedThemes.has(theme.toLowerCase()));
		if (unusedThemes.length === 0) {
			usedThemes.clear();
			return themes[Math.floor(Math.random() * themes.length)];
		}
		return unusedThemes[Math.floor(Math.random() * unusedThemes.length)];
	}
	function createWordPairKey(civilian: string, imposter: string): string {
		return `${civilian.toLowerCase()}-${imposter.toLowerCase()}`;
	}
	function isRecentlyUsed(wordPair: WordPair): boolean {
		const key = createWordPairKey(wordPair.civilian, wordPair.imposter);
		return usedWordPairs.has(key);
	}
	function addToUsedPairs(wordPair: WordPair): void {
		const key = createWordPairKey(wordPair.civilian, wordPair.imposter);
		usedWordPairs.add(key);
		currentSessionPairs.push(wordPair);
		if (currentSessionPairs.length > 10) {
			const oldestPair = currentSessionPairs.shift();
			if (oldestPair) {
				const oldKey = createWordPairKey(oldestPair.civilian, oldestPair.imposter);
				usedWordPairs.delete(oldKey);
			}
		}
	}
	async function generateWithGemini(prompt: string): Promise<{ teamA: WordPair; teamB: WordPair }> {
		try {
			const genAI = new GoogleGenerativeAI(
				import.meta.env.VITE_REACT_APP_GEMINI_API_KEY || 'your-api-key-here'
			);
			const models = [
				'gemini-2.0-flash-exp',
				'gemini-2.0-flash-lite',
				'gemini-2.0-flash',
				'gemini-2.5-flash'
			];
			let text = null;
			for (const modelName of models) {
				try {
					const model = genAI.getGenerativeModel({ model: modelName });
					const result = await model.generateContent(prompt);
					const response = await result.response;
					text = await response.text();
					if (text) break;
				} catch (err) {
					console.warn(`Failed with ${modelName}:`, err.message);
				}
			}
			if (!text) throw new Error('All models failed');
			return parseTeamJSON(text);
		} catch (error) {
			console.error('Error generating with Gemini:', error);
			const fallbacks = {
				english: {
					teamA: { civilian: 'Dog', imposter: 'Cat' },
					teamB: { civilian: 'Lion', imposter: 'Tiger' }
				}
			};
			return fallbacks[selectedLanguage] || fallbacks.english;
		}
	}
	function parseTeamJSON(text: string): { teamA: WordPair; teamB: WordPair } {
		try {
			const jsonMatch = text.match(/\{[\s\S]*\}/);
			if (jsonMatch) {
				return JSON.parse(jsonMatch[0]);
			}
			const fallbacks = {
				english: {
					teamA: { civilian: 'Dog', imposter: 'Cat' },
					teamB: { civilian: 'Lion', imposter: 'Tiger' }
				}
			};
			return fallbacks[selectedLanguage] || fallbacks.english;
		} catch (error) {
			const fallbacks = {
				english: {
					teamA: { civilian: 'Dog', imposter: 'Cat' },
					teamB: { civilian: 'Lion', imposter: 'Tiger' }
				}
			};
			return fallbacks[selectedLanguage] || fallbacks.english;
		}
	}
	async function startGame(): Promise<void> {
		if (teamASize < 2 || teamBSize < 2) {
			alert('Each team needs at least 2 players!');
			return;
		}
		savedTeamANames = [...teamANames];
		savedTeamBNames = [...teamBNames];
		isGeneratingWords = true;
		let actualTheme: string;
		if (selectedTheme.toLowerCase() === 'random' || !selectedTheme.trim()) {
			actualTheme = getRandomUnusedTheme();
		} else {
			actualTheme = selectedTheme;
		}
		usedThemes.add(actualTheme.toLowerCase());
		const prompt = `Generate word pairs for a team battle "Who is the Spy" game.
THEME: "${actualTheme}"
RULES:
- Generate TWO separate word pairs for Team A and Team B
- Both pairs must be from the "${actualTheme}" category
- Each pair should have a civilian word and an imposter word
- Make them challenging but fair
- Words should be in ${languages[selectedLanguage]?.name || 'English'}
Respond in JSON format:
{
  "teamA": {
    "civilian": "team_A_civilian_word",
    "imposter": "team_A_imposter_word"
  },
  "teamB": {
    "civilian": "team_B_civilian_word", 
    "imposter": "team_B_imposter_word"
  }
}`;
		const wordPairs = await generateWithGemini(prompt);
		addToUsedPairs(wordPairs.teamA);
		addToUsedPairs(wordPairs.teamB);
		// Create players
		const players: Player[] = [];
		// Team A players
		for (let i = 0; i < teamASize; i++) {
			const isImposter = i === Math.floor(Math.random() * teamASize);
			players.push({
				name: teamANames[i] || `Team A Player ${i + 1}`,
				role: isImposter ? 'imposter' : 'civilian',
				word: isImposter ? wordPairs.teamA.imposter : wordPairs.teamA.civilian,
				team: 'A'
			});
		}
		// Team B players
		for (let i = 0; i < teamBSize; i++) {
			const isImposter = i === Math.floor(Math.random() * teamBSize);
			players.push({
				name: teamBNames[i] || `Team B Player ${i + 1}`,
				role: isImposter ? 'imposter' : 'civilian',
				word: isImposter ? wordPairs.teamB.imposter : wordPairs.teamB.civilian,
				team: 'B'
			});
		}
		// Create alternating play order
		const teamAPlayers = players.filter((p) => p.team === 'A');
		const teamBPlayers = players.filter((p) => p.team === 'B');
		teamPlayOrder = [];
		const maxTeamSize = Math.max(teamAPlayers.length, teamBPlayers.length);
		for (let i = 0; i < maxTeamSize; i++) {
			if (teamAPlayers[i]) teamPlayOrder.push(teamAPlayers[i]);
			if (teamBPlayers[i]) teamPlayOrder.push(teamBPlayers[i]);
		}
		gameData = {
			players,
			teamAWord: wordPairs.teamA.civilian,
			teamBWord: wordPairs.teamB.civilian,
			teamAImposterWord: wordPairs.teamA.imposter,
			teamBImposterWord: wordPairs.teamB.imposter,
			theme: actualTheme
		};
		currentPlayerIndex = 0;
		wordRevealed = false;
		discussionPhase = false;
		votingPhase = false;
		gameState = 'game';
		isGeneratingWords = false;
	}
	function nextPlayer(): void {
		wordRevealed = false;
		if (currentPlayerIndex < teamPlayOrder.length - 1) {
			currentPlayerIndex++;
		} else {
			discussionPhase = true;
		}
	}
	function revealWord(): void {
		wordRevealed = true;
	}
	function startVoting(): void {
		votingPhase = true;
		discussionPhase = false;
	}
	function resetGame(): void {
		gameState = 'home';
		currentPlayerIndex = 0;
		gameData = null;
		wordRevealed = false;
		teamANames = [''];
		teamBNames = [''];
		savedTeamANames = [];
		savedTeamBNames = [];
		selectedTheme = 'random';
		showThemeInput = false;
		discussionPhase = false;
		votingPhase = false;
		teamPlayOrder = [];
	}
	function showResults(): void {
		gameState = 'reveal';
	}
	function toggleThemeInput(): void {
		showThemeInput = !showThemeInput;
		if (!showThemeInput) {
			selectedTheme = 'random';
		}
	}
	$: currentPlayer = teamPlayOrder[currentPlayerIndex];
	$: teamAImposter = gameData?.players.find((p) => p.team === 'A' && p.role === 'imposter');
	$: teamBImposter = gameData?.players.find((p) => p.team === 'B' && p.role === 'imposter');
	$: totalPlayers = teamASize + teamBSize;
</script>
<Nav />
<div class="min-h-screen bg-gradient-to-br from-slate-900 via-purple-900 to-slate-900 relative overflow-hidden">
	<!-- Animated Background Elements -->
	<div class="absolute inset-0 pointer-events-none">
		<div class="absolute -top-20 -right-20 w-40 h-40 bg-yellow-500/10 rounded-full blur-2xl animate-pulse"></div>
		<div class="absolute -bottom-20 -left-20 w-40 h-40 bg-orange-500/10 rounded-full blur-2xl animate-pulse" style="animation-delay: 2s;"></div>
		<div class="absolute top-1/2 left-1/2 transform -translate-x-1/2 -translate-y-1/2 w-48 h-48 bg-amber-500/5 rounded-full blur-2xl animate-pulse" style="animation-delay: 4s;"></div>
	</div>
	<!-- Main Content -->
	<div class="relative z-10 min-h-screen flex flex-col">
		<div class="p-3 lg:p-5 flex-1 flex flex-col overflow-hidden">
			<!-- Home Screen -->
			{#if gameState === 'home'}
				<div class="flex-1 flex flex-col overflow-hidden">
					<!-- Main Content Area - Takes remaining space -->
					<div class="flex-1 flex flex-col lg:flex-row gap-3 lg:gap-5 overflow-hidden">
						<!-- Left Column - Team Configuration (60% width on desktop) -->
						<div class="lg:w-3/5 bg-white/5 backdrop-blur-xl border border-white/20 rounded-xl lg:rounded-2xl p-3 lg:p-5 shadow-xl flex flex-col overflow-hidden">
							<div class="bg-yellow-500/10 border border-yellow-400/30 rounded-lg lg:rounded-xl p-3 lg:p-4 flex-1 flex flex-col overflow-hidden">
								<div class="flex items-center justify-between mb-3 lg:mb-4">
									<div class="flex items-center gap-2 lg:gap-3">
										<div class="w-8 lg:w-10 h-8 lg:h-10 bg-gradient-to-br from-yellow-500 to-orange-500 rounded-lg lg:rounded-xl flex items-center justify-center shadow-lg">
											<div class="w-4 lg:w-5 h-4 lg:h-5 text-white">{@html UsersIcon}</div>
										</div>
										<div>
											<h3 class="text-base lg:text-xl font-bold text-white">Team Configuration</h3>
											<p class="text-xs text-slate-400">Set up your battle teams</p>
										</div>
									</div>
									<div class="px-3 py-1.5 lg:px-4 lg:py-2 bg-white/20 rounded-lg border border-white/20">
										<span class="text-white text-sm lg:text-base font-bold">{teamASize + teamBSize}</span>
										<span class="text-slate-400 text-xs ml-1">total</span>
									</div>
								</div>
								<!-- Teams Configuration - Flex row for horizontal layout -->
								<div class="flex-1 flex flex-col lg:flex-row gap-3 lg:gap-4">
									<!-- Team A -->
									<div class="flex-1 bg-yellow-500/10 border border-yellow-400/30 rounded-lg p-3 flex flex-col">
										<div class="flex items-center justify-between mb-2">
											<div class="flex items-center gap-2">
												<div class="w-3 h-3 bg-gradient-to-br from-yellow-400 to-yellow-500 rounded-full"></div>
												<label class="text-sm lg:text-base font-bold text-yellow-300">Team Alpha</label>
											</div>
											<div class="px-2 py-1 bg-yellow-500/20 border border-yellow-400/40 rounded-lg">
												<span class="text-yellow-300 text-sm lg:text-lg font-bold">{teamASize}</span>
											</div>
										</div>
										<div class="mb-3">
											<input
												type="range"
												min="2"
												max="6"
												bind:value={teamASize}
												class="w-full h-1.5 lg:h-2 bg-yellow-600/20 rounded-full appearance-none cursor-pointer slider-yellow"
											/>
											<div class="flex justify-between text-xs text-slate-500 mt-1 px-1">
												<span>2</span>
												<span>6</span>
											</div>
										</div>
										<!-- Team A Visualization -->
										<div class="bg-yellow-500/10 border border-yellow-400/30 rounded-lg p-2 flex-1 flex items-center justify-center">
											<div class="flex gap-1 lg:gap-2 flex-wrap justify-center">
												{#each Array(teamASize) as _, i}
													<div class="w-6 lg:w-8 h-6 lg:h-8 bg-gradient-to-br from-yellow-400 to-yellow-600 rounded-lg shadow border border-yellow-300/30 flex items-center justify-center">
														<span class="text-white text-xs font-bold">A{i + 1}</span>
													</div>
												{/each}
											</div>
										</div>
									</div>
									<!-- Team B -->
									<div class="flex-1 bg-orange-500/10 border border-orange-400/30 rounded-lg p-3 flex flex-col">
										<div class="flex items-center justify-between mb-2">
											<div class="flex items-center gap-2">
												<div class="w-3 h-3 bg-gradient-to-br from-orange-400 to-orange-500 rounded-full"></div>
												<label class="text-sm lg:text-base font-bold text-orange-300">Team Bravo</label>
											</div>
											<div class="px-2 py-1 bg-orange-500/20 border border-orange-400/40 rounded-lg">
												<span class="text-orange-300 text-sm lg:text-lg font-bold">{teamBSize}</span>
											</div>
										</div>
										<div class="mb-3">
											<input
												type="range"
												min="2"
												max="6"
												bind:value={teamBSize}
												class="w-full h-1.5 lg:h-2 bg-orange-600/20 rounded-full appearance-none cursor-pointer slider-orange"
											/>
											<div class="flex justify-between text-xs text-slate-500 mt-1 px-1">
												<span>2</span>
												<span>6</span>
											</div>
										</div>
										<!-- Team B Visualization -->
										<div class="bg-orange-500/10 border border-orange-400/30 rounded-lg p-2 flex-1 flex items-center justify-center">
											<div class="flex gap-1 lg:gap-2 flex-wrap justify-center">
												{#each Array(teamBSize) as _, i}
													<div class="w-6 lg:w-8 h-6 lg:h-8 bg-gradient-to-br from-orange-400 to-orange-600 rounded-lg shadow border border-orange-300/30 flex items-center justify-center">
														<span class="text-white text-xs font-bold">B{i + 1}</span>
													</div>
												{/each}
											</div>
										</div>
									</div>
								</div>
								<!-- Battle Preview -->
								<div class="mt-3 pt-3 border-t border-white/20">
									<div class="flex items-center justify-center gap-4 text-center">
										<div class="bg-yellow-500/10 rounded-lg p-2 flex-1">
											<div class="text-lg lg:text-xl font-bold text-yellow-400">{teamASize}</div>
											<div class="text-xs text-yellow-300 font-medium">Team Alpha</div>
										</div>
										<div class="text-lg lg:text-xl font-bold text-white">VS</div>
										<div class="bg-orange-500/10 rounded-lg p-2 flex-1">
											<div class="text-lg lg:text-xl font-bold text-orange-400">{teamBSize}</div>
											<div class="text-xs text-orange-300 font-medium">Team Bravo</div>
										</div>
									</div>
								</div>
							</div>
						</div>
						<!-- Right Column - Settings & How to Play (40% width on desktop) -->
						<div class="lg:w-2/5 flex flex-col gap-3 lg:gap-4">
							<!-- Game Settings -->
							<div class="bg-white/5 backdrop-blur-xl border border-white/20 rounded-xl lg:rounded-2xl p-3 lg:p-4 shadow-xl flex-1 flex flex-col">
								<!-- Language Selection -->
								<div class="bg-blue-500/10 border border-blue-400/30 rounded-lg p-3 mb-3 lg:mb-4">
									<div class="flex items-center gap-2 mb-2">
										<div class="w-6 lg:w-8 h-6 lg:h-8 bg-gradient-to-br from-blue-500 to-indigo-600 rounded-lg flex items-center justify-center">
											<span class="text-white text-sm lg:text-base">🌍</span>
										</div>
										<div>
											<h3 class="text-sm lg:text-base font-bold text-white">Language</h3>
										</div>
									</div>
									<div class="relative language-dropdown">
										<button
											on:click={() => (showLanguageDropdown = !showLanguageDropdown)}
											class="w-full p-2 bg-white/10 border border-white/20 rounded-lg text-white hover:bg-white/20 transition-all flex items-center justify-between text-xs lg:text-sm"
										>
											<div class="flex items-center gap-2">
												<span class="text-base">{languages[selectedLanguage]?.flag || '🌍'}</span>
												<span>{languages[selectedLanguage]?.name || 'English'}</span>
											</div>
											<svg class="w-3 h-3 lg:w-4 lg:h-4 transition-transform {showLanguageDropdown ? 'rotate-180' : ''}" fill="none" stroke="currentColor" viewBox="0 0 24 24">
												<path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"></path>
											</svg>
										</button>
										{#if showLanguageDropdown}
											<div class="absolute top-full left-0 right-0 mt-1 bg-slate-800/95 backdrop-blur-xl border border-white/20 rounded-lg shadow-xl z-50 max-h-32 overflow-y-auto">
												{#each Object.entries(languages) as [code, lang]}
													<button
														on:click={() => {
															selectedLanguage = code;
															showLanguageDropdown = false;
														}}
														class="w-full p-2 text-left hover:bg-white/20 flex items-center gap-2 text-white text-xs lg:text-sm {selectedLanguage === code ? 'bg-blue-500/30' : ''}"
													>
														<span class="text-base">{lang.flag}</span>
														<span>{lang.name}</span>
													</button>
												{/each}
											</div>
										{/if}
									</div>
								</div>
								<!-- Theme Selection -->
								<div class="bg-emerald-500/10 border border-emerald-400/30 rounded-lg p-3 mb-3 lg:mb-4">
									<div class="flex items-center justify-between mb-2">
										<div class="flex items-center gap-2">
											<div class="w-6 lg:w-8 h-6 lg:h-8 bg-gradient-to-br from-emerald-500 to-teal-600 rounded-lg flex items-center justify-center">
												<span class="text-white text-sm lg:text-base">🎨</span>
											</div>
											<div>
												<h3 class="text-sm lg:text-base font-bold text-white">Theme</h3>
											</div>
										</div>
										<button
											on:click={toggleThemeInput}
											class="px-2 py-1 text-xs font-semibold rounded-lg transition-all {showThemeInput ? 'bg-emerald-500/30 text-emerald-300 border border-emerald-400/50' : 'bg-white/20 text-slate-300 hover:bg-white/30'}"
										>
											{showThemeInput ? 'Random' : 'Custom'}
										</button>
									</div>
									{#if showThemeInput}
										<input
											type="text"
											placeholder="Enter theme..."
											bind:value={selectedTheme}
											class="w-full p-2 bg-white/10 border border-white/20 rounded-lg text-white placeholder-slate-400 text-xs focus:border-emerald-400 focus:outline-none"
										/>
										<div class="flex gap-1 mt-2 flex-wrap">
											{#each ['Animals', 'Movies', 'Food'] as suggestion}
												<button
													on:click={() => (selectedTheme = suggestion)}
													class="px-2 py-1 bg-emerald-500/20 text-emerald-300 text-xs rounded-lg hover:bg-emerald-500/30 transition-all"
												>
													{suggestion}
												</button>
											{/each}
										</div>
									{:else}
										<div class="p-2 bg-gradient-to-r from-emerald-500/20 to-teal-500/20 border border-emerald-400/30 rounded-lg text-center">
											<div class="text-emerald-300 font-semibold flex items-center justify-center gap-1 text-xs">
												<span>🎲</span>
												<span>Surprise Theme</span>
											</div>
										</div>
									{/if}
								</div>
								<!-- Game Options -->
								<div class="bg-purple-500/10 border border-purple-400/30 rounded-lg p-3 flex-1 flex flex-col">
									<div class="flex items-center gap-2 mb-3">
										<div class="w-6 lg:w-8 h-6 lg:h-8 bg-gradient-to-br from-purple-500 to-pink-600 rounded-lg flex items-center justify-center">
											<span class="text-white text-sm lg:text-base">⚙️</span>
										</div>
										<div>
											<h3 class="text-sm lg:text-base font-bold text-white">Options</h3>
										</div>
									</div>
									<div class="space-y-3 flex-1 flex flex-col justify-between">
										<div class="flex items-center justify-between p-2 bg-white/10 border border-white/20 rounded-lg">
											<div class="flex-1">
												<div class="font-semibold text-white text-xs">Show Player Roles</div>
											</div>
											<button
												on:click={() => (showRoleInitially = !showRoleInitially)}
												class="relative w-8 h-4 rounded-full transition-all duration-300 {showRoleInitially ? 'bg-gradient-to-r from-purple-500 to-pink-500' : 'bg-slate-600'}"
											>
												<div class="absolute w-3 h-3 bg-white rounded-full top-0.5 transition-all duration-300 {showRoleInitially ? 'translate-x-4' : 'translate-x-0.5'}"></div>
											</button>
										</div>
										<div class="bg-white/10 rounded-lg p-3">
											<div class="text-center">
												<div class="font-bold text-white text-sm mb-1">Game Summary</div>
												<div class="flex justify-center items-center gap-2 text-xs">
													<span class="text-yellow-300 font-semibold">{teamASize}</span>
													<span class="text-slate-400">vs</span>
													<span class="text-orange-300 font-semibold">{teamBSize}</span>
													<span class="text-slate-400">|</span>
													<span class="text-white font-semibold">{totalPlayers} total</span>
												</div>
											</div>
										</div>
									</div>
								</div>
							</div>
							<!-- How to Play Panel -->
							<div class="bg-slate-500/10 border border-slate-400/30 rounded-xl lg:rounded-2xl p-3 lg:p-4 shadow-xl">
								<div class="flex items-center gap-2 mb-3">
									<div class="w-6 lg:w-8 h-6 lg:h-8 bg-gradient-to-br from-slate-500 to-gray-600 rounded-lg flex items-center justify-center">
										<span class="text-white text-sm lg:text-base">📋</span>
									</div>
									<div>
										<h3 class="text-sm lg:text-base font-bold text-white">How to Play</h3>
									</div>
								</div>
								<div class="space-y-3 max-h-40 overflow-y-auto">
									<div class="flex gap-2">
										<div class="w-5 h-5 bg-gradient-to-br from-blue-500 to-blue-600 rounded flex items-center justify-center text-white text-xs font-bold">1</div>
										<div>
											<div class="font-semibold text-white text-xs">Team Words</div>
											<p class="text-xs text-slate-400">Each team gets different word pairs. One player per team is the spy.</p>
										</div>
									</div>
									<div class="flex gap-2">
										<div class="w-5 h-5 bg-gradient-to-br from-green-500 to-green-600 rounded flex items-center justify-center text-white text-xs font-bold">2</div>
										<div>
											<div class="font-semibold text-white text-xs">Cooperate</div>
											<p class="text-xs text-slate-400">Work with your team to identify the spy in the opposing team.</p>
										</div>
									</div>
									<div class="flex gap-2">
										<div class="w-5 h-5 bg-gradient-to-br from-red-500 to-red-600 rounded flex items-center justify-center text-white text-xs font-bold">3</div>
										<div>
											<div class="font-semibold text-white text-xs">Vote & Win</div>
											<p class="text-xs text-slate-400">Vote to eliminate the opposing team's spy. Successful identification wins!</p>
										</div>
									</div>
								</div>
								<div class="mt-3 p-2 bg-gradient-to-r from-yellow-500/20 to-orange-500/20 border border-yellow-400/30 rounded-lg">
									<div class="text-xs font-semibold text-yellow-300">Pro Tip</div>
									<p class="text-xs text-yellow-200/80">Share clues with your team, but don't reveal too much to the opposition!</p>
								</div>
							</div>
						</div>
					</div>
					<!-- Start Game Button - Fixed height -->
					<div class="mt-4 pt-3 border-t border-white/20 flex-shrink-0">
						<button
							on:click={() => (gameState = 'setup')}
							class="w-full bg-gradient-to-r from-yellow-600 via-orange-500 to-yellow-600 hover:from-yellow-500 hover:via-orange-400 hover:to-yellow-500 text-white font-black py-3 px-6 rounded-xl lg:rounded-2xl transition-all duration-300 shadow-lg hover:shadow-xl hover:shadow-yellow-500/25 hover:scale-[1.02] active:scale-[0.98] group relative"
						>
							<div class="absolute inset-0 bg-gradient-to-r from-transparent via-white/20 to-transparent -skew-x-12 translate-x-[-200%] group-hover:translate-x-[200%] transition-transform duration-1000"></div>
							<div class="relative flex items-center justify-center gap-2">
								<div class="w-4 h-4 group-hover:scale-110 transition-transform">{@html PlayIcon}</div>
								<span class="text-base lg:text-lg tracking-wide">Setup Team Battle</span>
							</div>
						</button>
					</div>
				</div>
			{/if}
			<!-- Setup Screen -->
			{#if gameState === 'setup'}
				<div class="flex-1 flex flex-col">
					<!-- Header -->
					<div class="text-center mb-4 lg:mb-6 flex-shrink-0">
						<div class="inline-flex items-center bg-yellow-500/20 border border-yellow-500/30 rounded-full px-3 lg:px-5 py-1.5 lg:py-2 mb-3">
							<div class="w-2 h-2 bg-yellow-400 rounded-full mr-2 animate-pulse"></div>
							<span class="text-yellow-200 text-xs lg:text-sm font-medium">TEAM SETUP</span>
						</div>
						<h2 class="text-xl lg:text-3xl font-black text-white mb-2">Enter Your Warriors</h2>
						<p class="text-xs lg:text-base text-slate-300">Name your team members for the upcoming battle</p>
					</div>
					<!-- Team Setup Cards -->
					<div class="flex-1 bg-white/5 backdrop-blur-xl border border-white/20 rounded-xl lg:rounded-2xl p-3 lg:p-5 shadow-xl flex flex-col">
						<div class="flex-1 grid grid-cols-1 lg:grid-cols-2 gap-3 lg:gap-5">
							<!-- Team Alpha Setup -->
							<div class="bg-yellow-500/20 border border-yellow-400/40 rounded-xl lg:rounded-2xl p-3 lg:p-4 flex flex-col">
								<div class="flex items-center justify-center gap-3 mb-4">
									<div class="w-10 h-10 bg-gradient-to-br from-yellow-500 to-yellow-600 rounded-xl flex items-center justify-center shadow-lg">
										<span class="text-white text-lg font-bold">A</span>
									</div>
									<div class="text-center">
										<h3 class="text-lg lg:text-xl font-bold text-yellow-300">Team Alpha</h3>
										<p class="text-xs text-yellow-200/70">{teamASize} warriors ready</p>
									</div>
								</div>
								<div class="flex-1 space-y-3 overflow-y-auto">
									{#each Array(teamASize) as _, index}
										<div class="group">
											<label class="block text-xs lg:text-sm font-medium text-yellow-400 mb-1 flex items-center gap-2">
												<div class="w-4 h-4 bg-yellow-500/30 rounded flex items-center justify-center">
													<span class="text-yellow-300 text-xs font-bold">{index + 1}</span>
												</div>
												Warrior {index + 1}
											</label>
											<input
												type="text"
												bind:value={teamANames[index]}
												placeholder="Enter name..."
												class="w-full p-2 lg:p-3 bg-white/10 border border-yellow-400/30 rounded-lg text-white placeholder-slate-400 text-xs lg:text-sm focus:border-yellow-400 focus:bg-white/20 focus:outline-none transition-all"
											/>
										</div>
									{/each}
								</div>
							</div>
							<!-- Team Bravo Setup -->
							<div class="bg-orange-500/20 border border-orange-400/40 rounded-xl lg:rounded-2xl p-3 lg:p-4 flex flex-col">
								<div class="flex items-center justify-center gap-3 mb-4">
									<div class="w-10 h-10 bg-gradient-to-br from-orange-500 to-orange-600 rounded-xl flex items-center justify-center shadow-lg">
										<span class="text-white text-lg font-bold">B</span>
									</div>
									<div class="text-center">
										<h3 class="text-lg lg:text-xl font-bold text-orange-300">Team Bravo</h3>
										<p class="text-xs text-orange-200/70">{teamBSize} warriors ready</p>
									</div>
								</div>
								<div class="flex-1 space-y-3 overflow-y-auto">
									{#each Array(teamBSize) as _, index}
										<div class="group">
											<label class="block text-xs lg:text-sm font-medium text-orange-400 mb-1 flex items-center gap-2">
												<div class="w-4 h-4 bg-orange-500/30 rounded flex items-center justify-center">
													<span class="text-orange-300 text-xs font-bold">{index + 1}</span>
												</div>
												Warrior {index + 1}
											</label>
											<input
												type="text"
												bind:value={teamBNames[index]}
												placeholder="Enter name..."
												class="w-full p-2 lg:p-3 bg-white/10 border border-orange-400/30 rounded-lg text-white placeholder-slate-400 text-xs lg:text-sm focus:border-orange-400 focus:bg-white/20 focus:outline-none transition-all"
											/>
										</div>
									{/each}
								</div>
							</div>
						</div>
						<!-- Action Buttons -->
						<div class="flex flex-col sm:flex-row gap-3 pt-4 border-t border-white/20 mt-4">
							<button
								on:click={() => (gameState = 'home')}
								class="flex-1 bg-white/10 hover:bg-white/20 text-white font-semibold py-3 px-4 rounded-xl transition-all duration-300 border border-white/20 hover:border-white/30"
							>
								← Back to Setup
							</button>
							<button
								on:click={startGame}
								disabled={isGeneratingWords}
								class="flex-1 bg-gradient-to-r from-yellow-600 via-orange-500 to-orange-600 hover:from-yellow-500 hover:via-orange-400 hover:to-orange-500 disabled:from-gray-600 disabled:to-gray-600 text-white font-black py-3 px-4 rounded-xl transition-all duration-300 disabled:opacity-50 shadow-lg hover:shadow-xl relative"
							>
								{#if isGeneratingWords}
									<div class="flex items-center justify-center gap-2">
										<div class="animate-spin rounded-full h-4 w-4 border-2 border-white border-t-transparent"></div>
										<span>Generating...</span>
									</div>
								{:else}
									<div class="absolute inset-0 bg-gradient-to-r from-transparent via-white/20 to-transparent -skew-x-12 translate-x-[-200%] group-hover:translate-x-[200%] transition-transform duration-1000"></div>
									<div class="relative flex items-center justify-center gap-2">
										<span class="text-lg">⚔️</span>
										<span>Begin Team Battle</span>
									</div>
								{/if}
							</button>
						</div>
					</div>
				</div>
			{/if}
			<!-- Game Screen -->
			{#if gameState === 'game'}
				<div class="flex-1 flex flex-col lg:flex-row gap-3 lg:gap-4">
					<!-- Left Panel -->
					<div class="lg:w-64 bg-white/10 backdrop-blur-xl border border-white/20 rounded-xl lg:rounded-2xl p-3 lg:p-4 flex flex-col shadow-xl flex-shrink-0">
						<!-- Game Status Header -->
						<div class="text-center pb-2 border-b border-white/20 mb-3">
							<div class="inline-flex items-center bg-yellow-500/20 border border-yellow-500/30 rounded-full px-2 py-1 mb-1">
								<div class="w-1.5 h-1.5 bg-yellow-400 rounded-full mr-1 animate-pulse"></div>
								<span class="text-yellow-200 text-xs">Team Battle</span>
							</div>
							<h3 class="text-sm lg:text-base font-bold text-white">Round {roundNumber}</h3>
						</div>
						{#if !discussionPhase && !votingPhase}
							<!-- Word Reveal Phase -->
							<div class="mb-3">
								<div class="flex justify-between text-xs text-slate-400 mb-1">
									<span>Progress</span>
									<span>{currentPlayerIndex + 1}/{teamPlayOrder.length}</span>
								</div>
								<div class="w-full bg-white/20 rounded-full h-2">
									<div
										class="bg-gradient-to-r from-yellow-500 to-orange-500 h-2 rounded-full transition-all duration-500"
										style="width: {((currentPlayerIndex + 1) / teamPlayOrder.length) * 100}%"
									></div>
								</div>
							</div>
							<!-- Current Player -->
							<div class="bg-{currentPlayer?.team === 'A' ? 'yellow' : 'orange'}-500/10 border border-{currentPlayer?.team === 'A' ? 'yellow' : 'orange'}-400/20 rounded-lg p-3 mb-3">
								<div class="text-center">
									<div class="w-10 h-10 bg-gradient-to-br from-{currentPlayer?.team === 'A' ? 'yellow' : 'orange'}-500 to-{currentPlayer?.team === 'A' ? 'yellow' : 'orange'}-600 rounded-lg flex items-center justify-center text-base font-bold text-white mb-2 mx-auto">
										{currentPlayer?.team}
									</div>
									<div class="text-sm font-bold text-white truncate">
										{currentPlayer?.name || `Team ${currentPlayer?.team} Player`}
									</div>
									<div class="text-xs text-slate-400">
										Team {currentPlayer?.team}
									</div>
								</div>
							</div>
							<!-- Action Button -->
							{#if !wordRevealed}
								<button
									on:click={revealWord}
									class="w-full bg-gradient-to-r from-purple-600 to-pink-600 hover:from-purple-500 hover:to-pink-500 text-white font-bold py-3 px-4 rounded-lg mb-3 transition-all hover:scale-105 active:scale-95"
								>
									<div class="flex items-center justify-center gap-2">
										<div class="w-4 h-4">{@html EyeIcon}</div>
										<span>Reveal Word</span>
									</div>
								</button>
							{:else}
								<button
									on:click={nextPlayer}
									class="w-full bg-gradient-to-r from-emerald-600 to-green-600 hover:from-emerald-500 hover:to-green-500 text-white font-bold py-3 px-4 rounded-lg mb-3 transition-all hover:scale-105 active:scale-95"
								>
									<div class="flex items-center justify-center gap-2">
										<div class="w-4 h-4">{@html ArrowRightIcon}</div>
										<span>{currentPlayerIndex < teamPlayOrder.length - 1 ? 'Next Player' : 'Team Discussion'}</span>
									</div>
								</button>
							{/if}
						{:else if discussionPhase}
							<!-- Discussion Phase -->
							<div class="bg-purple-500/10 border border-purple-400/20 rounded-lg p-4 mb-3">
								<h4 class="text-sm font-bold text-purple-300 mb-2">Team Discussion</h4>
								<p class="text-xs text-purple-200 mb-3">
									Work with your team to identify the opposing team's spy!
								</p>
								<button
									on:click={startVoting}
									class="w-full bg-gradient-to-r from-purple-600 to-pink-600 hover:from-purple-500 hover:to-pink-500 text-white font-bold py-3 px-4 rounded-lg transition-all"
								>
									Start Voting
								</button>
							</div>
						{:else}
							<!-- Voting Phase -->
							<div class="bg-red-500/10 border border-red-400/20 rounded-lg p-4 mb-3">
								<h4 class="text-sm font-bold text-red-300 mb-2">Voting Phase</h4>
								<p class="text-xs text-red-200 mb-3">Teams vote to eliminate suspected spies!</p>
								<button
									on:click={showResults}
									class="w-full bg-gradient-to-r from-red-600 to-pink-600 hover:from-red-500 hover:to-pink-500 text-white font-bold py-3 px-4 rounded-lg transition-all"
								>
									Show Results
								</button>
							</div>
						{/if}
						<!-- Team Rosters -->
						<div class="flex-1 overflow-y-auto mt-2">
							<div class="space-y-3">
								<!-- Team A -->
								<div>
									<h4 class="text-xs lg:text-sm font-bold text-yellow-300 mb-2">Team A</h4>
									<div class="space-y-1 max-h-32 overflow-y-auto">
										{#each gameData?.players.filter((p) => p.team === 'A') || [] as player, index}
											<div class="flex items-center gap-2 p-2 rounded-lg bg-yellow-500/10 border border-yellow-400/20">
												<div class="w-5 h-5 bg-gradient-to-br from-yellow-500 to-yellow-600 rounded text-xs font-bold text-white flex items-center justify-center">
													A{index + 1}
												</div>
												<span class="text-xs lg:text-sm text-white flex-1 truncate">{player.name}</span>
											</div>
										{/each}
									</div>
								</div>
								<!-- Team B -->
								<div>
									<h4 class="text-xs lg:text-sm font-bold text-orange-300 mb-2">Team B</h4>
									<div class="space-y-1 max-h-32 overflow-y-auto">
										{#each gameData?.players.filter((p) => p.team === 'B') || [] as player, index}
											<div class="flex items-center gap-2 p-2 rounded-lg bg-orange-500/10 border border-orange-400/20">
												<div class="w-5 h-5 bg-gradient-to-br from-orange-500 to-orange-600 rounded text-xs font-bold text-white flex items-center justify-center">
													B{index + 1}
												</div>
												<span class="text-xs lg:text-sm text-white flex-1 truncate">{player.name}</span>
											</div>
										{/each}
									</div>
								</div>
							</div>
						</div>
					</div>
					<!-- Main Content Area -->
					<div class="flex-1 bg-white/5 backdrop-blur-xl border border-white/10 rounded-xl lg:rounded-2xl p-4 lg:p-6 flex flex-col justify-center">
						{#if !discussionPhase && !votingPhase}
							<!-- Word Display -->
							<div class="flex-1 flex items-center justify-center">
								{#if !wordRevealed}
									<div class="text-center max-w-md w-full">
										<div class="bg-slate-800/50 border-2 border-dashed border-white/30 rounded-xl p-6 lg:p-8">
											<div class="w-10 h-10 lg:w-12 lg:h-12 mx-auto mb-3 lg:mb-4 text-slate-400 opacity-50">
												{@html EyeOffIcon}
											</div>
											<h3 class="text-base lg:text-xl font-bold text-slate-300 mb-2">Word Hidden</h3>
											<p class="text-sm text-slate-400">
												{currentPlayer?.name || `Team ${currentPlayer?.team} Player`}, click "Reveal Word"
											</p>
										</div>
									</div>
								{:else}
									<div class="text-center max-w-lg w-full px-2">
										{#if showRoleInitially}
											<!-- Role & Word Display -->
											<div class="bg-white/10 border {currentPlayer?.role === 'imposter' ? 'border-red-400/30' : 'border-green-400/30'} rounded-xl p-5 lg:p-6">
												<div class="text-3xl lg:text-4xl mb-3 lg:mb-4">
													{currentPlayer?.role === 'imposter' ? '🕵️' : '👥'}
												</div>
												<h3 class="text-lg lg:text-xl font-bold {currentPlayer?.role === 'imposter' ? 'text-red-400' : 'text-green-400'} mb-2">
													{currentPlayer?.role === 'imposter' ? 'You are the SPY!' : 'You are a CIVILIAN'}
												</h3>
												<p class="text-sm text-slate-300 mb-4">
													Team {currentPlayer?.team} - {currentPlayer?.role === 'imposter' ? 'Your word is different!' : 'Remember this word!'}
												</p>
												<div class="bg-white/20 rounded-xl p-4">
													<div class="text-xl lg:text-2xl font-black text-white">
														{currentPlayer?.word}
													</div>
												</div>
											</div>
										{:else}
											<!-- Word Only Display -->
											<div class="space-y-4">
												<div class="text-lg lg:text-xl font-bold text-{currentPlayer?.team === 'A' ? 'yellow' : 'orange'}-300">
													Team {currentPlayer?.team}
												</div>
												<div class="bg-white/20 rounded-xl p-4">
													<div class="text-xl lg:text-2xl font-black text-white">
														{currentPlayer?.word}
													</div>
												</div>
											</div>
										{/if}
									</div>
								{/if}
							</div>
						{:else if discussionPhase}
							<!-- Discussion Instructions -->
							<div class="text-center max-w-2xl mx-auto px-2">
								<div class="text-3xl lg:text-4xl mb-4">💬</div>
								<h2 class="text-xl lg:text-2xl font-black text-white mb-3">Team Discussion Time!</h2>
								<p class="text-sm lg:text-base text-slate-300 mb-5">
									Work with your teammates to identify the spy from the opposing team. Share clues carefully!
								</p>
								<div class="bg-white/5 border border-white/10 rounded-xl p-4">
									<h3 class="text-sm lg:text-base font-bold text-white mb-3">Team Words Reference</h3>
									<div class="grid grid-cols-1 lg:grid-cols-2 gap-3">
										<div class="bg-yellow-500/10 border border-yellow-400/20 rounded-lg p-3">
											<h4 class="font-bold text-yellow-300 mb-2 text-sm">Team A Word</h4>
											<div class="text-lg text-white">{gameData?.teamAWord}</div>
										</div>
										<div class="bg-orange-500/10 border border-orange-400/20 rounded-lg p-3">
											<h4 class="font-bold text-orange-300 mb-2 text-sm">Team B Word</h4>
											<div class="text-lg text-white">{gameData?.teamBWord}</div>
										</div>
									</div>
								</div>
							</div>
						{:else}
							<!-- Voting Instructions -->
							<div class="text-center max-w-2xl mx-auto px-2">
								<div class="text-3xl lg:text-4xl mb-4">🗳️</div>
								<h2 class="text-xl lg:text-2xl font-black text-white mb-3">Voting Time!</h2>
								<p class="text-sm lg:text-base text-slate-300 mb-5">
									Each team votes to eliminate one player from the opposing team. Choose wisely!
								</p>
								<div class="bg-white/5 border border-white/10 rounded-xl p-4">
									<p class="text-sm text-slate-400">Conduct your voting outside the app, then reveal the results!</p>
								</div>
							</div>
						{/if}
					</div>
				</div>
			{/if}
			<!-- Results Screen -->
			{#if gameState === 'reveal'}
				<div class="flex-1 flex flex-col">
					<div class="text-center mb-5 flex-shrink-0">
						<h2 class="text-xl lg:text-2xl font-black text-white mb-4">Game Results</h2>
					</div>
					<div class="flex-1 space-y-4 mb-5 overflow-y-auto">
						<!-- Team A Results -->
						<div class="bg-yellow-500/10 border border-yellow-400/20 rounded-xl p-4">
							<h3 class="text-lg lg:text-xl font-bold text-yellow-300 mb-3 text-center">Team A</h3>
							<div class="grid grid-cols-1 lg:grid-cols-2 gap-3">
								<div class="text-center">
									<h4 class="font-bold text-yellow-200 mb-2 text-sm">Civilian Word</h4>
									<div class="bg-white/10 rounded-lg p-3 text-lg font-bold text-white">
										{gameData?.teamAWord}
									</div>
								</div>
								<div class="text-center">
									<h4 class="font-bold text-red-300 mb-2 text-sm">The Spy</h4>
									<div class="bg-red-500/20 rounded-lg p-3">
										<div class="font-bold text-red-200 text-base">{teamAImposter?.name}</div>
										<div class="text-sm text-red-300">Word: {gameData?.teamAImposterWord}</div>
									</div>
								</div>
							</div>
						</div>
						<!-- Team B Results -->
						<div class="bg-orange-500/10 border border-orange-400/20 rounded-xl p-4">
							<h3 class="text-lg lg:text-xl font-bold text-orange-300 mb-3 text-center">Team B</h3>
							<div class="grid grid-cols-1 lg:grid-cols-2 gap-3">
								<div class="text-center">
									<h4 class="font-bold text-orange-200 mb-2 text-sm">Civilian Word</h4>
									<div class="bg-white/10 rounded-lg p-3 text-lg font-bold text-white">
										{gameData?.teamBWord}
									</div>
								</div>
								<div class="text-center">
									<h4 class="font-bold text-red-300 mb-2 text-sm">The Spy</h4>
									<div class="bg-red-500/20 rounded-lg p-3">
										<div class="font-bold text-red-200 text-base">{teamBImposter?.name}</div>
										<div class="text-sm text-red-300">Word: {gameData?.teamBImposterWord}</div>
									</div>
								</div>
							</div>
						</div>
					</div>
					<!-- Action Buttons -->
					<div class="bg-white/5 backdrop-blur-xl border border-white/10 rounded-xl p-4 flex-shrink-0">
						<div class="flex flex-col sm:flex-row gap-3">
							<button
								on:click={() => {
									teamANames = [...savedTeamANames];
									teamBNames = [...savedTeamBNames];
									startGame();
								}}
								disabled={isGeneratingWords}
								class="flex-1 bg-gradient-to-r from-yellow-600 to-orange-600 hover:from-yellow-500 hover:to-orange-500 disabled:from-gray-600 disabled:to-gray-600 text-white font-bold py-3 px-4 rounded-xl transition-all disabled:opacity-50"
							>
								{#if isGeneratingWords}
									<div class="flex items-center justify-center gap-2">
										<div class="animate-spin rounded-full h-4 w-4 border-2 border-white border-t-transparent"></div>
										<span>Starting...</span>
									</div>
								{:else}
									<div class="flex items-center justify-center gap-2">
										<span class="text-base">Play Again</span>
									</div>
								{/if}
							</button>
							<button
								on:click={resetGame}
								class="flex-1 bg-white/10 hover:bg-white/20 text-white font-medium py-3 px-4 rounded-xl transition-all"
							>
								New Game
							</button>
						</div>
					</div>
				</div>
			{/if}
		</div>
	</div>
</div>
<style>
	/* Custom slider styling for Team A */
	.slider-yellow::-webkit-slider-thumb {
		-webkit-appearance: none;
		appearance: none;
		height: 16px;
		width: 16px;
		border-radius: 50%;
		background: linear-gradient(45deg, #f59e0b, #d97706);
		cursor: pointer;
		border: 2px solid white;
		box-shadow: 0 0 8px rgba(245, 158, 11, 0.6);
		transition: all 0.2s ease;
	}
	.slider-yellow::-webkit-slider-thumb:hover {
		transform: scale(1.1);
		box-shadow: 0 0 12px rgba(245, 158, 11, 0.8);
	}
	/* Custom slider styling for Team B */
	.slider-orange::-webkit-slider-thumb {
		-webkit-appearance: none;
		appearance: none;
		height: 16px;
		width: 16px;
		border-radius: 50%;
		background: linear-gradient(45deg, #ea580c, #c2410c);
		cursor: pointer;
		border: 2px solid white;
		box-shadow: 0 0 8px rgba(234, 88, 12, 0.6);
		transition: all 0.2s ease;
	}
	.slider-orange::-webkit-slider-thumb:hover {
		transform: scale(1.1);
		box-shadow: 0 0 12px rgba(234, 88, 12, 0.8);
	}
	/* Firefox slider styling */
	.slider-yellow::-moz-range-thumb,
	.slider-orange::-moz-range-thumb {
		height: 16px;
		width: 16px;
		border-radius: 50%;
		cursor: pointer;
		border: 2px solid white;
		background: linear-gradient(45deg, #f59e0b, #ea580c);
	}
	/* Custom Slider Styling */
	input[type='range'] {
		-webkit-appearance: none;
		appearance: none;
		background: transparent;
		cursor: pointer;
	}
	input[type='range']::-webkit-slider-track {
		background: rgba(255, 255, 255, 0.2);
		height: 4px;
		border-radius: 2px;
	}
	/* Focus styles for accessibility */
	button:focus-visible,
	input:focus-visible {
		outline: 2px solid #f59e0b;
		outline-offset: 2px;
	}
	/* Custom scrollbar */
	.overflow-y-auto::-webkit-scrollbar {
		width: 6px;
	}
	.overflow-y-auto::-webkit-scrollbar-track {
		background: rgba(255, 255, 255, 0.05);
		border-radius: 8px;
	}
	.overflow-y-auto::-webkit-scrollbar-thumb {
		background: linear-gradient(to bottom, #f59e0b, #ea580c);
		border-radius: 8px;
	}
</style>