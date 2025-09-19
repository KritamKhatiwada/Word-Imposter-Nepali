<script lang="ts">
    import Nav from '../components/Nav.svelte';
    import { GoogleGenerativeAI } from '@google/generative-ai';
    import { database } from '../../firebase'; // Adjust path based on your structure
    import { 
        ref, 
        set, 
        get, 
        push, 
        onValue, 
        off, 
        update,
        remove,
        onDisconnect,
        serverTimestamp,
        query,
        orderByChild,
        equalTo,
        limitToLast
    } from 'firebase/database';
    import { onDestroy } from 'svelte';
    
    interface Player {
        id: string;
        name: string;
        role: 'imposter' | 'civilian' | 'dumb';
        word: string;
        isHost: boolean;
        hasRevealed: boolean;
        lastSeen: number;
    }
    
    interface GameRoom {
        roomCode: string;
        roomName: string;
        isPublic: boolean;
        maxPlayers: number;
        players: { [playerId: string]: Player };
        gameStarted: boolean;
        gameComplete: boolean;
        civilianWord: string;
        imposterWord: string;
        hostId: string;
        createdAt: number;
        lastActivity: number;
        settings: {
            language: 'english' | 'nepali';
            showRoles: boolean;
            includeDumbRole: boolean;
        };
    }
    
    interface PublicRoom {
        roomCode: string;
        roomName: string;
        playerCount: number;
        maxPlayers: number;
        gameStarted: boolean;
        language: 'english' | 'nepali';
        createdAt: number;
        hostName: string;
    }
    
    interface WordPair {
        civilian: string;
        imposter: string;
    }

    // Firebase error interface
    interface FirebaseError extends Error {
        code?: string;
    }
    
    let gameState: 'home' | 'create' | 'join' | 'browse' | 'lobby' | 'game' | 'complete' | 'reveal' = 'home';
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
    let roomListener: any = null;
    let publicRoomsListener: any = null;
    let connectionStatus: 'connected' | 'disconnected' | 'connecting' = 'connected';
    let cleanupInterval: any = null;
    
    // New variables for searchable rooms
    let roomName: string = '';
    let isPublicRoom: boolean = true;
    let maxPlayers: number = 8;
    let publicRooms: PublicRoom[] = [];
    let searchQuery: string = '';
    let selectedRoom: PublicRoom | null = null;
    let isLoadingRooms: boolean = false;
    
    // Icons
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
    const SearchIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
        <circle cx="11" cy="11" r="8"/>
        <path d="m21 21-4.35-4.35"/>
    </svg>`;
    const GlobeIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
        <circle cx="12" cy="12" r="10"/>
        <line x1="2" y1="12" x2="22" y2="12"/>
        <path d="M12 2a15.3 15.3 0 0 1 4 10 15.3 15.3 0 0 1-4 10 15.3 15.3 0 0 1-4-10 15.3 15.3 0 0 1 4-10z"/>
    </svg>`;
    const LockIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
        <rect x="3" y="11" width="18" height="11" rx="2" ry="2"/>
        <circle cx="12" cy="16" r="1"/>
        <path d="M7 11V7a5 5 0 0 1 10 0v4"/>
    </svg>`;
    const UnlockIcon = `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
        <rect x="3" y="11" width="18" height="11" rx="2" ry="2"/>
        <circle cx="12" cy="16" r="1"/>
        <path d="M7 11V7a5 5 0 0 1 9.9-1"/>
    </svg>`;
    const StarIcon = `<svg viewBox="0 0 24 24" fill="currentColor">
        <path d="M12 2l3.09 6.26L22 9.27l-5 4.87 1.18 6.88L12 17.77l-6.18 3.25L7 14.14 2 9.27l6.91-1.01L12 2z"/>
    </svg>`;
    const CrownIcon = `<svg viewBox="0 0 24 24" fill="currentColor">
        <path d="M5 16L3 12l5.5 2L12 7l3.5 7L21 12l-2 4H5zm2.7-2h8.6l.9-1.4-3.1-1.4L12 14l-2.1-2.8-3.1 1.4L7.7 14z"/>
    </svg>`;
    
    // Generate a random room code
    function generateRoomCode(): string {
        return Math.random().toString(36).substring(2, 8).toUpperCase();
    }
    
    // Generate unique player ID
    function generatePlayerId(): string {
        return Math.random().toString(36).substring(2, 15);
    }
    
    // Setup room cleanup - runs every 5 minutes
    function setupRoomCleanup(): void {
        if (cleanupInterval) {
            clearInterval(cleanupInterval);
            cleanupInterval = null;
        }
        
        cleanupInterval = setInterval(async () => {
            try {
                // Only run cleanup if user is authenticated/has permissions
                if (!roomCode && !playerId) return;
                
                const roomsRef = ref(database, 'rooms');
                const snapshot = await get(roomsRef);
                
                if (snapshot.exists()) {
                    const rooms = snapshot.val();
                    const now = Date.now();
                    const INACTIVE_TIME = 30 * 60 * 1000; // 30 minutes
                    
                    for (const [code, room] of Object.entries(rooms as { [key: string]: GameRoom })) {
                        // Check if room is inactive
                        if (now - room.lastActivity > INACTIVE_TIME) {
                            // Check if any players are still active
                            const activePlayers = Object.values(room.players || {}).filter(
                                player => now - player.lastSeen < INACTIVE_TIME
                            );
                            
                            if (activePlayers.length === 0) {
                                await remove(ref(database, `rooms/${code}`));
                                console.log(`Cleaned up inactive room: ${code}`);
                            }
                        }
                    }
                }
            } catch (error) {
                const firebaseError = error as FirebaseError;
                console.error('Error cleaning up rooms:', error);
                // Stop cleanup on permission errors
                if (firebaseError.code?.includes('permission-denied')) {
                    if (cleanupInterval) {
                        clearInterval(cleanupInterval);
                        cleanupInterval = null;
                    }
                }
            }
        }, 5 * 60 * 1000); // Run every 5 minutes
    }
    
    // Setup public rooms listener
    function setupPublicRoomsListener(): void {
        // Clean up existing listener properly
        if (publicRoomsListener) {
            try {
                if (typeof publicRoomsListener === 'function') {
                    publicRoomsListener(); // Call the unsubscribe function
                } else {
                    off(publicRoomsListener);
                }
            } catch (error) {
                console.warn('Error removing public rooms listener:', error);
            }
            publicRoomsListener = null;
        }
        
        isLoadingRooms = true;
        const roomsRef = ref(database, 'rooms');
        
        try {
            publicRoomsListener = onValue(roomsRef, (snapshot) => {
                const data = snapshot.val();
                publicRooms = [];
                
                if (data) {
                    const rooms = Object.entries(data).map(([code, room]: [string, any]) => {
                        const roomData = room as GameRoom;
                        if (roomData.isPublic && !roomData.gameStarted) {
                            return {
                                roomCode: code,
                                roomName: roomData.roomName,
                                playerCount: Object.keys(roomData.players || {}).length,
                                maxPlayers: roomData.maxPlayers,
                                gameStarted: roomData.gameStarted,
                                language: roomData.settings.language,
                                createdAt: roomData.createdAt,
                                hostName: Object.values(roomData.players || {}).find(p => p.isHost)?.name || 'Unknown'
                            } as PublicRoom;
                        }
                        return null;
                    })
                    .filter(Boolean)
                    .sort((a, b) => b!.createdAt - a!.createdAt) as PublicRoom[];
                    
                    publicRooms = rooms;
                }
                
                isLoadingRooms = false;
            }, (error) => {
                const firebaseError = error as FirebaseError;
                console.error('Public rooms listener error:', error);
                isLoadingRooms = false;
                
                // Handle permission denied by showing appropriate message
                if (firebaseError.code === 'permission-denied') {
                    alert('Unable to load public rooms. Please check Firebase permissions or try again later.');
                    gameState = 'home';
                }
            });
        } catch (error) {
            console.error('Error setting up public rooms listener:', error);
            isLoadingRooms = false;
            alert('Failed to connect to room browser. Please try again later.');
            gameState = 'home';
        }
    }
    
    // Setup room listener
    function setupRoomListener(code: string): void {
        // Clean up existing listener properly
        if (roomListener) {
            try {
                if (typeof roomListener === 'function') {
                    roomListener(); // Call the unsubscribe function
                } else {
                    off(roomListener);
                }
            } catch (error) {
                console.warn('Error removing room listener:', error);
            }
            roomListener = null;
        }
        
        const roomRef = ref(database, `rooms/${code}`);
        
        try {
            roomListener = onValue(roomRef, (snapshot) => {
                const data = snapshot.val();
                if (data) {
                    gameRoom = {
                        ...data,
                        roomCode: code
                    };
                    
                    // Update current player reference
                    if (playerId && gameRoom.players && gameRoom.players[playerId]) {
                        currentPlayer = gameRoom.players[playerId];
                    }
                    
                    // Update game state based on room state
                    if (gameRoom.gameComplete) {
                        gameState = 'reveal';
                    } else if (gameRoom.gameStarted) {
                        gameState = 'game';
                    } else {
                        gameState = 'lobby';
                    }
                } else {
                    // Room doesn't exist or was deleted
                    if (gameState !== 'home' && gameState !== 'create' && gameState !== 'join' && gameState !== 'browse') {
                        alert('Room no longer exists!');
                        resetGame();
                    }
                }
            }, (error) => {
                console.error('Room listener error:', error);
            });
        } catch (error) {
            console.error('Error setting up room listener:', error);
        }
    }
    
    // Update player presence and room activity
    async function updatePlayerPresence(): Promise<void> {
        if (!roomCode || !playerId) return;
        
        try {
            const now = Date.now();
            const updates: { [key: string]: any } = {};
            
            updates[`rooms/${roomCode}/players/${playerId}/lastSeen`] = now;
            updates[`rooms/${roomCode}/lastActivity`] = now;
            
            await update(ref(database), updates);
            
            // Set up disconnect handler
            onDisconnect(ref(database, `rooms/${roomCode}/players/${playerId}/lastSeen`)).set(now);
        } catch (error) {
            console.error('Error updating presence:', error);
        }
    }
    
    // Create a new room
    async function createRoom(): Promise<void> {
        if (!playerName.trim()) {
            alert('Please enter your name!');
            return;
        }
        
        if (isPublicRoom && !roomName.trim()) {
            alert('Please enter a room name for public rooms!');
            return;
        }
        
        connectionStatus = 'connecting';
        
        try {
            roomCode = generateRoomCode();
            playerId = generatePlayerId();
            const now = Date.now();
            
            currentPlayer = {
                id: playerId,
                name: playerName.trim(),
                role: 'civilian',
                word: '',
                isHost: true,
                hasRevealed: false,
                lastSeen: now
            };
            
            const newRoom: GameRoom = {
                roomCode,
                roomName: isPublicRoom ? roomName.trim() : `${playerName.trim()}'s Room`,
                isPublic: isPublicRoom,
                maxPlayers: maxPlayers,
                players: {
                    [playerId]: currentPlayer
                },
                gameStarted: false,
                gameComplete: false,
                civilianWord: '',
                imposterWord: '',
                hostId: playerId,
                createdAt: now,
                lastActivity: now,
                settings: {
                    language: selectedLanguage,
                    showRoles: showRoleInitially,
                    includeDumbRole: includeDumbRole
                }
            };
            
            // Save to Firebase
            await set(ref(database, `rooms/${roomCode}`), newRoom);
            
            // Setup listeners
            setupRoomListener(roomCode);
            updatePlayerPresence();
            
            gameState = 'lobby';
            connectionStatus = 'connected';
        } catch (error) {
            console.error('Error creating room:', error);
            alert('Failed to create room. Please try again.');
            connectionStatus = 'disconnected';
        }
    }
    
    // Join an existing room by code
    async function joinRoom(): Promise<void> {
        if (!playerName.trim()) {
            alert('Please enter your name!');
            return;
        }
        
        if (!joinRoomCode.trim()) {
            alert('Please enter a room code!');
            return;
        }
        
        connectionStatus = 'connecting';
        
        try {
            const code = joinRoomCode.trim().toUpperCase();
            await joinRoomByCode(code);
        } catch (error) {
            console.error('Error joining room:', error);
            alert('Failed to join room. Please try again.');
            connectionStatus = 'disconnected';
        }
    }
    
    // Join room by code (shared logic)
    async function joinRoomByCode(code: string): Promise<void> {
        const roomRef = ref(database, `rooms/${code}`);
        const snapshot = await get(roomRef);
        
        if (!snapshot.exists()) {
            alert('Room not found!');
            connectionStatus = 'connected';
            return;
        }
        
        const room = snapshot.val();
        
        if (room.gameStarted) {
            alert('Game has already started!');
            connectionStatus = 'connected';
            return;
        }
        
        const playerCount = Object.keys(room.players || {}).length;
        if (playerCount >= room.maxPlayers) {
            alert('Room is full!');
            connectionStatus = 'connected';
            return;
        }
        
        // Check if name already exists
        const players = room.players || {};
        const nameExists = Object.values(players).some((p: any) => 
            p.name.toLowerCase() === playerName.trim().toLowerCase()
        );
        
        if (nameExists) {
            alert('A player with this name already exists in the room!');
            connectionStatus = 'connected';
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
            hasRevealed: false,
            lastSeen: Date.now()
        };
        
        // Add player to room
        await set(ref(database, `rooms/${roomCode}/players/${playerId}`), currentPlayer);
        await update(ref(database, `rooms/${roomCode}`), { lastActivity: Date.now() });
        
        // Setup listeners
        setupRoomListener(roomCode);
        updatePlayerPresence();
        
        gameState = 'lobby';
        connectionStatus = 'connected';
    }
    
    // Join a public room
    async function joinPublicRoom(room: PublicRoom): Promise<void> {
        if (!playerName.trim()) {
            alert('Please enter your name!');
            return;
        }
        
        connectionStatus = 'connecting';
        selectedRoom = room;
        
        try {
            await joinRoomByCode(room.roomCode);
            selectedRoom = null;
        } catch (error) {
            console.error('Error joining public room:', error);
            alert('Failed to join room. Please try again.');
            connectionStatus = 'disconnected';
            selectedRoom = null;
        }
    }
    
    // Generate words with Gemini
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
    
    // Start the game (host only)
    async function startGame(): Promise<void> {
        if (!gameRoom || !currentPlayer?.isHost || !roomCode) return;
        
        const players = Object.values(gameRoom.players);
        if (players.length < 3) {
            alert('At least 3 players are required!');
            return;
        }
        
        isGeneratingWords = true;
        
        try {
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
            const playerIds = Object.keys(gameRoom.players);
            const playerCount = playerIds.length;
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
            const updates: { [key: string]: any } = {};
            
            playerIds.forEach((playerId, index) => {
                if (index === imposterIndex) {
                    updates[`players/${playerId}/role`] = 'imposter';
                    updates[`players/${playerId}/word`] = wordPair.imposter;
                } else if (index === dumbIndex) {
                    updates[`players/${playerId}/role`] = 'dumb';
                    updates[`players/${playerId}/word`] = '???';
                } else {
                    updates[`players/${playerId}/role`] = 'civilian';
                    updates[`players/${playerId}/word`] = wordPair.civilian;
                }
                updates[`players/${playerId}/hasRevealed`] = false;
            });
            
            updates['civilianWord'] = wordPair.civilian;
            updates['imposterWord'] = wordPair.imposter;
            updates['gameStarted'] = true;
            updates['lastActivity'] = Date.now();
            
            await update(ref(database, `rooms/${roomCode}`), updates);
            
            wordRevealed = false;
            isGeneratingWords = false;
        } catch (error) {
            console.error('Error starting game:', error);
            alert('Failed to start game. Please try again.');
            isGeneratingWords = false;
        }
    }
    
    function copyRoomCode(): void {
        navigator.clipboard.writeText(roomCode).then(() => {
            // Could show a toast notification here
        });
    }
    
    async function revealWord(): Promise<void> {
        if (!currentPlayer || !roomCode) return;
        
        try {
            wordRevealed = true;
            await update(ref(database, `rooms/${roomCode}/players/${playerId}`), {
                hasRevealed: true
            });
            await update(ref(database, `rooms/${roomCode}`), { lastActivity: Date.now() });
        } catch (error) {
            console.error('Error revealing word:', error);
        }
    }
    
    async function showResults(): Promise<void> {
        if (!gameRoom || !roomCode) return;
        
        try {
            await update(ref(database, `rooms/${roomCode}`), {
                gameComplete: true,
                lastActivity: Date.now()
            });
        } catch (error) {
            console.error('Error showing results:', error);
        }
    }
    
    // Cleanup room when leaving
    async function cleanupPlayerFromRoom(): Promise<void> {
        if (!roomCode || !playerId) return;
        
        try {
            // Remove player from room
            await remove(ref(database, `rooms/${roomCode}/players/${playerId}`));
            
            // Check if room is empty, if so delete it
            const roomSnapshot = await get(ref(database, `rooms/${roomCode}`));
            if (roomSnapshot.exists()) {
                const room = roomSnapshot.val();
                const remainingPlayers = Object.keys(room.players || {});
                
                if (remainingPlayers.length === 0) {
                    await remove(ref(database, `rooms/${roomCode}`));
                } else {
                    // If the leaving player was the host, assign a new host
                    if (room.hostId === playerId) {
                        const newHostId = remainingPlayers[0];
                        await update(ref(database, `rooms/${roomCode}`), {
                            hostId: newHostId,
                            lastActivity: Date.now()
                        });
                        await update(ref(database, `rooms/${roomCode}/players/${newHostId}`), {
                            isHost: true
                        });
                    }
                }
            }
        } catch (error) {
            console.error('Error cleaning up player from room:', error);
        }
    }
    
    function resetGame(): void {
        // Clean up listeners properly with null checks
        if (roomListener) {
            try {
                if (typeof roomListener === 'function') {
                    roomListener(); // Call the unsubscribe function
                } else {
                    off(roomListener);
                }
            } catch (error) {
                console.warn('Error removing room listener:', error);
            }
            roomListener = null;
        }
        
        if (publicRoomsListener) {
            try {
                if (typeof publicRoomsListener === 'function') {
                    publicRoomsListener(); // Call the unsubscribe function
                } else {
                    off(publicRoomsListener);
                }
            } catch (error) {
                console.warn('Error removing public rooms listener:', error);
            }
            publicRoomsListener = null;
        }
        
        // Clean up player from room if they were in one
        if (roomCode && playerId) {
            cleanupPlayerFromRoom();
        }
        
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
        connectionStatus = 'connected';
        roomName = '';
        isPublicRoom = true;
        maxPlayers = 8;
        searchQuery = '';
        selectedRoom = null;
    }
    
    async function startNewGame(): Promise<void> {
        if (!gameRoom || !currentPlayer?.isHost || !roomCode) return;
        
        try {
            // Reset game state but keep players
            const updates: { [key: string]: any } = {};
            
            Object.keys(gameRoom.players).forEach(playerId => {
                updates[`players/${playerId}/role`] = 'civilian';
                updates[`players/${playerId}/word`] = '';
                updates[`players/${playerId}/hasRevealed`] = false;
            });
            
            updates['gameStarted'] = false;
            updates['gameComplete'] = false;
            updates['civilianWord'] = '';
            updates['imposterWord'] = '';
            updates['lastActivity'] = Date.now();
            
            await update(ref(database, `rooms/${roomCode}`), updates);
            
            wordRevealed = false;
        } catch (error) {
            console.error('Error starting new game:', error);
            alert('Failed to start new game. Please try again.');
        }
    }
    
    function goToCreateRoom(): void {
        gameState = 'create';
    }
    
    function goToJoinRoom(): void {
        gameState = 'join';
    }
    
    function goToBrowseRooms(): void {
        gameState = 'browse';
        setupPublicRoomsListener();
    }
    
    function backToHome(): void {
        if (publicRoomsListener) {
            try {
                if (typeof publicRoomsListener === 'function') {
                    publicRoomsListener(); // Call the unsubscribe function
                } else {
                    off(publicRoomsListener);
                }
            } catch (error) {
                console.warn('Error removing public rooms listener:', error);
            }
            publicRoomsListener = null;
        }
        gameState = 'home';
    }
    
    // Keep player presence updated
    let presenceInterval: any;
    $: if (roomCode && playerId) {
        if (presenceInterval) clearInterval(presenceInterval);
        presenceInterval = setInterval(updatePlayerPresence, 30000); // Update every 30 seconds
    }
    
    // Filter public rooms based on search query
    $: filteredRooms = publicRooms.filter(room => 
        room.roomName.toLowerCase().includes(searchQuery.toLowerCase()) ||
        room.hostName.toLowerCase().includes(searchQuery.toLowerCase()) ||
        room.roomCode.toLowerCase().includes(searchQuery.toLowerCase())
    );
    
    // Get game statistics with null checks
    $: playersRevealed = gameRoom ? Object.values(gameRoom.players).filter(p => p.hasRevealed).length : 0;
    $: totalPlayers = gameRoom ? Object.keys(gameRoom.players).length : 0;
    $: imposterPlayer = gameRoom ? Object.values(gameRoom.players).find(p => p.role === 'imposter') : null;
    $: dumbPlayer = gameRoom ? Object.values(gameRoom.players).find(p => p.role === 'dumb') : null;
    
    // Initialize cleanup on component mount - but only when needed
    let cleanupInitialized = false;
    $: if (!cleanupInitialized && (roomCode || playerId)) {
        setupRoomCleanup();
        cleanupInitialized = true;
    }
    
    // Cleanup on component destroy
    onDestroy(() => {
        if (roomListener) {
            try {
                if (typeof roomListener === 'function') {
                    roomListener(); // Call the unsubscribe function
                } else {
                    off(roomListener);
                }
            } catch (error) {
                console.warn('Error removing room listener on destroy:', error);
            }
            roomListener = null;
        }
        
        if (publicRoomsListener) {
            try {
                if (typeof publicRoomsListener === 'function') {
                    publicRoomsListener(); // Call the unsubscribe function
                } else {
                    off(publicRoomsListener);
                }
            } catch (error) {
                console.warn('Error removing public rooms listener on destroy:', error);
            }
            publicRoomsListener = null;
        }
        
        if (presenceInterval) {
            clearInterval(presenceInterval);
            presenceInterval = null;
        }
        
        if (cleanupInterval) {
            clearInterval(cleanupInterval);
            cleanupInterval = null;
        }
        
        // Clean up player from room if they were in one
        if (roomCode && playerId) {
            cleanupPlayerFromRoom();
        }
    });
