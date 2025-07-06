<script setup>
import { ref, onMounted, computed } from 'vue';
import Layout from './components/layouts/Layout.vue';
const showLogin = ref(true);
const username = ref(null);
const profilePicture = ref(null);
const spotifyId = ref(null);
const likedTracks = ref([]);
const userPlaylists = ref([]);
const foundBPMs = ref([]);
const notFoundBPMs = ref([]);
const selectedTempo = ref(120);
const loadingComplete = ref(false);
const playlistSuccess = ref(false);
const createdPlaylistUrl = ref('');
const filteredSongs = computed(() =>
    foundBPMs.value.filter(song =>
        Math.abs(song.bpm - selectedTempo.value) <= 5
    )
);
const accessToken = localStorage.getItem('access_token');
if (accessToken && !isTokenExpired()) {
    showLogin.value = false;
} else {
    showLogin.value = true;
    localStorage.removeItem('access_token');
    localStorage.removeItem('token_timestamp');
}
function generateRandomString(length) {
    const possible = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
    const values = crypto.getRandomValues(new Uint8Array(length));
    return values.reduce((acc, x) => acc + possible[x % possible.length], "");
}
async function sha256(plain) {
    const encoder = new TextEncoder();
    const data = encoder.encode(plain);
    return window.crypto.subtle.digest('SHA-256', data);
}
function base64encode(input) {
    return btoa(String.fromCharCode(...new Uint8Array(input)))
        .replace(/=/g, '')
        .replace(/\+/g, '-')
        .replace(/\//g, '_');
}
function isTokenExpired() {
    const timestamp = localStorage.getItem('token_timestamp');
    if (!timestamp) return true;
    const age = Date.now() - Number(timestamp);
    return age > 3600 * 1000;
}
function logout() {
    localStorage.removeItem('access_token');
    localStorage.removeItem('token_timestamp');
    localStorage.removeItem('code_verifier');
    location.reload();
}
async function fetchUsername() {
    const token = localStorage.getItem('access_token');
    if (!token) return;
    const res = await fetch('https://api.spotify.com/v1/me', {
        headers: {
            Authorization: `Bearer ${token}`
        }
    });
    if (res.ok) {
        const data = await res.json();
        username.value = data.display_name || data.id;
        profilePicture.value = data.images?.[0]?.url || null;
        spotifyId.value = data.id;
    }
}
async function fetchLikedSongs() {
    const token = localStorage.getItem('access_token');
    const headers = { Authorization: `Bearer ${token}` };
    let url = 'https://api.spotify.com/v1/me/tracks?limit=50';
    while (url) {
        const res = await fetch(url, { headers });
        const data = await res.json();
        likedTracks.value.push(...data.items.map(item => item.track));
        url = data.next;
    }
}
async function fetchUserPlaylistsAndTracks() {
    const token = localStorage.getItem('access_token');
    const headers = { Authorization: `Bearer ${token}` };
    let url = 'https://api.spotify.com/v1/me/playlists?limit=50';
    const playlists = [];
    while (url) {
        const res = await fetch(url, { headers });
        const data = await res.json();
        playlists.push(...data.items);
        url = data.next;
    }
    const playlistsWithTracks = [];
    for (const playlist of playlists) {
        let tracksUrl = `https://api.spotify.com/v1/playlists/${playlist.id}/tracks?limit=50`;
        const tracks = [];
        while (tracksUrl) {
            const res = await fetch(tracksUrl, { headers });
            const data = await res.json();
            tracks.push(...data.items.map(item => item.track));
            tracksUrl = data.next;
        }
        playlistsWithTracks.push({
            name: playlist.name,
            tracks: tracks.filter(Boolean)
        });
    }
    userPlaylists.value = playlistsWithTracks;
}
async function fetchBPMFromFormattedSongBPM() {
    const slugify = str =>
        str
            .toString()
            .toLowerCase()
            .replace(/&/g, 'and')
            .replace(/[^\w\s-]/g, '')
            .replace(/\s+/g, '-')
            .replace(/-+/g, '-')
            .trim();
    const allTracks = [
        ...likedTracks.value,
        ...userPlaylists.value.flatMap(p => p.tracks)
    ];
    const uniqueTracks = Array.from(new Map(
        allTracks.filter(t => t?.id).map(t => [t.id, t])
    ).values());
    foundBPMs.value = [];
    notFoundBPMs.value = [];
    for (const track of uniqueTracks) {
        const title = track?.name;
        const artist = track?.artists?.[0]?.name;
        if (!title || !artist) {
            console.log(`Missing title or artist: ${title} - ${artist}.`);
            continue;
        }
        console.log(`Searching BPM for: ${title} by ${artist}.`);
        const artistSlug = slugify(artist);
        const titleSlug = slugify(title);
        const proxyUrl = `https://api.codetabs.com/v1/proxy?quest=https://songbpm.com/@${artistSlug}/${titleSlug}`;
        try {
            const res = await fetch(proxyUrl);
            const html = await res.text();
            const bpmMatch = html.match(/(\d+)\s*BPM/i);
            if (bpmMatch) {
                const bpm = parseInt(bpmMatch[1]);
                console.log(`${title} by ${artist}: ${bpm} BPM.`);
                foundBPMs.value.push({ title, artist, bpm });
            } else {
                console.warn(`No BPM found for ${title}.`);
                notFoundBPMs.value.push(`${title} by ${artist}`);
            }
        } catch (err) {
            console.error(`Error fetching BPM for ${title} by ${artist}.`, err);
            notFoundBPMs.value.push(`${title} by ${artist}`);
        }
    }
    console.log("Found BPMs: ");
    console.table(foundBPMs);
    console.log("Not Found BPMs: ");
    console.table(notFoundBPMs);
    loadingComplete.value = true;
}
async function createTempoPlaylist() {
    const token = localStorage.getItem('access_token');
    if (!spotifyId.value || !token) return;
    const createRes = await fetch(`https://api.spotify.com/v1/users/${spotifyId.value}/playlists`, {
        method: 'POST',
        headers: {
            Authorization: `Bearer ${token}`,
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            name: `Tempo ${selectedTempo.value} Playlist`,
            description: `Songs with BPM near ${selectedTempo.value}`,
            public: false
        })
    });
    const playlistData = await createRes.json();
    if (!playlistData.id) {
        console.log("Failed to create the playlist.");
        return;
    }
    const allTracks = [
        ...likedTracks.value,
        ...userPlaylists.value.flatMap(p => p.tracks)
    ];
    const trackUris = filteredSongs.value.map(filtered => {
        const match = allTracks.find(t =>
            t?.name === filtered.title && t?.artists?.[0]?.name === filtered.artist
        );
        return match?.uri;
    }).filter(Boolean);
    for (let i = 0; i < trackUris.length; i += 100) {
        const chunk = trackUris.slice(i, i + 100);
        await fetch(`https://api.spotify.com/v1/playlists/${playlistData.id}/tracks`, {
            method: 'POST',
            headers: {
                Authorization: `Bearer ${token}`,
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({ uris: chunk })
        });
    }
    createdPlaylistUrl.value = playlistData.external_urls.spotify;
    playlistSuccess.value = true;
    setTimeout(() => {
        playlistSuccess.value = false;
        createdPlaylistUrl.value = '';
    }, 30000);
    console.log("Playlist is created.");
}
const clientId = '1be48ee2258e4e1fa3bc0bd385194313';
const redirectUri = 'http://127.0.0.1:5173/';
const scope = 'user-read-private user-read-email user-library-read playlist-read-private playlist-modify-private';
const authUrl = new URL("https://accounts.spotify.com/authorize");
async function login() {
    const codeVerifier = generateRandomString(64);
    const hashed = await sha256(codeVerifier)
    const codeChallenge = base64encode(hashed);
    const params = {
        response_type: 'code',
        client_id: clientId,
        scope,
        code_challenge_method: 'S256',
        code_challenge: codeChallenge,
        redirect_uri: redirectUri
    }
    window.localStorage.setItem('code_verifier', codeVerifier);
    authUrl.search = new URLSearchParams(params).toString();
    window.location.href = authUrl.toString();
}
onMounted(async () => {
    const urlParams = new URLSearchParams(window.location.search);
    const code = urlParams.get('code');
    if (code) {
        window.history.replaceState({}, document.title, '/');
        const codeVerifier = localStorage.getItem('code_verifier');
        if (!codeVerifier) {
            return;
        }
        const body = new URLSearchParams({
            client_id: clientId,
            grant_type: 'authorization_code',
            code: code,
            redirect_uri: redirectUri,
            code_verifier: codeVerifier
        });
        const res = await fetch('https://accounts.spotify.com/api/token', {
            method: 'POST',
            headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
            body: body.toString()
        });
        const data = await res.json();
        if (data.access_token) {
            localStorage.setItem('access_token', data.access_token);
            localStorage.setItem('token_timestamp', Date.now().toString());
            showLogin.value = false;
            await fetchUsername();
            await fetchLikedSongs();
            await fetchUserPlaylistsAndTracks();
            await fetchBPMFromFormattedSongBPM();
        }
    } else if (accessToken && !isTokenExpired()) {
        showLogin.value = false;
        await fetchUsername();
        await fetchLikedSongs();
        await fetchUserPlaylistsAndTracks();
        await fetchBPMFromFormattedSongBPM();
    }
});
</script>

<template>
    <Layout>
        <section class="app">
            <div v-if="showLogin" class="card login-card">
                <h2>Songs Classifier</h2>
                <h5>Organize your music by how it moves you.</h5>
                <button @click="login">Login with Spotify</button>
            </div>
            <div v-else class="cards-container">
                <div class="left-column">
                    <div class="card profile-card">
                        <h2>Profile</h2>
                        <div class="profile-info">
                            <img v-if="profilePicture" :src="profilePicture" alt="Profile Picture" />
                            <span>{{ username }}</span>
                        </div>
                        <h4>ID: {{ spotifyId }}</h4>
                        <button @click="logout">Logout</button>
                    </div>
                    <div v-if="!loadingComplete" class="card profile-card">
                        <h2>Loading BPM data.</h2>
                        <p>Please wait while we scan your songs.</p>
                    </div>
                    <div class="card profile-card">
                        <h2>Tempo Setting</h2>
                        <label for="tempoRange">Select Tempo: {{ selectedTempo }}</label>
                        <input type="range" id="tempoRange" min="60" max="200" step="1" v-model="selectedTempo" />
                    </div>
                    <div class="card music-card">
                        <h2>Playlist with songs at {{ selectedTempo }} BPM ({{ filteredSongs.length }})</h2>
                        <ul>
                            <li v-for="song in filteredSongs" :key="song.title + song.artist">
                                {{ song.title }} by {{ song.artist }} - {{ song.bpm }} BPM
                            </li>
                        </ul>
                        <button @click="createTempoPlaylist" :disabled="!loadingComplete || filteredSongs.length === 0">
                            Create a playlist!
                        </button>
                        <div v-if="playlistSuccess" class="success-message">
                            <p>Open playlist in</p>
                            <a :href="createdPlaylistUrl" target="_blank" rel="noopener noreferrer">Spotify</a>
                        </div>
                    </div>
                    <div class="card music-card">
                        <h2>Songs with no known BPM ({{ notFoundBPMs.length }})</h2>
                        <ul>
                            <li v-for="song in notFoundBPMs" :key="song">
                                {{ song }}
                            </li>
                        </ul>
                    </div>
                </div>
                <div class="right-column">
                    <div class="card music-card">
                        <h2>Liked Songs</h2>
                        <div v-if="likedTracks.length">
                            <ul>
                                <li v-for="song in likedTracks" :key="song.id">
                                    {{ song.name }} - {{song.artists.map(a => a.name).join(', ')}}
                                </li>
                            </ul>
                        </div>
                        <p v-else>No liked songs found.</p>
                    </div>
                    <div v-if="userPlaylists.length === 0" class="card music-card">
                        <h2>Playlists</h2>
                        <p>No playlists found.</p>
                    </div>
                    <div v-else class="playlists-wrapper">
                        <div v-for="playlist in userPlaylists" :key="playlist.id" class="card music-card">
                            <h2>Playlist: {{ playlist.name }}</h2>
                            <ul>
                                <li v-for="track in playlist.tracks" :key="track.id">
                                    {{ track.name }} - {{track.artists.map(a => a.name).join(', ')}}
                                </li>
                            </ul>
                        </div>
                    </div>
                </div>
            </div>
        </section>
    </Layout>
</template>

<style scope>
.app {
    padding: 3rem 1rem;
    padding-top: 0px;
    padding-bottom: 0px;
}

.login-card,
.left-column,
.right-column,
.playlists-wrapper,
.profile-card,
.success-message {
    display: flex;
    flex-direction: column;
}

.login-card {
    gap: 0.5rem;
    width: 30%;
    margin: 0 auto;
}

.cards-container {
    display: flex;
    gap: 2rem;
    justify-content: center;
}

.left-column,
.right-column {
    gap: 1.5rem;
    flex: 1;
}

.playlists-wrapper,
.profile-card {
    gap: 1.5rem;
}

.profile-card {
    gap: 0.75rem;
}

.profile-info {
    display: flex;
    align-items: center;
    gap: 0.5rem;
}

.profile-info img {
    width: 2.5rem;
    height: 2.5rem;
    border-radius: 50%;
    aspect-ratio: 1 / 1;
    transition: transform 0.2s ease;
}

.profile-info img:hover {
    transform: scale(1.1);
}

.profile-info span {
    font-size: 1.2rem;
    font-weight: 600;
}

.music-card ul {
    padding-left: 1.25rem;
    list-style: disc;
    line-height: 1.5;
    margin-top: 0.5rem;
}

.music-card button {
    margin-top: 0.5rem;
}

.music-card h2,
.music-card h3 {
    margin-bottom: 0.3rem;
}

@media (max-width: 900px) {
    .cards-container {
        flex-direction: column;
    }
}

.success-message {
    margin-top: 0.3rem;
    flex-direction: row;
    gap: 0.25rem;
}
</style>