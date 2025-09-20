<script lang="ts">
	import { onMount } from 'svelte';
	import { GoogleGenerativeAI } from '@google/generative-ai';
	import Nav from '../components/Nav.svelte';

	interface Player {
		name: string;
		role: 'imposter' | 'civilian' | 'dumb';
		word: string;
	}
	interface GameData {
		players: Player[];
		civilianWord: string;
		imposterWord: string;
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
	let numPlayers: number = 4;
	let playerNames: string[] = [''];
	let savedPlayerNames: string[] = [];
	let currentPlayerIndex: number = 0;
	let gameData: GameData | null = null;
	let showRoleInitially: boolean = false;
	let wordRevealed: boolean = false;
	let isGeneratingWords: boolean = false;
	let selectedLanguage: string = 'english';
	let showLanguageDropdown: boolean = false;
	let selectedTheme: string = 'random';
	let showThemeInput: boolean = false;
	let includeDumbRole: boolean = false;

	// New variables for randomized player order
	let randomizedPlayerOrder: number[] = [];

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
	const UserIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <path d="M20 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2"/>
    <circle cx="12" cy="7" r="4"/>
  </svg>`;
	const SettingsIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <circle cx="12" cy="12" r="3"/>
    <path d="m12 1 2.86 5.79A4 4 0 0 0 18.65 9l5.79 2.86a1 1 0 0 1 0 1.79L18.65 15a4 4 0 0 0-3.79 3.21L12 24.14a1 1 0 0 1-1.79 0L8.35 18.21A4 4 0 0 0 4.56 15l-5.79-2.86a1 1 0 0 1 0-1.79L4.56 9a4 4 0 0 0 3.79-3.21L10.21.86a1 1 0 0 1 1.79 0Z"/>
  </svg>`;
	const ClockIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <circle cx="12" cy="12" r="10"/>
    <polyline points="12,6 12,12 16,14"/>
  </svg>`;
	const GamepadIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
    <line x1="6" y1="12" x2="18" y2="12"/>
    <line x1="12" y1="6" x2="12" y2="18"/>
    <rect width="20" height="16" x="2" y="4" rx="2"/>
  </svg>`;

	const languages = {
		english: { name: 'English', flag: '🇺🇸' },
		nepali: { name: 'नेपाली', flag: '🇳🇵' },
		hindi: { name: 'हिंदी', flag: '🇮🇳' },
		spanish: { name: 'Español', flag: '🇪🇸' },
		french: { name: 'Français', flag: '🇫🇷' },
		german: { name: 'Deutsch', flag: '🇩🇪' },
		portuguese: { name: 'Português', flag: '🇵🇹' },
		italian: { name: 'Italiano', flag: '🇮🇹' },
		chinese: { name: '中文', flag: '🇨🇳' },
		japanese: { name: '日本語', flag: '🇯🇵' },
		korean: { name: '한국어', flag: '🇰🇷' },
		arabic: { name: 'العربية', flag: '🇸🇦' },
		russian: { name: 'Русский', flag: '🇷🇺' },
		dutch: { name: 'Nederlands', flag: '🇳🇱' },
		swedish: { name: 'Svenska', flag: '🇸🇪' },
		turkish: { name: 'Türkçe', flag: '🇹🇷' }
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
			'Vegetables',
			'School Subjects',
			'Weather',
			'Clothing',
			'Furniture',
			'Technology',
			'Musical Instruments',
			'Olympic Sports',
			'Kitchen Appliances',
			'Beverages',
			'Desserts',
			'Flowers',
			'Birds',
			'Ocean Animals',
			'Superheroes',
			'TV Shows',
			'Video Games',
			'Social Media Apps',
			'Pizza Toppings',
			'Books',
			'Planets',
			'Gemstones',
			'Dance Styles',
			'Ice Cream Flavors',
			'Dog Breeds',
			'Cat Breeds',
			'Cooking Methods',
			'Art Supplies',
			'Phone Brands',
			'Card Games',
			'Cartoon Characters',
			'Fast Food Chains',
			'Halloween Costumes',
			'Christmas Decorations',
			'Office Supplies',
			'Camping Gear',
			'Exercise Equipment'
		],
		nepali: [
			'जनावरहरू',
			'खाना',
			'चलचित्र',
			'देशहरू',
			'खेलहरू',
			'रङहरू',
			'पेशाहरू',
			'सङ्गीत',
			'गाडीका ब्रान्डहरू',
			'फलफूलहरू',
			'तरकारीहरू',
			'मौसम',
			'लुगाहरू',
			'फर्निचर',
			'प्रविधि',
			'वाद्ययन्त्रहरू',
			'खेलकुद',
			'भाँडाकुँडा',
			'पेयपदार्थ',
			'मिठाईहरू',
			'फूलहरू',
			'चराहरू',
			'समुद्री जनावरहरू',
			'टिभी कार्यक्रमहरू',
			'किताबहरू',
			'ग्रहहरू',
			'रत्नहरू',
			'नृत्य शैलीहरू',
			'आइसक्रिम स्वादहरू',
			'कुकुरका जातहरू',
			'बिरालाका जातहरू',
			'खाना पकाउने तरिकाहरू',
			'कला सामग्रीहरू',
			'मोबाइल ब्रान्डहरू',
			'ताश खेलहरू',
			'कार्टुन पात्रहरू',
			'फास्ट फूड चेनहरू'
		],
		hindi: [
			'जानवर',
			'खाना',
			'फिल्में',
			'देश',
			'खेल',
			'रंग',
			'पेशे',
			'संगीत',
			'कार ब्रांड',
			'फल',
			'सब्जियां',
			'मौसम',
			'कपड़े',
			'फर्नीचर',
			'तकनीक',
			'वाद्य यंत्र',
			'खेल',
			'बर्तन',
			'पेय पदार्थ',
			'मिठाइयां',
			'फूल',
			'पक्षी',
			'समुद्री जानवर',
			'टीवी शो',
			'किताबें',
			'ग्रह',
			'रत्न',
			'नृत्य शैली',
			'आइसक्रीम स्वाद'
		],
		spanish: [
			'Animales',
			'Comida',
			'Películas',
			'Países',
			'Deportes',
			'Colores',
			'Profesiones',
			'Géneros Musicales',
			'Marcas de Autos',
			'Frutas',
			'Verduras',
			'Clima',
			'Ropa',
			'Muebles',
			'Tecnología',
			'Instrumentos Musicales',
			'Deportes Olímpicos',
			'Electrodomésticos',
			'Bebidas',
			'Postres',
			'Flores',
			'Aves',
			'Animales Marinos',
			'Programas de TV'
		],
		french: [
			'Animaux',
			'Nourriture',
			'Films',
			'Pays',
			'Sports',
			'Couleurs',
			'Professions',
			'Genres Musicaux',
			'Marques de Voiture',
			'Fruits',
			'Légumes',
			'Météo',
			'Vêtements',
			'Meubles',
			'Technologie',
			'Instruments de Musique',
			'Sports Olympiques',
			'Électroménager'
		],
		german: [
			'Tiere',
			'Essen',
			'Filme',
			'Länder',
			'Sport',
			'Farben',
			'Berufe',
			'Musikrichtungen',
			'Automarken',
			'Früchte',
			'Gemüse',
			'Wetter',
			'Kleidung',
			'Möbel',
			'Technologie',
			'Musikinstrumente',
			'Olympische Sportarten',
			'Haushaltsgeräte'
		],
		portuguese: [
			'Animais',
			'Comida',
			'Filmes',
			'Países',
			'Esportes',
			'Cores',
			'Profissões',
			'Gêneros Musicais',
			'Marcas de Carros',
			'Frutas',
			'Legumes',
			'Clima',
			'Roupas',
			'Móveis',
			'Tecnologia',
			'Instrumentos Musicais',
			'Esportes Olímpicos',
			'Eletrodomésticos'
		],
		italian: [
			'Animali',
			'Cibo',
			'Film',
			'Paesi',
			'Sport',
			'Colori',
			'Professioni',
			'Generi Musicali',
			'Marche di Auto',
			'Frutta',
			'Verdure',
			'Tempo',
			'Vestiti',
			'Mobili',
			'Tecnologia',
			'Strumenti Musicali',
			'Sport Olimpici',
			'Elettrodomestici'
		],
		chinese: [
			'动物',
			'食物',
			'电影',
			'国家',
			'运动',
			'颜色',
			'职业',
			'音乐类型',
			'汽车品牌',
			'水果',
			'蔬菜',
			'天气',
			'服装',
			'家具',
			'技术',
			'乐器',
			'奥运项目',
			'厨房用具'
		],
		japanese: [
			'動物',
			'食べ物',
			'映画',
			'国',
			'スポーツ',
			'色',
			'職業',
			'音楽ジャンル',
			'車のブランド',
			'果物',
			'野菜',
			'天気',
			'服',
			'家具',
			'技術',
			'楽器',
			'オリンピック競技',
			'キッチン用品'
		],
		korean: [
			'동물',
			'음식',
			'영화',
			'국가',
			'스포츠',
			'색깔',
			'직업',
			'음악 장르',
			'자동차 브랜드',
			'과일',
			'야채',
			'날씨',
			'옷',
			'가구',
			'기술',
			'악기',
			'올림픽 종목',
			'주방용품'
		],
		arabic: [
			'الحيوانات',
			'الطعام',
			'الأفلام',
			'البلدان',
			'الرياضة',
			'الألوان',
			'المهن',
			'الأنواع الموسيقية',
			'ماركات السيارات',
			'الفواكه',
			'الخضروات',
			'الطقس',
			'الملابس',
			'الأثاث',
			'التكنولوجيا',
			'الآلات الموسيقية',
			'الألعاب الأولمبية',
			'أدوات المطبخ'
		],
		russian: [
			'Животные',
			'Еда',
			'Фильмы',
			'Страны',
			'Спорт',
			'Цвета',
			'Профессии',
			'Музыкальные жанры',
			'Марки автомобилей',
			'Фрукты',
			'Овощи',
			'Погода',
			'Одежда',
			'Мебель',
			'Технологии',
			'Музыкальные инструменты',
			'Олимпийские виды спорта',
			'Кухонная техника'
		],
		dutch: [
			'Dieren',
			'Eten',
			'Films',
			'Landen',
			'Sport',
			'Kleuren',
			'Beroepen',
			'Muziekgenres',
			'Automerken',
			'Fruit',
			'Groenten',
			'Weer',
			'Kleding',
			'Meubels',
			'Technologie',
			'Muziekinstrumenten',
			'Olympische sporten',
			'Keukenapparatuur'
		],
		swedish: [
			'Djur',
			'Mat',
			'Filmer',
			'Länder',
			'Sport',
			'Färger',
			'Yrken',
			'Musikgenrer',
			'Bilmärken',
			'Frukt',
			'Grönsaker',
			'Väder',
			'Kläder',
			'Möbler',
			'Teknik',
			'Musikinstrument',
			'Olympiska sporter',
			'Köksapparater'
		],
		turkish: [
			'Hayvanlar',
			'Yiyecek',
			'Filmler',
			'Ülkeler',
			'Spor',
			'Renkler',
			'Meslekler',
			'Müzik Türleri',
			'Araba Markaları',
			'Meyveler',
			'Sebzeler',
			'Hava Durumu',
			'Giyim',
			'Mobilya',
			'Teknoloji',
			'Müzik Aletleri',
			'Olimpik Sporlar',
			'Mutfak Aletleri'
		]
	};

	$: {
		if (numPlayers > 0) {
			playerNames = Array(numPlayers)
				.fill('')
				.map((_, i: number) => playerNames[i] || '');
		}
	}

	onMount(() => {
		// Close language dropdown when clicking outside
		function handleClickOutside(event: MouseEvent) {
			if (showLanguageDropdown && !event.target?.closest('.language-dropdown')) {
				showLanguageDropdown = false;
			}
		}

		document.addEventListener('click', handleClickOutside);

		return () => {
			document.removeEventListener('click', handleClickOutside);
		};
	});

	// Function to shuffle array using Fisher-Yates algorithm
	function shuffleArray(array: number[]): number[] {
		const shuffled = [...array];
		for (let i = shuffled.length - 1; i > 0; i--) {
			const j = Math.floor(Math.random() * (i + 1));
			[shuffled[i], shuffled[j]] = [shuffled[j], shuffled[i]];
		}
		return shuffled;
	}

	// Function to generate randomized player order
	function generateRandomPlayerOrder(): void {
		const playerIndices = Array.from({ length: numPlayers }, (_, i) => i);
		randomizedPlayerOrder = shuffleArray(playerIndices);
	}

	function getRandomUnusedTheme(): string {
		const themes = availableThemes[selectedLanguage] || availableThemes.english;
		const unusedThemes = themes.filter((theme) => !usedThemes.has(theme.toLowerCase()));

		if (unusedThemes.length === 0) {
			usedThemes.clear();
			usedThemes = new Set();
			return themes[Math.floor(Math.random() * themes.length)];
		}

		return unusedThemes[Math.floor(Math.random() * unusedThemes.length)];
	}

	function markThemeAsUsed(theme: string): void {
		usedThemes.add(theme.toLowerCase());
		usedThemes = usedThemes;
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

		if (currentSessionPairs.length > 15) {
			const oldestPair = currentSessionPairs.shift();
			if (oldestPair) {
				const oldKey = createWordPairKey(oldestPair.civilian, oldestPair.imposter);
				usedWordPairs.delete(oldKey);
			}
		}

		usedWordPairs = usedWordPairs;
		currentSessionPairs = currentSessionPairs;
	}

	async function generateWithGemini(prompt: string): Promise<WordPair> {
		try {
			const genAI = new GoogleGenerativeAI(
				import.meta.env.VITE_REACT_APP_GEMINI_API_KEY || 'your-api-key-here'
			);

			// List of models in priority order
			const models = [
				'gemini-2.0-flash-exp',
				'gemini-2.0-flash-lite',
				'gemini-2.0-flash',
				'gemini-2.5-flash',
				'gemini-2.5-flash-lite'
			];

			let text = null;

			for (const modelName of models) {
				try {
					console.log(`🔄 Trying model: ${modelName}`);
					const model = genAI.getGenerativeModel({ model: modelName });
					const result = await model.generateContent(prompt);
					const response = await result.response;
					text = await response.text();
					if (text) {
						console.log(`✅ Success with model: ${modelName}`);
						break;
					}
				} catch (err) {
					console.warn(`❌ Failed with ${modelName}:`, err.message);
					// Try next model
				}
			}

			if (!text) throw new Error('All models failed');

			return parseJSON(text);
		} catch (error) {
			console.error('Error generating with Gemini:', error);
			// Fallback words for different languages
			const fallbacks = {
				english: { civilian: 'Dog', imposter: 'Cat' },
				nepali: { civilian: 'कुकुर', imposter: 'बिरालो' },
				hindi: { civilian: 'कुत्ता', imposter: 'बिल्ली' },
				spanish: { civilian: 'Perro', imposter: 'Gato' },
				french: { civilian: 'Chien', imposter: 'Chat' },
				german: { civilian: 'Hund', imposter: 'Katze' },
				portuguese: { civilian: 'Cão', imposter: 'Gato' },
				italian: { civilian: 'Cane', imposter: 'Gatto' },
				chinese: { civilian: '狗', imposter: '猫' },
				japanese: { civilian: '犬', imposter: '猫' },
				korean: { civilian: '개', imposter: '고양이' },
				arabic: { civilian: 'كلب', imposter: 'قطة' },
				russian: { civilian: 'Собака', imposter: 'Кошка' },
				dutch: { civilian: 'Hond', imposter: 'Kat' },
				swedish: { civilian: 'Hund', imposter: 'Katt' },
				turkish: { civilian: 'Köpek', imposter: 'Kedi' }
			};
			return fallbacks[selectedLanguage] || fallbacks.english;
		}
	}

	function parseJSON(text: string): WordPair {
		try {
			const jsonMatch = text.match(/\{[\s\S]*\}/);
			if (jsonMatch) {
				return JSON.parse(jsonMatch[0]);
			}

			const lines = text.split('\n').filter((line: string) => line.trim());
			const civilianMatch = lines.find(
				(line: string) => line.toLowerCase().includes('civilian') || line.includes('सिभिलियन')
			);
			const imposterMatch = lines.find(
				(line: string) => line.toLowerCase().includes('imposter') || line.includes('इम्पोस्टर')
			);

			if (civilianMatch && imposterMatch) {
				return {
					civilian: civilianMatch.split(':').pop()?.trim().replace(/['"]/g, '') || 'Dog',
					imposter: imposterMatch.split(':').pop()?.trim().replace(/['"]/g, '') || 'Cat'
				};
			}

			const fallbacks = {
				english: { civilian: 'Dog', imposter: 'Cat' },
				nepali: { civilian: 'कुकुर', imposter: 'बिरालो' },
				hindi: { civilian: 'कुत्ता', imposter: 'बिल्ली' },
				spanish: { civilian: 'Perro', imposter: 'Gato' },
				french: { civilian: 'Chien', imposter: 'Chat' },
				german: { civilian: 'Hund', imposter: 'Katze' },
				portuguese: { civilian: 'Cão', imposter: 'Gato' },
				italian: { civilian: 'Cane', imposter: 'Gatto' },
				chinese: { civilian: '狗', imposter: '猫' },
				japanese: { civilian: '犬', imposter: '猫' },
				korean: { civilian: '개', imposter: '고양이' },
				arabic: { civilian: 'كلب', imposter: 'قطة' },
				russian: { civilian: 'Собака', imposter: 'Кошка' },
				dutch: { civilian: 'Hond', imposter: 'Kat' },
				swedish: { civilian: 'Hund', imposter: 'Katt' },
				turkish: { civilian: 'Köpek', imposter: 'Kedi' }
			};
			return fallbacks[selectedLanguage] || fallbacks.english;
		} catch (error) {
			console.error('Error parsing response:', error);
			const fallbacks = {
				english: { civilian: 'Dog', imposter: 'Cat' },
				nepali: { civilian: 'कुकुर', imposter: 'बिरालो' },
				hindi: { civilian: 'कुत्ता', imposter: 'बिल्ली' },
				spanish: { civilian: 'Perro', imposter: 'Gato' },
				french: { civilian: 'Chien', imposter: 'Chat' },
				german: { civilian: 'Hund', imposter: 'Katze' },
				portuguese: { civilian: 'Cão', imposter: 'Gato' },
				italian: { civilian: 'Cane', imposter: 'Gatto' },
				chinese: { civilian: '狗', imposter: '猫' },
				japanese: { civilian: '犬', imposter: '猫' },
				korean: { civilian: '개', imposter: '고양이' },
				arabic: { civilian: 'كلب', imposter: 'قطة' },
				russian: { civilian: 'Собака', imposter: 'Кошка' },
				dutch: { civilian: 'Hond', imposter: 'Kat' },
				swedish: { civilian: 'Hund', imposter: 'Katt' },
				turkish: { civilian: 'Köpek', imposter: 'Kedi' }
			};
			return fallbacks[selectedLanguage] || fallbacks.english;
		}
	}

	async function startGame(): Promise<void> {
		if (numPlayers < 3) {
			alert('At least 3 players are required!');
			return;
		}

		savedPlayerNames = [...playerNames];
		isGeneratingWords = true;

		// Generate randomized player order
		generateRandomPlayerOrder();

		let actualTheme: string;
		if (selectedTheme.toLowerCase() === 'random' || !selectedTheme.trim()) {
			actualTheme = getRandomUnusedTheme();
		} else {
			actualTheme = selectedTheme;
		}

		markThemeAsUsed(actualTheme);

		const usedPairsContext =
			currentSessionPairs.length > 0
				? `\n- AVOID these recently used pairs: ${currentSessionPairs
						.map((pair) => `${pair.civilian}/${pair.imposter}`)
						.join(', ')}`
				: '';

		// Replace the problematic section around line 459-461 with this:

		const prompt =
			selectedLanguage === 'nepali' || selectedLanguage === 'hindi'
				? `Generate two ${languages[selectedLanguage]?.name || 'Nepali'} words for a "Who is the Imposter" game.

THEME: "${actualTheme}"
DIFFICULTY: Make it challenging but fair

RULES:
- Both words MUST be from the "${actualTheme}" category
- Words should be in the SAME FIELD but NOT too obviously related
- Avoid rhyming words, similar spellings, or words that start with same letters
- Choose words that are distinctly different but within the same theme
- Make civilians think carefully, not guess immediately  
- Both should be commonly known words in ${languages[selectedLanguage]?.name || 'English'}${usedPairsContext}

EXAMPLES OF GOOD PAIRS:
- Theme "Animals": Lion/Elephant (both big animals, but very different)
- Theme "Food": Pizza/Sushi (both popular foods, but different cuisines) 
- Theme "Sports": Cricket/Basketball (both team sports, but very different)
- Theme "Movies": Comedy/Horror (both movie genres, but opposite feelings)
- Theme "Countries": Japan/Brazil (both countries, but different continents)

Respond in JSON format:
{
  "civilian": "word_from_theme_in_${languages[selectedLanguage]?.name || 'selected_language'}",
  "imposter": "different_but_related_word_from_theme_in_${languages[selectedLanguage]?.name || 'selected_language'}"
}`
				: `Generate two ${languages[selectedLanguage]?.name || 'English'} words for a "Who is the Imposter" game.

THEME: "${actualTheme}"
DIFFICULTY: Make it challenging but fair

RULES:
- Both words MUST be from the "${actualTheme}" category
- Words should be in the SAME FIELD but NOT too obviously related
- Avoid rhyming words, similar spellings, or words that start with same letters
- Choose words that are distinctly different but within the same theme
- Make civilians think carefully, not guess immediately  
- Both should be commonly known words in ${languages[selectedLanguage]?.name || 'English'}${usedPairsContext}

EXAMPLES OF GOOD PAIRS:
- Theme "Animals": Lion/Elephant (both big animals, but very different)
- Theme "Food": Pizza/Sushi (both popular foods, but different cuisines) 
- Theme "Sports": Cricket/Basketball (both team sports, but very different)
- Theme "Movies": Comedy/Horror (both movie genres, but opposite feelings)
- Theme "Countries": Japan/Brazil (both countries, but different continents)

Respond in JSON format:
{
  "civilian": "word_from_theme_in_${languages[selectedLanguage]?.name || 'selected_language'}",
  "imposter": "different_but_related_word_from_theme_in_${languages[selectedLanguage]?.name || 'selected_language'}"
}`;

		let wordPair: WordPair;
		let attempts = 0;
		const maxAttempts = 3;

		do {
			wordPair = await generateWithGemini(prompt);
			attempts++;
		} while (isRecentlyUsed(wordPair) && attempts < maxAttempts);

		addToUsedPairs(wordPair);

		const imposterIndex = Math.floor(Math.random() * numPlayers);

		let dumbIndex = -1;
		if (includeDumbRole) {
			const availableIndices = Array.from({ length: numPlayers }, (_, i) => i).filter(
				(i) => i !== imposterIndex && i !== 0
			);
			if (availableIndices.length > 0) {
				dumbIndex = availableIndices[Math.floor(Math.random() * availableIndices.length)];
			}
		}

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
		savedPlayerNames = [];
		selectedTheme = 'random';
		showThemeInput = false;
		randomizedPlayerOrder = [];
	}

	function restartWithSamePlayers(): void {
		playerNames = [...savedPlayerNames];
		numPlayers = savedPlayerNames.length;
		startGame();
	}

	function showImposter(): void {
		gameState = 'reveal';
	}

	function toggleThemeInput(): void {
		showThemeInput = !showThemeInput;
		if (!showThemeInput) {
			selectedTheme = 'random';
		}
	}

	// Updated reactive statements to use randomized order
	$: currentPlayer = gameData?.players[randomizedPlayerOrder[currentPlayerIndex]];
	$: imposter = gameData?.players.find((p: Player) => p.role === 'imposter');
	$: dumbPlayer = gameData?.players.find((p: Player) => p.role === 'dumb');
</script>

<Nav />

<div
	class="h-screen bg-gradient-to-br from-slate-900 via-purple-900 to-slate-900 relative overflow-hidden"
>
	<!-- Animated Background Elements -->
	<div class="absolute inset-0 overflow-hidden pointer-events-none">
		<div
			class="absolute -top-20 -right-20 w-40 h-40 bg-purple-500/10 rounded-full blur-2xl animate-pulse"
		></div>
		<div
			class="absolute -bottom-20 -left-20 w-40 h-40 bg-blue-500/10 rounded-full blur-2xl animate-pulse"
			style="animation-delay: 2s;"
		></div>
		<div
			class="absolute top-1/2 left-1/2 transform -translate-x-1/2 -translate-y-1/2 w-48 h-48 bg-indigo-500/5 rounded-full blur-2xl animate-pulse"
			style="animation-delay: 4s;"
		></div>
	</div>

	<!-- Main Content -->
	<div class="relative z-10 h-full flex flex-col">
		<div class="flex-1 p-3 md:p-4 lg:p-32 overflow-hidden">
			<!-- Home Screen - Responsive Layout -->
			{#if gameState === 'home'}
				<div class="max flex flex-col py-2 md:py-4 max-w-7xl mx-auto px-4">
					<!-- Game Settings Panel - Full Height Grid -->
					<div
						class="flex-1 bg-white/5 backdrop-blur-xl border border-white/10 rounded-xl p-3 md:p-4 grid grid-rows-[auto_1fr_auto] gap-3 min-h-0"
					>
						<!-- Stats Header - Compact -->
						<div class="bg-white/5 border border-white/10 rounded-lg p-2">
							<div class="grid grid-cols-3 gap-2 text-center">
								<div>
									<div class="text-base md:text-lg font-bold text-white">
										{currentSessionPairs.length}
									</div>
									<div class="text-xs text-slate-400">Rounds</div>
								</div>
								<div>
									<div class="text-base md:text-lg font-bold text-white">{usedThemes.size}</div>
									<div class="text-xs text-slate-400">Themes</div>
								</div>
								<div>
									<div class="text-base md:text-lg font-bold text-white">∞</div>
									<div class="text-xs text-slate-400">Fun</div>
								</div>
							</div>
						</div>

						<!-- Main Settings Grid - Flexible -->
						<div class="grid grid-cols-1 lg:grid-cols-2 gap-3 min-h-0">
							<!-- Left Column -->
							<div class="space-y-3">
								<!-- Language Card - Compact -->
								<div
									class="bg-gradient-to-br from-blue-500/5 to-indigo-500/5 border border-blue-400/20 rounded-lg p-3 hover:border-blue-400/30 transition-all"
								>
									<div class="flex items-center justify-between mb-2">
										<h3 class="text-sm font-bold text-white flex items-center gap-1">
											<span class="text-blue-400">🌍</span>
											Language
										</h3>
										<p class="text-xs text-blue-200/60 mb-1">Choose your preferred language</p>
										<h3 class="text-sm font-bold text-white flex items-center gap-1">
											<div class="px-1.5 py-0.5 bg-blue-500/20 text-blue-300 text-xs rounded">
												Global
											</div>
										</h3>
									</div>

									<div class="relative language-dropdown">
										<button
											on:click={() => (showLanguageDropdown = !showLanguageDropdown)}
											class="w-full p-2 bg-white/5 border border-white/20 rounded text-white hover:bg-white/10 transition-all flex items-center justify-between text-sm"
										>
											<div class="flex items-center gap-2">
												<span>{languages[selectedLanguage]?.flag || '🌍'}</span>
												<span class="font-medium"
													>{languages[selectedLanguage]?.name || 'English'}</span
												>
											</div>
											<svg
												class="w-3 h-3 transition-transform {showLanguageDropdown
													? 'rotate-180'
													: ''}"
												fill="none"
												stroke="currentColor"
												viewBox="0 0 24 24"
											>
												<path
													stroke-linecap="round"
													stroke-linejoin="round"
													stroke-width="2"
													d="M19 9l-7 7-7-7"
												></path>
											</svg>
										</button>

										{#if showLanguageDropdown}
											<div
												class="absolute top-full left-0 right-0 mt-1 bg-slate-800/95 backdrop-blur-xl border border-white/20 rounded shadow-xl z-50 max-h-32 overflow-y-auto"
											>
												{#each Object.entries(languages) as [code, lang]}
													<button
														on:click={() => {
															selectedLanguage = code;
															showLanguageDropdown = false;
														}}
														class="w-full p-2 text-left hover:bg-white/10 flex items-center gap-2 text-white text-sm {selectedLanguage ===
														code
															? 'bg-blue-500/20'
															: ''}"
													>
														<span>{lang.flag}</span>
														<span class="truncate">{lang.name}</span>
														{#if selectedLanguage === code}
															<div class="w-1 h-1 bg-blue-400 rounded-full ml-auto"></div>
														{/if}
													</button>
												{/each}
											</div>
										{/if}
									</div>
								</div>

								<!-- Players Card -->
								<div
									class="bg-gradient-to-br from-purple-500/5 to-pink-500/5 border border-purple-400/20 rounded-lg p-3 hover:border-purple-400/30 transition-all"
								>
									<div class="flex items-center justify-between mb-3">
										<div>
											<h3 class="text-sm font-bold text-white flex items-center gap-2">
												<span class="text-purple-400">🧑‍🤝‍🧑</span>
												Players
											</h3>
											<p class="text-xs text-purple-200/60 mt-1">Number of people playing (3-12)</p>
										</div>
										<div
											class="px-3 py-1 bg-purple-500/20 text-purple-300 text-lg font-bold rounded-full border border-purple-400/30"
										>
											{numPlayers}
										</div>
									</div>

									<div class="space-y-3">
										<input
											type="range"
											min="3"
											max="12"
											bind:value={numPlayers}
											class="w-full h-2 bg-purple-600/20 rounded-full appearance-none cursor-pointer slider"
										/>

										<div class="flex justify-between text-xs text-slate-400 font-medium">
											<span>Min: 3</span>
											<span>Max: 12</span>
										</div>

										<!-- Enhanced player visualization -->
										<div class="bg-white/5 rounded-lg p-[14px] border border-white/10">
											<div class="flex gap-1.5 justify-center items-center">
												{#each Array(Math.min(numPlayers, 6)) as _, i}
													<div class="relative group">
														<div
															class="w-4 h-4 bg-gradient-to-br from-purple-500 to-pink-500 rounded-full shadow-sm border border-purple-400/30 hover:scale-110 transition-transform"
														></div>
														<div
															class="absolute -bottom-4 left-1/2 transform -translate-x-1/2 text-xs text-slate-400 opacity-0 group-hover:opacity-100 transition-opacity"
														>
															{i + 1}
														</div>
													</div>
												{/each}
												{#if numPlayers > 6}
													<div class="relative">
														<div
															class="w-4 h-4 bg-gradient-to-br from-slate-500 to-slate-700 rounded-full flex items-center justify-center text-xs text-white font-bold shadow-sm border border-slate-400/30"
														>
                            +{numPlayers - 6}
															
								
														</div>
													</div>
												{/if}
											</div>
										</div>
									</div>
								</div>
							</div>

							<!-- Right Column -->
							<div class="space-y-3">
								<!-- Theme Card -->
								<div
									class="bg-gradient-to-br from-emerald-500/5 to-teal-500/5 border border-emerald-400/20 rounded-lg p-3 hover:border-emerald-400/30 transition-all"
								>
									<div class="flex items-center justify-between mb-2">
										<h3 class="text-sm font-bold text-white flex items-center gap-1">
											<span class="text-emerald-400">🎨</span>
											Theme
										</h3>
										<p class="text-xs text-emerald-200/60 mb-1">Word category for the game</p>
										<h3 class="text-sm font-bold text-white flex items-center gap-1">
											<button
												on:click={toggleThemeInput}
												class="px-2 py-0.5 text-xs font-medium rounded transition-all {showThemeInput
													? 'bg-emerald-500/20 text-emerald-300'
													: 'bg-white/10 text-slate-300'}"
											>
												{showThemeInput ? 'Random' : 'Custom'}
											</button>
										</h3>
									</div>

									{#if showThemeInput}
										<input
											type="text"
											placeholder="Movies, Animals..."
											bind:value={selectedTheme}
											class="w-full p-2 bg-white/5 border border-white/20 rounded text-white placeholder-slate-400 focus:border-emerald-400 focus:outline-none text-sm mb-1"
										/>
										<div class="flex gap-1">
											{#each ['Movies', 'Food'] as suggestion}
												<button
													on:click={() => (selectedTheme = suggestion)}
													class="px-1.5 py-0.5 bg-emerald-500/10 text-emerald-300 text-xs rounded hover:bg-emerald-500/20"
												>
													{suggestion}
												</button>
											{/each}
										</div>
									{:else}
										<div
											class="p-2 bg-gradient-to-r from-emerald-500/10 to-teal-500/10 border border-emerald-400/20 rounded text-center"
										>
											<div
												class="text-emerald-300 font-medium text-sm flex items-center justify-center gap-1"
											>
												<span>🎲</span> Surprise Mode
												<button
													on:click={toggleThemeInput}
													class="px-2 py-0.5 text-xs font-medium rounded transition-all {showThemeInput
														? 'bg-emerald-500/20 text-emerald-300'
														: 'bg-white/10 text-slate-300'}"
												>
													{showThemeInput ? 'Random' : 'Click To Change'}
												</button>
											</div>
										</div>
									{/if}
								</div>

								<!-- Game Options Card -->
								<div
									class="bg-gradient-to-br from-amber-500/5 to-orange-500/5 border border-amber-400/20 rounded-lg p-3 hover:border-amber-400/30 transition-all"
								>
									<div class="flex items-center justify-between mb-2">
										<h3 class="text-sm font-bold text-white flex items-center gap-1">
											<span class="text-amber-400">⚙️</span>
											Options
										</h3>
										<p class="text-xs text-amber-200/60 mb-1">Game difficulty settings</p>
										<h3 class="text-sm font-bold text-white flex items-center gap-1">
											<div class="px-1.5 py-0.5 bg-amber-500/20 text-amber-300 text-xs rounded">
												Advanced
											</div>
										</h3>
									</div>

									<div class="space-y-2">
										<!-- Show Roles Toggle -->
										<div
											class="flex items-center justify-between p-2 bg-white/5 border border-white/10 rounded"
										>
											<div class="flex items-center gap-1 flex-1 min-w-0">
												<span class="text-amber-400 text-xs">👁️</span>
												<div>
													<span class="font-medium text-white text-xs">Show Roles</span>
													<div class="text-xs text-slate-400">Display player role with word</div>
												</div>
											</div>
											<button
												on:click={() => (showRoleInitially = !showRoleInitially)}
												class="relative w-8 h-4 rounded-full transition-all flex-shrink-0 {showRoleInitially
													? 'bg-gradient-to-r from-purple-500 to-pink-500'
													: 'bg-slate-600'}"
											>
												<div
													class="absolute w-3 h-3 bg-white rounded-full top-[1.5px] transition-transform {showRoleInitially
														? 'translate-x-4'
														: 'translate-x-0.5'}"
												>
													{#if showRoleInitially}
														<div
															class="w-full h-full flex items-center justify-center text-xs text-purple-600"
														></div>
													{/if}
												</div>
											</button>
										</div>

										<!-- Mystery Player Toggle -->
										<div
											class="flex items-center justify-between p-2 bg-white/5 border border-white/10 rounded"
										>
											<div class="flex items-center gap-1 flex-1 min-w-0">
												<span class="text-amber-400 text-xs">🤔</span>
												<div>
													<span class="font-medium text-white text-xs">Mystery Player</span>
													<div class="text-xs text-slate-400">Add player with no word</div>
												</div>
											</div>
											<button
												on:click={() => (includeDumbRole = !includeDumbRole)}
												class="relative w-8 h-4 rounded-full transition-all flex-shrink-0 {includeDumbRole
													? 'bg-gradient-to-r from-yellow-500 to-orange-500'
													: 'bg-slate-600'}"
											>
												<div
													class="absolute w-3 h-3 bg-white rounded-full top-[1.5px] transition-transform {includeDumbRole
														? 'translate-x-4'
														: 'translate-x-0.5'}"
												>
													{#if includeDumbRole}
														<div
															class="w-full h-full flex items-center justify-center text-xs text-yellow-600"
														></div>
													{/if}
												</div>
											</button>
										</div>
									</div>
								</div>
							</div>
						</div>

						<!-- Play Button Footer -->
						<div class="border-t border-white/10 pt-3">
							<button
								on:click={() => (gameState = 'setup')}
								class="relative w-full bg-gradient-to-r from-purple-600 to-pink-600 hover:from-purple-500 hover:to-pink-500 text-white font-bold py-2.5 md:py-3 px-4 rounded-lg transition-all shadow-lg hover:shadow-xl hover:scale-105 active:scale-95 group overflow-hidden"
							>
								<div
									class="absolute inset-0 bg-gradient-to-r from-transparent via-white/10 to-transparent -skew-x-12 translate-x-[-200%] group-hover:translate-x-[200%] transition-transform duration-1000"
								></div>
								<div class="relative flex items-center justify-center gap-2">
									<svg
										class="w-4 h-4 group-hover:scale-110 transition-transform"
										fill="currentColor"
										viewBox="0 0 24 24"
									>
										<path d="M8 5v14l11-7z" />
									</svg>
									<span class="font-black tracking-wide">Start Game</span>
								</div>
							</button>
						</div>
					</div>
				</div>
			{/if}

			<!-- Setup Screen -->
			{#if gameState === 'setup'}
				<div class="h-full flex flex-col lg:py-32 max-w-6xl mx-auto">
					<div class="text-center mb-4 lg:mb-6">
						<div
							class="inline-flex items-center bg-blue-500/20 border border-blue-500/30 rounded-full px-3 lg:px-4 py-1 lg:py-2 mb-2 lg:mb-4"
						>
							<div
								class="w-1.5 h-1.5 lg:w-2 lg:h-2 bg-blue-400 rounded-full mr-2 animate-pulse"
							></div>
							<span class="text-blue-200 text-xs lg:text-sm font-medium">Player Setup</span>
						</div>
						<h2 class="text-2xl lg:text-3xl font-black text-white mb-1 lg:mb-2">Who's Playing?</h2>
						<p class="text-sm lg:text-lg text-slate-300">Enter player names below</p>
					</div>

					<div
						class="flex-1 bg-white/5 backdrop-blur-xl border border-white/10 rounded-xl lg:rounded-2xl p-4 lg:p-6 flex flex-col min-h-0"
					>
						<!-- Player Grid -->
						<div
							class="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 xl:grid-cols-6 gap-3 lg:gap-4 mb-4 lg:mb-6 flex-1 overflow-y-auto custom-scrollbar"
						>
							{#each Array(numPlayers) as _, index}
								<div class="space-y-2">
									<label class="block text-xs font-medium text-slate-400">
										Player {index + 1}
									</label>
									<input
										type="text"
										bind:value={playerNames[index]}
										placeholder="Enter name..."
										class="w-full p-2 lg:p-3 bg-white/10 border border-white/20 rounded-lg text-white placeholder-slate-400 focus:border-purple-400 focus:outline-none text-sm"
									/>
								</div>
							{/each}
						</div>

						<!-- Action Buttons -->
						<div class="flex gap-3 lg:gap-4 pt-4 border-t border-white/20">
							<button
								on:click={() => (gameState = 'home')}
								class="flex-1 bg-white/10 hover:bg-white/20 text-white font-medium py-2 lg:py-3 px-4 lg:px-6 rounded-lg transition-all text-sm lg:text-base"
							>
								← Back
							</button>
							<button
								on:click={startGame}
								disabled={isGeneratingWords}
								class="flex-2 bg-gradient-to-r from-purple-600 to-pink-600 hover:from-purple-500 hover:to-pink-500 disabled:from-gray-600 disabled:to-gray-600 text-white font-bold py-2 lg:py-3 px-4 lg:px-6 rounded-lg transition-all disabled:opacity-50 text-sm lg:text-base"
							>
								{#if isGeneratingWords}
									<div class="flex items-center justify-center gap-2">
										<div
											class="animate-spin rounded-full h-3 w-3 lg:h-4 lg:w-4 border-2 border-white border-t-transparent"
										></div>
										<span>Generating...</span>
									</div>
								{:else}
									Generate & Start
								{/if}
							</button>
						</div>
					</div>
				</div>
			{/if}

			<!-- Game Screen -->
			{#if gameState === 'game'}
				<div class="h-full flex flex-col lg:py-32 lg:flex-row gap-3 lg:gap-4 max-w-7xl mx-auto">
					<!-- Left Action Panel -->
					<div
						class="lg:w-72 bg-white/5 backdrop-blur-xl border border-white/10 rounded-xl lg:rounded-2xl p-3 lg:p-4 flex flex-col order-2 lg:order-1"
					>
						<!-- Game Header -->
						<div class="text-center pb-3 lg:pb-4 border-b border-white/20 mb-3 lg:mb-4">
							<div
								class="inline-flex items-center bg-purple-500/20 border border-purple-500/30 rounded-full px-2 lg:px-3 py-1 mb-1 lg:mb-2"
							>
								<div
									class="w-1 h-1 lg:w-1.5 lg:h-1.5 bg-purple-400 rounded-full mr-1 lg:mr-2 animate-pulse"
								></div>
								<span class="text-purple-200 text-xs">In Progress</span>
							</div>
							<h3 class="text-sm lg:text-lg font-bold text-white">Find the Spy</h3>
						</div>

						<!-- Progress -->
						<div class="mb-3 lg:mb-4">
							<div class="flex justify-between text-xs text-slate-400 mb-2">
								<span>Progress</span>
								<span>{currentPlayerIndex + 1}/{numPlayers}</span>
							</div>
							<div class="w-full bg-white/20 rounded-full h-2">
								<div
									class="bg-gradient-to-r from-purple-500 to-pink-500 h-2 rounded-full transition-all duration-500"
									style="width: {((currentPlayerIndex + 1) / numPlayers) * 100}%"
								></div>
							</div>
						</div>

						<!-- Current Player -->
						<div
							class="bg-purple-500/10 border border-purple-400/20 rounded-lg p-2 lg:p-3 mb-3 lg:mb-4"
						>
							<div class="text-center">
								<div
									class="w-10 h-10 lg:w-12 lg:h-12 bg-gradient-to-br from-purple-500 to-pink-500 rounded-lg flex items-center justify-center text-sm lg:text-lg font-bold text-white mb-2 mx-auto"
								>
									{randomizedPlayerOrder[currentPlayerIndex] + 1}
								</div>
								<div class="text-xs lg:text-sm font-bold text-white truncate">
									{currentPlayer?.name || `Player ${randomizedPlayerOrder[currentPlayerIndex] + 1}`}
								</div>
								<div class="text-xs text-slate-400">Current Player</div>
							</div>
						</div>

						<!-- Action Button -->
						{#if !wordRevealed}
							<button
								on:click={revealWord}
								class="w-full bg-gradient-to-r from-purple-600 to-pink-600 hover:from-purple-500 hover:to-pink-500 text-white font-bold py-2 lg:py-3 px-3 lg:px-4 rounded-lg mb-3 lg:mb-4 transition-all hover:scale-105 active:scale-95"
							>
								<div class="flex flex-col items-center gap-1">
									<div class="w-4 h-4 lg:w-5 lg:h-5">{@html EyeIcon}</div>
									<span class="text-xs lg:text-sm">Reveal Word</span>
								</div>
							</button>
						{:else}
							<button
								on:click={nextPlayer}
								class="w-full bg-gradient-to-r from-emerald-600 to-green-600 hover:from-emerald-500 hover:to-green-500 text-white font-bold py-2 lg:py-3 px-3 lg:px-4 rounded-lg mb-3 lg:mb-4 transition-all hover:scale-105 active:scale-95"
							>
								<div class="flex flex-col items-center gap-1">
									<div class="w-4 h-4 lg:w-5 lg:h-5">{@html ArrowRightIcon}</div>
									<span class="text-xs lg:text-sm"
										>{currentPlayerIndex < numPlayers - 1 ? 'Next Player' : 'Discussion'}</span
									>
								</div>
							</button>
						{/if}

						<!-- Player List -->
						<div class="flex-1 overflow-y-auto custom-scrollbar">
							<h4 class="text-xs font-bold text-white uppercase tracking-wider mb-2">Players</h4>
							<div class="space-y-1">
								{#each randomizedPlayerOrder as playerIndex, listIndex}
									<div
										class="flex items-center gap-2 p-1.5 lg:p-2 rounded-lg {listIndex ===
										currentPlayerIndex
											? 'bg-purple-500/20'
											: listIndex < currentPlayerIndex
												? 'bg-green-500/10'
												: 'bg-white/5'}"
									>
										<div
											class="w-5 h-5 lg:w-6 lg:h-6 bg-gradient-to-br {listIndex ===
											currentPlayerIndex
												? 'from-purple-500 to-pink-500'
												: listIndex < currentPlayerIndex
													? 'from-green-500 to-emerald-500'
													: 'from-slate-600 to-slate-700'} rounded text-xs font-bold text-white flex items-center justify-center"
										>
											{playerIndex + 1}
										</div>
										<span class="text-xs text-white flex-1 truncate">
											{playerNames[playerIndex] || `Player ${playerIndex + 1}`}
										</span>
										{#if listIndex === currentPlayerIndex}
											<div
												class="w-1 h-1 lg:w-1.5 lg:h-1.5 bg-purple-400 rounded-full animate-pulse"
											></div>
										{:else if listIndex < currentPlayerIndex}
											<div class="text-green-400 text-xs">✓</div>
										{/if}
									</div>
								{/each}
							</div>
						</div>
					</div>

					<!-- Main Content Area -->
					<div
						class="flex-1 bg-white/5 backdrop-blur-xl border border-white/10 rounded-xl lg:rounded-2xl p-4 lg:p-8 flex flex-col justify-center order-1 lg:order-2 min-h-0"
					>
						<!-- Word Display -->
						<div class="flex-1 flex items-center justify-center min-h-0">
							{#if !wordRevealed}
								<div class="text-center max-w-md w-full">
									<div
										class="bg-slate-800/50 border-2 border-dashed border-white/30 rounded-xl lg:rounded-2xl p-6 lg:p-12"
									>
										<div
											class="w-12 h-12 lg:w-16 lg:h-16 mx-auto mb-4 lg:mb-6 text-slate-400 opacity-50"
										>
											{@html EyeOffIcon}
										</div>
										<h3 class="text-lg lg:text-xl font-bold text-slate-300 mb-2">Word Hidden</h3>
										<p class="text-sm lg:text-base text-slate-400">
											Click "Reveal Word" to see your secret word
										</p>
									</div>
								</div>
							{:else}
								<div class="text-center max-w-lg w-full">
									{#if showRoleInitially}
										<!-- Role & Word Display -->
										<div
											class="bg-white/10 border {currentPlayer?.role === 'imposter'
												? 'border-red-400/30'
												: currentPlayer?.role === 'dumb'
													? 'border-yellow-400/30'
													: 'border-green-400/30'} rounded-xl lg:rounded-2xl p-6 lg:p-8"
										>
											<div class="text-4xl lg:text-6xl mb-3 lg:mb-4">
												{currentPlayer?.role === 'imposter'
													? '🕵️'
													: currentPlayer?.role === 'dumb'
														? '🤔'
														: '👥'}
											</div>

											<h3
												class="text-lg lg:text-2xl font-bold {currentPlayer?.role === 'imposter'
													? 'text-red-400'
													: currentPlayer?.role === 'dumb'
														? 'text-yellow-400'
														: 'text-green-400'} mb-1 lg:mb-2"
											>
												{currentPlayer?.role === 'imposter'
													? 'You are the SPY!'
													: currentPlayer?.role === 'dumb'
														? 'You are the MYSTERY!'
														: 'You are a CIVILIAN'}
											</h3>

											<p class="text-sm lg:text-base text-slate-300 mb-4 lg:mb-6">
												{currentPlayer?.role === 'imposter'
													? 'Your word is different!'
													: currentPlayer?.role === 'dumb'
														? 'Figure out the word!'
														: 'Remember this word!'}
											</p>

											<div class="bg-white/20 rounded-lg lg:rounded-xl p-4 lg:p-6">
												<div class="text-2xl lg:text-4xl font-black text-white">
													{currentPlayer?.word}
												</div>
											</div>
										</div>
									{:else}
										<!-- Word Only Display -->
										<div class="bg-white/20 rounded-lg lg:rounded-xl p-4 lg:p-6">
											<div class="text-2xl lg:text-4xl font-black text-white">
												{currentPlayer?.word}
											</div>
										</div>
									{/if}
								</div>
							{/if}
						</div>
					</div>
				</div>
			{/if}

			<!-- Complete Screen -->
			{#if gameState === 'complete'}
				<div class="h-full flex flex-col justify-center items-center text-center">
					<div class="max-w-4xl w-full">
						<div class="mb-8">
							<div
								class="inline-flex items-center bg-green-500/20 border border-green-500/30 rounded-full px-4 py-2 mb-4"
							>
								<div class="w-2 h-2 bg-green-400 rounded-full mr-2 animate-pulse"></div>
								<span class="text-green-200 text-sm">Ready to Hunt</span>
							</div>
							<h2 class="text-5xl font-black text-white mb-4">Discussion Time!</h2>
							<p class="text-xl text-slate-300">
								Find the spy{includeDumbRole ? ' and mystery player' : ''}!
							</p>
						</div>

						<div class="bg-white/5 backdrop-blur-xl border border-white/10 rounded-2xl p-8 mb-6">
							<div class="flex gap-4">
								<button
									on:click={showImposter}
									class="flex-1 bg-gradient-to-r from-red-600 to-pink-600 hover:from-red-500 hover:to-pink-500 text-white font-bold py-4 px-6 rounded-xl transition-all hover:scale-105"
								>
									Reveal Results
								</button>

								{#if savedPlayerNames.length > 0}
									<button
										on:click={restartWithSamePlayers}
										class="flex-1 bg-gradient-to-r from-green-600 to-emerald-600 hover:from-green-500 hover:to-emerald-500 disabled:opacity-50 text-white font-bold py-4 px-6 rounded-xl transition-all hover:scale-105"
										disabled={isGeneratingWords}
									>
										{#if isGeneratingWords}
											<div class="flex items-center justify-center gap-2">
												<div
													class="animate-spin rounded-full h-4 w-4 border-2 border-white border-t-transparent"
												></div>
												<span>Starting...</span>
											</div>
										{:else}
											New Round
										{/if}
									</button>
								{/if}
								<button
									on:click={resetGame}
									class="bg-white/10 hover:bg-white/20 text-white font-medium py-3 px-6 rounded-xl transition-all"
								>
									New Game
								</button>
							</div>
						</div>
					</div>
				</div>
			{/if}

			<!-- Reveal Screen -->
			{#if gameState === 'reveal'}
				<div class="max-w-4xl mx-auto lg:py-32">
					<div class="text-center mb-4">
						<h2 class="text-2xl font-black text-white mb-6">Game Results</h2>
					</div>

					<div class="space-y-3 mb-10">
						<!-- Spy Result -->
						<div class="relative group">
							<div
								class="absolute inset-0 bg-gradient-to-br from-red-500/20 to-pink-500/20 rounded-3xl blur-2xl opacity-75"
							></div>
							<div
								class="relative bg-white/5 backdrop-blur-xl border border-red-400/20 rounded-3xl p-4 text-center transform hover:scale-105 transition-transform"
							>
								<h3 class="text-2xl font-bold text-red-400 mb-4">{imposter?.name}</h3>
								<p class="text-lg text-red-300 mb-6">was the SPY!</p>
								<div
									class="inline-block bg-red-500/20 border border-red-400/30 rounded-2xl px-8 py-4"
								>
									<span class="text-red-200 font-bold text-lg">Word: {gameData?.imposterWord}</span>
								</div>
							</div>
						</div>

						<!-- Mystery Player Result -->
						{#if dumbPlayer}
							<div class="relative group">
								<div
									class="absolute inset-0 bg-gradient-to-br from-yellow-500/20 to-orange-500/20 rounded-3xl blur-2xl opacity-75"
								></div>
								<div
									class="relative bg-white/5 backdrop-blur-xl border border-yellow-400/20 rounded-3xl p-4 text-center transform hover:scale-105 transition-transform"
								>
									<h3 class="text-2xl font-bold text-yellow-400 mb-4">{dumbPlayer?.name}</h3>
									<p class="text-lg text-yellow-300 mb-6">was the MYSTERY!</p>
									<div
										class="inline-block bg-yellow-500/20 border border-yellow-400/30 rounded-2xl px-4 py-2"
									>
										<span class="text-yellow-200 font-bold text-lg">Had no word!</span>
									</div>
								</div>
							</div>
						{/if}

						<!-- Civilian Word -->
						<div class="relative group">
							<div
								class="absolute inset-0 bg-gradient-to-br from-green-500/20 to-emerald-500/20 rounded-3xl blur-2xl opacity-75"
							></div>
							<div
								class="relative bg-white/5 backdrop-blur-xl border border-green-400/20 rounded-3xl p-10 text-center"
							>
								<h3 class="text-xl font-bold text-green-400 mb-4">Civilian Word</h3>
								<div class="text-lg font-black text-white">
									{gameData?.civilianWord}
								</div>
							</div>
						</div>
					</div>

					<!-- Actions -->
					<div class="bg-white/5 backdrop-blur-xl border border-white/10 rounded-3xl p-4">
						<div class="flex flex-col sm:flex-row gap-6">
							{#if savedPlayerNames.length > 0}
								<button
									on:click={restartWithSamePlayers}
									disabled={isGeneratingWords}
									class="flex-1 group relative bg-gradient-to-r from-purple-600 to-pink-600 hover:from-purple-500 hover:to-pink-500 disabled:from-gray-600 disabled:to-gray-600 text-white font-bold py-8 px-8 rounded-2xl transition-all duration-200 shadow-lg shadow-purple-500/25 hover:shadow-purple-500/40 disabled:shadow-none disabled:opacity-50 disabled:cursor-not-allowed hover:scale-105 disabled:hover:scale-100 active:scale-95"
								>
									{#if isGeneratingWords}
										<div class="flex items-center justify-center gap-3">
											<div
												class="animate-spin rounded-full h-6 w-6 border-2 border-white border-t-transparent"
											></div>
											<span class="text-xl">Starting...</span>
										</div>
									{:else}
										<div class="flex items-center justify-center gap-3">
											<div class="w-6 h-6">{@html RefreshIcon}</div>
											<span class="text-xl">Play Again</span>
											<div
												class="absolute right-6 opacity-0 group-hover:opacity-100 transition-opacity text-2xl"
											>
												✨
											</div>
										</div>
									{/if}
								</button>
							{/if}

							<button
								on:click={resetGame}
								class="flex-1 bg-white/5 hover:bg-white/10 text-white font-medium py-8 px-8 rounded-2xl transition-all duration-200 border border-white/10 hover:scale-105 active:scale-95 text-xl"
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
	/* Custom Slider Styling */
	.slider {
		background: linear-gradient(to right, #8b5cf6, #ec4899);
	}

	.slider::-webkit-slider-thumb {
		appearance: none;
		height: 24px;
		width: 24px;
		border-radius: 50%;
		background: linear-gradient(45deg, #8b5cf6, #ec4899);
		cursor: pointer;
		box-shadow: 0 0 20px rgba(139, 92, 246, 0.5);
		border: 3px solid white;
	}

	.slider::-moz-range-thumb {
		height: 24px;
		width: 24px;
		border-radius: 50%;
		background: linear-gradient(45deg, #8b5cf6, #ec4899);
		cursor: pointer;
		box-shadow: 0 0 20px rgba(139, 92, 246, 0.5);
		border: 3px solid white;
	}

	/* Custom Scrollbar */
	.custom-scrollbar::-webkit-scrollbar {
		width: 8px;
	}

	.custom-scrollbar::-webkit-scrollbar-track {
		background: rgba(255, 255, 255, 0.05);
		border-radius: 10px;
	}

	.custom-scrollbar::-webkit-scrollbar-thumb {
		background: linear-gradient(to bottom, #8b5cf6, #ec4899);
		border-radius: 10px;
	}

	.custom-scrollbar::-webkit-scrollbar-thumb:hover {
		background: linear-gradient(to bottom, #7c3aed, #db2777);
	}

	/* Focus Styles */
	button:focus-visible,
	input:focus-visible {
		outline: 2px solid #8b5cf6;
		outline-offset: 2px;
	}

	/* Mobile Optimizations */
	@media (max-width: 1023px) {
		.min-h-screen {
			min-height: 100vh;
			min-height: 100dvh;
		}

		/* Stack layout on mobile for game screen */
		.flex.gap-6.h-screen {
			flex-direction: column;
			height: auto;
			min-height: 100vh;
		}

		.w-80 {
			width: 100%;
			flex-shrink: 0;
		}

		/* Mobile sidebar height adjustment */
		.w-80 .rounded-3xl.h-full {
			height: auto;
			max-height: 50vh;
		}
	}

	/* Glassmorphism Effect */
	.glass {
		background: rgba(255, 255, 255, 0.05);
		backdrop-filter: blur(20px);
		border: 1px solid rgba(255, 255, 255, 0.1);
	}

	/* Animated Gradient Text */
	.gradient-text {
		background: linear-gradient(-45deg, #8b5cf6, #ec4899, #06b6d4, #10b981);
		background-size: 400% 400%;
		animation: gradient 3s ease infinite;
		-webkit-background-clip: text;
		-webkit-text-fill-color: transparent;
		background-clip: text;
	}

	@keyframes gradient {
		0% {
			background-position: 0% 50%;
		}
		50% {
			background-position: 100% 50%;
		}
		100% {
			background-position: 0% 50%;
		}
	}

	/* Hover Glow Effects */
	.glow-purple:hover {
		box-shadow: 0 0 30px rgba(139, 92, 246, 0.4);
	}

	.glow-pink:hover {
		box-shadow: 0 0 30px rgba(236, 72, 153, 0.4);
	}

	.glow-blue:hover {
		box-shadow: 0 0 30px rgba(6, 182, 212, 0.4);
	}

	/* Pulse Animation */
	@keyframes pulse-slow {
		0%,
		100% {
			opacity: 1;
		}
		50% {
			opacity: 0.5;
		}
	}

	.animate-pulse-slow {
		animation: pulse-slow 3s cubic-bezier(0.4, 0, 0.6, 1) infinite;
	}

	/* Floating Animation */
	@keyframes float {
		0%,
		100% {
			transform: translateY(0px);
		}
		50% {
			transform: translateY(-10px);
		}
	}

	.animate-float {
		animation: float 3s ease-in-out infinite;
	}

	/* Bounce In Animation */
	@keyframes bounceIn {
		0% {
			transform: scale(0.3);
			opacity: 0;
		}
		50% {
			transform: scale(1.05);
		}
		70% {
			transform: scale(0.9);
		}
		100% {
			transform: scale(1);
			opacity: 1;
		}
	}

	.animate-bounce-in {
		animation: bounceIn 0.6s ease-out;
	}

	/* Slide In Animation */
	@keyframes slideIn {
		0% {
			transform: translateX(-100%);
			opacity: 0;
		}
		100% {
			transform: translateX(0);
			opacity: 1;
		}
	}

	.animate-slide-in {
		animation: slideIn 0.5s ease-out;
	}

	/* Button Press Animation */
	.btn-press:active {
		transform: scale(0.95);
		transition: transform 0.1s;
	}

	/* Card Hover Effects */
	.card-hover:hover {
		transform: translateY(-2px);
		box-shadow: 0 20px 40px rgba(0, 0, 0, 0.1);
	}

	/* Gradient Borders */
	.gradient-border {
		background: linear-gradient(45deg, #8b5cf6, #ec4899, #06b6d4);
		padding: 2px;
		border-radius: 1.5rem;
	}

	.gradient-border > div {
		background: rgba(15, 23, 42, 0.9);
		border-radius: calc(1.5rem - 2px);
	}

	/* Loading Spinner Enhancement */
	@keyframes spin-glow {
		0% {
			transform: rotate(0deg);
			filter: drop-shadow(0 0 10px rgba(139, 92, 246, 0.5));
		}
		100% {
			transform: rotate(360deg);
			filter: drop-shadow(0 0 10px rgba(236, 72, 153, 0.5));
		}
	}

	.animate-spin-glow {
		animation: spin-glow 1s linear infinite;
	}
</style>