</script>
<Nav/>
<div class="min-h-screen bg-gradient-to-br from-gray-50 via-white to-gray-100">

    <!-- Connection Status -->
    {#if connectionStatus !== 'connected'}
        <div class="fixed top-4 left-1/2 transform -translate-x-1/2 z-50">
            <div class="bg-yellow-100 border border-yellow-400 text-yellow-700 px-4 py-2 rounded-lg shadow-lg flex items-center space-x-2">
                <div class="animate-spin w-4 h-4">{@html RefreshIcon}</div>
                <span class="text-sm font-medium">
                    {connectionStatus === 'connecting' ? 'Connecting...' : 'Reconnecting...'}
                </span>
            </div>
        </div>
    {/if}

    <!-- Home Screen -->
    {#if gameState === 'home'}
        <div class="container mx-auto px-4  py-8 md:py-16">
            <div class="max-w-md mx-auto">
                
                <!-- Header -->
                <div class="text-center mt-16g mb-12">
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
                    
                    <!-- Browse Public Rooms Button -->
                    <button
                        on:click={goToBrowseRooms}
                        class="w-full bg-gradient-to-r from-purple-500 to-indigo-600 hover:from-purple-600 hover:to-indigo-700 text-white font-black py-8 px-8 rounded-2xl transition-all duration-200 shadow-xl hover:shadow-purple-500/25 flex items-center justify-center space-x-4 hover:scale-105 transform hover:cursor-pointer"
                    >
                        <div class="w-16 h-16 bg-white/20 rounded-2xl flex items-center justify-center">
                            <div class="w-10 h-10 text-white">{@html GlobeIcon}</div>
                        </div>
                        <div class="text-left">
                            <div class="text-2xl font-black">BROWSE ROOMS</div>
                            <div class="text-purple-100 text-sm font-medium">Find and join public games</div>
                        </div>
                    </button>
                    
                    <!-- Join Room Button -->
                    <button
                        on:click={goToJoinRoom}
                        class="w-full bg-gradient-to-r from-blue-500 to-cyan-600 hover:from-blue-600 hover:to-cyan-700 text-white font-black py-6 px-8 rounded-2xl transition-all duration-200 shadow-xl hover:shadow-blue-500/25 flex items-center justify-center space-x-4 hover:scale-105 transform hover:cursor-pointer"
                    >
                        <div class="w-12 h-12 bg-white/20 rounded-xl flex items-center justify-center">
                            <div class="w-8 h-8 text-white">{@html LoginIcon}</div>
                        </div>
                        <div class="text-left">
                            <div class="text-xl font-black">JOIN BY CODE</div>
                            <div class="text-blue-100 text-sm font-medium">Enter a private room code</div>
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

    <!-- Browse Public Rooms Screen -->
    {#if gameState === 'browse'}
        <div class="container mx-auto px-4 py-8 md:py-16">
            <div class="max-w-lg mx-auto">
                
                <div class="bg-white rounded-2xl shadow-xl border border-gray-100 p-8">
                    <div class="text-center mb-6">
                        <div class="w-16 h-16 bg-gradient-to-br from-purple-500 to-indigo-600 rounded-2xl mx-auto mb-4 flex items-center justify-center shadow-xl">
                            <div class="w-8 h-8 text-white">{@html GlobeIcon}</div>
                        </div>
                        <h2 class="text-2xl font-black text-gray-900 mb-2">Public Rooms</h2>
                        <p class="text-gray-600 text-sm">Join an existing public game</p>
                    </div>
                    
                    <!-- Player Name Input -->
                    <div class="mb-6">
                        <label class="block text-sm font-bold text-gray-700 mb-2">Your Name</label>
                        <input
                            type="text"
                            bind:value={playerName}
                            placeholder="Enter your name..."
                            class="w-full p-4 border-2 border-gray-200 rounded-xl focus:border-purple-500 focus:outline-none transition-all bg-gray-50 hover:bg-gray-100"
                        />
                    </div>
                    
                    <!-- Search -->
                    <div class="mb-4">
                        <div class="relative">
                            <input
                                type="text"
                                bind:value={searchQuery}
                                placeholder="Search rooms..."
                                class="w-full p-3 pl-10 border-2 border-gray-200 rounded-xl focus:border-purple-500 focus:outline-none transition-all bg-gray-50 hover:bg-gray-100"
                            />
                            <div class="absolute left-3 top-1/2 transform -translate-y-1/2 w-5 h-5 text-gray-400">
                                {@html SearchIcon}
                            </div>
                        </div>
                    </div>
                    
                    <!-- Rooms List -->
                    <div class="space-y-3 max-h-80 overflow-y-auto mb-6">
                        {#if isLoadingRooms}
                            <div class="text-center py-8">
                                <div class="animate-spin rounded-full h-8 w-8 border-2 border-purple-500 border-t-transparent mx-auto mb-2"></div>
                                <p class="text-gray-500">Loading rooms...</p>
                            </div>
                        {:else if filteredRooms.length === 0}
                            <div class="text-center py-8">
                                <div class="text-4xl mb-3">🎭</div>
                                <p class="text-gray-500 font-medium">
                                    {publicRooms.length === 0 ? 'No public rooms available' : 'No rooms match your search'}
                                </p>
                                <p class="text-gray-400 text-sm mt-1">Create your own room to get started!</p>
                            </div>
                        {:else}
                            {#each filteredRooms as room}
                                <div class="p-4 bg-gray-50 rounded-xl border border-gray-100 hover:bg-gray-100 transition-all">
                                    <div class="flex items-center justify-between mb-2">
                                        <h3 class="font-bold text-gray-900 truncate pr-2">{room.roomName}</h3>
                                        <div class="flex items-center space-x-2 text-xs">
                                            <span class="bg-gray-200 text-gray-700 px-2 py-1 rounded-full font-medium">
                                                {room.roomCode}
                                            </span>
                                        </div>
                                    </div>
                                    
                                    <div class="flex items-center justify-between text-sm text-gray-600 mb-3">
                                        <span>Host: {room.hostName}</span>
                                        <span class="bg-blue-100 text-blue-700 px-2 py-1 rounded-full text-xs font-bold">
                                            {room.language === 'nepali' ? 'नेपाली' : 'English'}
                                        </span>
                                    </div>
                                    
                                    <div class="flex items-center justify-between">
                                        <div class="flex items-center space-x-4 text-sm text-gray-600">
                                            <span class="flex items-center space-x-1">
                                                <div class="w-4 h-4">{@html UsersIcon}</div>
                                                <span>{room.playerCount}/{room.maxPlayers}</span>
                                            </span>
                                            <span class="text-xs text-gray-400">
                                                {new Date(room.createdAt).toLocaleTimeString([], {hour: '2-digit', minute:'2-digit'})}
                                            </span>
                                        </div>
                                        
                                        <button
                                            on:click={() => joinPublicRoom(room)}
                                            disabled={!playerName.trim() || selectedRoom === room || room.playerCount >= room.maxPlayers}
                                            class="bg-purple-500 hover:bg-purple-600 text-white font-bold py-2 px-4 rounded-lg text-sm transition-all disabled:opacity-50 disabled:cursor-not-allowed hover:cursor-pointer"
                                        >
                                            {selectedRoom === room ? 'Joining...' : 
                                             room.playerCount >= room.maxPlayers ? 'Full' : 'Join'}
                                        </button>
                                    </div>
                                </div>
                            {/each}
                        {/if}
                    </div>
                    
                    <div class="space-y-3">
                        <button
                            on:click={goToCreateRoom}
                            class="w-full bg-gradient-to-r from-green-500 to-teal-600 hover:from-green-600 hover:to-teal-700 text-white font-bold py-4 px-6 rounded-xl transition-all duration-200 shadow-lg hover:shadow-green-500/25 flex items-center justify-center space-x-2 hover:scale-105 transform hover:cursor-pointer"
                        >
                            <div class="w-5 h-5">{@html PlusIcon}</div>
                            <span>Create New Room</span>
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
                        
                        <!-- Room Visibility -->
                        <div>
                            <label class="block text-sm font-bold text-gray-700 mb-3">Room Type</label>
                            <div class="grid grid-cols-2 gap-2 p-1 bg-gray-100 rounded-xl">
                                <button
                                    type="button"
                                    on:click={() => isPublicRoom = true}
                                    class="flex items-center justify-center py-3 px-4 rounded-lg text-sm font-semibold transition-all duration-200 hover:cursor-pointer space-x-2
                                        {isPublicRoom ? 'bg-white shadow-lg text-gray-900 border-2 border-gray-200' : 'text-gray-600 hover:text-gray-800'}"
                                >
                                    <div class="w-4 h-4">{@html UnlockIcon}</div>
                                    <span>Public</span>
                                </button>
                                <button
                                    type="button"
                                    on:click={() => isPublicRoom = false}
                                    class="flex items-center justify-center py-3 px-4 rounded-lg text-sm font-semibold transition-all duration-200 hover:cursor-pointer space-x-2
                                        {!isPublicRoom ? 'bg-white shadow-lg text-gray-900 border-2 border-gray-200' : 'text-gray-600 hover:text-gray-800'}"
                                >
                                    <div class="w-4 h-4">{@html LockIcon}</div>
                                    <span>Private</span>
                                </button>
                            </div>
                            <p class="text-xs text-gray-500 mt-1">
                                {isPublicRoom ? 'Anyone can find and join your room' : 'Only people with the code can join'}
                            </p>
                        </div>
                        
                        <!-- Room Name (for public rooms) -->
                        {#if isPublicRoom}
                            <div>
                                <label class="block text-sm font-bold text-gray-700 mb-2">Room Name</label>
                                <input
                                    type="text"
                                    bind:value={roomName}
                                    placeholder="Give your room a name..."
                                    class="w-full p-4 border-2 border-gray-200 rounded-xl focus:border-green-500 focus:outline-none transition-all bg-gray-50 hover:bg-gray-100"
                                />
                            </div>
                        {/if}
                        
                        <!-- Max Players -->
                        <div>
                            <label class="block text-sm font-bold text-gray-700 mb-2">Max Players</label>
                            <select
                                bind:value={maxPlayers}
                                class="w-full p-4 border-2 border-gray-200 rounded-xl focus:border-green-500 focus:outline-none transition-all bg-gray-50 hover:bg-gray-100"
                            >
                                <option value={4}>4 players</option>
                                <option value={6}>6 players</option>
                                <option value={8}>8 players</option>
                                <option value={10}>10 players</option>
                                <option value={12}>12 players</option>
                            </select>
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
                        <div class="w-16 h-16 bg-gradient-to-br from-blue-500 to-cyan-600 rounded-2xl mx-auto mb-4 flex items-center justify-center shadow-xl">
                            <div class="w-8 h-8 text-white">{@html LoginIcon}</div>
                        </div>
                        <h2 class="text-2xl font-black text-gray-900 mb-2">Join by Code</h2>
                        <p class="text-gray-600 text-sm">Enter a private room code to join</p>
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
                            class="w-full bg-gradient-to-r from-blue-500 to-cyan-600 hover:from-blue-600 hover:to-cyan-700 text-white font-black py-5 px-6 rounded-xl transition-all duration-200 shadow-xl hover:shadow-blue-500/25 flex items-center justify-center space-x-3 hover:scale-105 transform hover:cursor-pointer"
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
                        <h2 class="text-3xl font-black text-gray-900 mb-2">{gameRoom?.roomName || 'Game Lobby'}</h2>
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
                        {#if gameRoom?.isPublic}
                            <div class="mt-2 inline-flex items-center text-xs text-green-600 bg-green-100 px-2 py-1 rounded-full">
                                <div class="w-3 h-3 mr-1">{@html UnlockIcon}</div>
                                Public Room
                            </div>
                        {:else}
                            <div class="mt-2 inline-flex items-center text-xs text-gray-600 bg-gray-100 px-2 py-1 rounded-full">
                                <div class="w-3 h-3 mr-1">{@html LockIcon}</div>
                                Private Room
                            </div>
                        {/if}
                    </div>
                    
                    <!-- Players List -->
                    <div class="mb-8">
                        <h3 class="text-lg font-bold text-gray-900 mb-4">Players ({totalPlayers}/{gameRoom?.maxPlayers || 8})</h3>
                        <div class="space-y-3 max-h-64 overflow-y-auto">
                            {#each Object.values(gameRoom?.players || {}) as player}
                                <div class="flex items-center justify-between p-4 bg-gray-50 rounded-xl border border-gray-100">
                                    <div class="flex items-center space-x-3">
                                        <div class="w-10 h-10 bg-gradient-to-br from-blue-500 to-purple-600 rounded-xl flex items-center justify-center shadow-lg">
                                            <span class="text-white font-bold text-sm">{player.name.charAt(0).toUpperCase()}</span>
                                        </div>
                                        <div>
                                            <div class="flex items-center space-x-2">
                                                <span class="font-bold text-gray-900">{player.name}</span>
                                                {#if player.isHost}
                                                    <div class="w-4 h-4 text-yellow-500">{@html CrownIcon}</div>
                                                {/if}
                                            </div>
                                            <div class="text-xs text-gray-500">
                                                {player.isHost ? 'Host' : 'Player'}
                                            </div>
                                        </div>
                                    </div>
                                    
                                    <div class="flex items-center space-x-2">
                                        <div class="w-3 h-3 rounded-full bg-green-400 shadow-sm"></div>
                                        <span class="text-xs text-gray-500">Online</span>
                                    </div>
                                </div>
                            {/each}
                        </div>
                    </div>
                    
                    <!-- Game Settings Display -->
                    <div class="mb-6 p-4 bg-gray-50 rounded-xl border border-gray-100">
                        <h4 class="font-bold text-gray-900 mb-3 text-sm">Game Settings</h4>
                        <div class="grid grid-cols-2 gap-3 text-sm">
                            <div class="flex items-center space-x-2">
                                <div class="w-4 h-4 text-blue-500">{@html GlobeIcon}</div>
                                <span class="text-gray-700">
                                    {gameRoom?.settings?.language === 'nepali' ? 'नेपाली' : 'English'}
                                </span>
                            </div>
                            <div class="flex items-center space-x-2">
                                <span class="text-xs">🤔</span>
                                <span class="text-gray-700">
                                    Mystery: {gameRoom?.settings?.includeDumbRole ? 'Yes' : 'No'}
                                </span>
                            </div>
                        </div>
                    </div>
                    
                    <!-- Host Controls -->
                    {#if currentPlayer?.isHost}
                        <div class="space-y-4">
                            <button
                                on:click={startGame}
                                disabled={isGeneratingWords || totalPlayers < 3}
                                class="w-full bg-gradient-to-r from-green-500 to-teal-600 hover:from-green-600 hover:to-teal-700 text-white font-black py-6 px-6 rounded-xl transition-all duration-200 shadow-xl hover:shadow-green-500/25 flex items-center justify-center space-x-3 hover:scale-105 transform disabled:opacity-50 disabled:cursor-not-allowed disabled:hover:scale-100 hover:cursor-pointer"
                            >
                                {#if isGeneratingWords}
                                    <div class="animate-spin rounded-full h-6 w-6 border-2 border-white border-t-transparent"></div>
                                    <span class="text-lg">GENERATING WORDS...</span>
                                {:else}
                                    <div class="w-6 h-6">{@html PlayIcon}</div>
                                    <span class="text-lg">START GAME</span>
                                {/if}
                            </button>
                            
                            {#if totalPlayers < 3}
                                <p class="text-center text-sm text-red-600 font-medium">
                                    Need at least 3 players to start
                                </p>
                            {/if}
                        </div>
                    {:else}
                        <div class="text-center p-6 bg-yellow-50 rounded-xl border border-yellow-200">
                            <div class="text-4xl mb-3">⏳</div>
                            <p class="text-yellow-800 font-bold">Waiting for host to start the game</p>
                            <p class="text-yellow-600 text-sm mt-1">The host will start when everyone is ready</p>
                        </div>
                    {/if}
                    
                    <!-- Leave Room Button -->
                    <div class="mt-6">
                        <button
                            on:click={resetGame}
                            class="w-full bg-gray-100 hover:bg-gray-200 text-gray-700 font-bold py-4 px-4 rounded-xl transition-all duration-200 hover:scale-105 flex items-center justify-center space-x-2 hover:cursor-pointer"
                        >
                            <div class="w-5 h-5">{@html ArrowLeftIcon}</div>
                            <span>Leave Room</span>
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
                    <!-- Game Header -->
                    <div class="text-center mb-8">
                        <div class="w-20 h-20 bg-gradient-to-br from-red-500 to-pink-600 rounded-2xl mx-auto mb-4 flex items-center justify-center shadow-xl">
                            <div class="w-10 h-10 text-white">{@html EyeIcon}</div>
                        </div>
                        <h2 class="text-3xl font-black text-gray-900 mb-2">Your Secret Word</h2>
                        <div class="bg-gray-100 px-4 py-2 rounded-xl inline-block">
                            <span class="text-sm font-bold text-gray-700">Room:</span>
                            <span class="text-lg font-black text-gray-900 ml-2">{roomCode}</span>
                        </div>
                    </div>
                    
                    <!-- Role & Word Display -->
                    <div class="mb-8">
                        <div class="text-center p-8 bg-gradient-to-br from-gray-900 to-gray-800 rounded-2xl shadow-xl">
                            {#if currentPlayer?.role === 'imposter'}
                                <div class="text-red-400 text-sm font-bold mb-2">🕵️ YOU ARE THE IMPOSTER</div>
                            {:else if currentPlayer?.role === 'dumb'}
                                <div class="text-purple-400 text-sm font-bold mb-2">🤔 YOU ARE THE MYSTERY PLAYER</div>
                            {:else}
                                <div class="text-green-400 text-sm font-bold mb-2">👥 YOU ARE A CIVILIAN</div>
                            {/if}
                            
                            <div class="text-4xl font-black text-white mb-4">
                                {currentPlayer?.word || '???'}
                            </div>
                            
                            {#if currentPlayer?.role === 'dumb'}
                                <div class="text-gray-400 text-sm">
                                    You don't have a word! Try to figure out what everyone else is talking about.
                                </div>
                            {:else if currentPlayer?.role === 'imposter'}
                                <div class="text-red-300 text-sm">
                                    Your word is different from the civilians. Blend in without being discovered!
                                </div>
                            {:else}
                                <div class="text-green-300 text-sm">
                                    Most players have your word. Find the imposter among you!
                                </div>
                            {/if}
                        </div>
                    </div>
                    
                    <!-- Game Instructions -->
                    <div class="mb-8 p-6 bg-blue-50 rounded-xl border border-blue-200">
                        <h3 class="font-bold text-blue-900 mb-3">How to Play:</h3>
                        <ol class="text-sm text-blue-800 space-y-2 list-decimal list-inside">
                            <li>Discuss your word without saying it directly</li>
                            <li>Listen carefully to what others say</li>
                            <li>Try to identify who doesn't belong</li>
                            <li>Vote out the imposter when ready!</li>
                        </ol>
                    </div>
                    
                    <!-- Player Status -->
                    <div class="mb-6">
                        <h3 class="text-lg font-bold text-gray-900 mb-4">Players ({playersRevealed}/{totalPlayers} revealed)</h3>
                        <div class="space-y-2 max-h-48 overflow-y-auto">
                            {#each Object.values(gameRoom?.players || {}) as player}
                                <div class="flex items-center justify-between p-3 bg-gray-50 rounded-lg">
                                    <div class="flex items-center space-x-3">
                                        <div class="w-8 h-8 bg-gradient-to-br from-blue-500 to-purple-600 rounded-lg flex items-center justify-center">
                                            <span class="text-white font-bold text-xs">{player.name.charAt(0).toUpperCase()}</span>
                                        </div>
                                        <div>
                                            <div class="flex items-center space-x-2">
                                                <span class="font-medium text-gray-900 text-sm">{player.name}</span>
                                                {#if player.isHost}
                                                    <div class="w-3 h-3 text-yellow-500">{@html CrownIcon}</div>
                                                {/if}
                                            </div>
                                        </div>
                                    </div>
                                    
                                    <div class="flex items-center space-x-2">
                                        {#if player.hasRevealed}
                                            <div class="w-6 h-6 text-green-500">{@html CheckIcon}</div>
                                        {:else}
                                            <div class="w-3 h-3 rounded-full bg-yellow-400"></div>
                                        {/if}
                                    </div>
                                </div>
                            {/each}
                        </div>
                    </div>
                    
                    <!-- Action Buttons -->
                    <div class="space-y-4">
                        {#if !wordRevealed}
                            <button
                                on:click={revealWord}
                                class="w-full bg-gradient-to-r from-green-500 to-teal-600 hover:from-green-600 hover:to-teal-700 text-white font-bold py-5 px-6 rounded-xl transition-all duration-200 shadow-lg hover:shadow-green-500/25 flex items-center justify-center space-x-3 hover:scale-105 transform hover:cursor-pointer"
                            >
                                <div class="w-6 h-6">{@html CheckIcon}</div>
                                <span class="text-lg">I'M READY</span>
                            </button>
                        {:else}
                            <div class="text-center p-6 bg-green-50 rounded-xl border border-green-200">
                                <div class="text-4xl mb-3">✅</div>
                                <p class="text-green-800 font-bold">You're ready!</p>
                                <p class="text-green-600 text-sm mt-1">Waiting for other players...</p>
                            </div>
                        {/if}
                        
                        {#if currentPlayer?.isHost && playersRevealed === totalPlayers}
                            <button
                                on:click={showResults}
                                class="w-full bg-gradient-to-r from-purple-500 to-indigo-600 hover:from-purple-600 hover:to-indigo-700 text-white font-bold py-5 px-6 rounded-xl transition-all duration-200 shadow-lg hover:shadow-purple-500/25 flex items-center justify-center space-x-3 hover:scale-105 transform hover:cursor-pointer"
                            >
                                <div class="w-6 h-6">{@html EyeIcon}</div>
                                <span class="text-lg">REVEAL RESULTS</span>
                            </button>
                        {/if}
                    </div>
                    
                    <!-- Leave Game Button -->
                    <div class="mt-6">
                        <button
                            on:click={resetGame}
                            class="w-full bg-gray-100 hover:bg-gray-200 text-gray-700 font-bold py-4 px-4 rounded-xl transition-all duration-200 hover:scale-105 flex items-center justify-center space-x-2 hover:cursor-pointer"
                        >
                            <div class="w-5 h-5">{@html ArrowLeftIcon}</div>
                            <span>Leave Game</span>
                        </button>
                    </div>
                </div>
            </div>
        </div>
    {/if}

    <!-- Results Screen -->
    {#if gameState === 'reveal'}
        <div class="container mx-auto px-4 py-8 md:py-16">
            <div class="max-w-md mx-auto">
                
                <div class="bg-white rounded-2xl shadow-xl border border-gray-100 p-8">
                    <!-- Results Header -->
                    <div class="text-center mb-8">
                        <div class="w-20 h-20 bg-gradient-to-br from-yellow-500 to-orange-600 rounded-2xl mx-auto mb-4 flex items-center justify-center shadow-xl">
                            <div class="w-10 h-10 text-white">{@html StarIcon}</div>
                        </div>
                        <h2 class="text-3xl font-black text-gray-900 mb-2">Game Results</h2>
                        <p class="text-gray-600">Here's how everyone did</p>
                    </div>
                    
                    <!-- Words Reveal -->
                    <div class="mb-8 space-y-4">
                        <div class="p-6 bg-green-50 rounded-xl border border-green-200">
                            <div class="text-center">
                                <div class="text-green-700 text-sm font-bold mb-2">👥 CIVILIAN WORD</div>
                                <div class="text-2xl font-black text-green-900">{gameRoom?.civilianWord}</div>
                            </div>
                        </div>
                        
                        <div class="p-6 bg-red-50 rounded-xl border border-red-200">
                            <div class="text-center">
                                <div class="text-red-700 text-sm font-bold mb-2">🕵️ IMPOSTER WORD</div>
                                <div class="text-2xl font-black text-red-900">{gameRoom?.imposterWord}</div>
                            </div>
                        </div>
                    </div>
                    
                    <!-- Players with Roles -->
                    <div class="mb-8">
                        <h3 class="text-lg font-bold text-gray-900 mb-4">Player Roles</h3>
                        <div class="space-y-3">
                            {#each Object.values(gameRoom?.players || {}) as player}
                                <div class="flex items-center justify-between p-4 rounded-xl border
                                    {player.role === 'imposter' ? 'bg-red-50 border-red-200' : 
                                     player.role === 'dumb' ? 'bg-purple-50 border-purple-200' : 
                                     'bg-green-50 border-green-200'}">
                                    <div class="flex items-center space-x-3">
                                        <div class="w-10 h-10 bg-gradient-to-br from-blue-500 to-purple-600 rounded-xl flex items-center justify-center shadow-lg">
                                            <span class="text-white font-bold text-sm">{player.name.charAt(0).toUpperCase()}</span>
                                        </div>
                                        <div>
                                            <div class="flex items-center space-x-2">
                                                <span class="font-bold text-gray-900">{player.name}</span>
                                                {#if player.isHost}
                                                    <div class="w-4 h-4 text-yellow-500">{@html CrownIcon}</div>
                                                {/if}
                                            </div>
                                            <div class="text-sm
                                                {player.role === 'imposter' ? 'text-red-700' : 
                                                 player.role === 'dumb' ? 'text-purple-700' : 
                                                 'text-green-700'}">
                                                {player.role === 'imposter' ? '🕵️ Imposter' : 
                                                 player.role === 'dumb' ? '🤔 Mystery Player' : 
                                                 '👥 Civilian'}
                                            </div>
                                        </div>
                                    </div>
                                    
                                    <div class="text-lg font-black
                                        {player.role === 'imposter' ? 'text-red-900' : 
                                         player.role === 'dumb' ? 'text-purple-900' : 
                                         'text-green-900'}">
                                        {player.word}
                                    </div>
                                </div>
                            {/each}
                        </div>
                    </div>
                    
                    <!-- Game Summary -->
                    <div class="mb-8 p-6 bg-blue-50 rounded-xl border border-blue-200">
                        <h3 class="font-bold text-blue-900 mb-3">Game Summary</h3>
                        <div class="space-y-2 text-sm text-blue-800">
                            {#if imposterPlayer}
                                <p><strong>{imposterPlayer.name}</strong> was the imposter with the word "<strong>{imposterPlayer.word}</strong>"</p>
                            {/if}
                            {#if dumbPlayer}
                                <p><strong>{dumbPlayer.name}</strong> was the mystery player with no word</p>
                            {/if}
                            <p>Civilians had the word "<strong>{gameRoom?.civilianWord}</strong>"</p>
                        </div>
                    </div>
                    
                    <!-- Action Buttons -->
                    <div class="space-y-4">
                        {#if currentPlayer?.isHost}
                            <button
                                on:click={startNewGame}
                                class="w-full bg-gradient-to-r from-green-500 to-teal-600 hover:from-green-600 hover:to-teal-700 text-white font-black py-5 px-6 rounded-xl transition-all duration-200 shadow-xl hover:shadow-green-500/25 flex items-center justify-center space-x-3 hover:scale-105 transform hover:cursor-pointer"
                            >
                                <div class="w-6 h-6">{@html RefreshIcon}</div>
                                <span class="text-lg">PLAY AGAIN</span>
                            </button>
                        {:else}
                            <div class="text-center p-6 bg-yellow-50 rounded-xl border border-yellow-200">
                                <div class="text-4xl mb-3">⏳</div>
                                <p class="text-yellow-800 font-bold">Waiting for host</p>
                                <p class="text-yellow-600 text-sm mt-1">The host can start a new game</p>
                            </div>
                        {/if}
                        
                        <button
                            on:click={resetGame}
                            class="w-full bg-gray-100 hover:bg-gray-200 text-gray-700 font-bold py-4 px-4 rounded-xl transition-all duration-200 hover:scale-105 flex items-center justify-center space-x-2 hover:cursor-pointer"
                        >
                            <div class="w-5 h-5">{@html ArrowLeftIcon}</div>
                            <span>Leave Game</span>
                        </button>
                    </div>
                </div>
            </div>
        </div>
    {/if}

</div>