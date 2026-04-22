<script>
    import { onMount } from 'svelte';

    // --- CONFIG ---
    const API_BASE = "https://berlin-walls-banner-bloomberg.trycloudflare.com";
    const DISTANCE_STREAM_ID = "03c97b78-0167-446e-8bd0-cbc941561b65";
    const MATRIX_STREAM_ID = "e4904706-0687-4da7-9b05-9cbec06d689c";
    
    const ROBOT_IDS = {
        distance: "a9f145b1-08f3-4e3a-9ff6-09cd6b2b3d46",
        matrix:   "f1e0c08b-77c5-4e50-a592-dd7732f0975b",
        red:      "c7ee3c42-c1d2-4e34-9f89-653f1398da61",
        green:    "06c7cfa4-6007-4458-999d-5d46d97623b5",
        blue:     "30cd3dbc-75f9-4216-aba5-1cc9d85a2d99"
    };

    // --- STATE ---
    let distanceOutput = $state('0.0 cm');
    let sliderRed   = $state(0);
    let sliderGreen = $state(0);
    let sliderBlue  = $state(0);
    let toggledButtons = $state(new Set());
    let lastUserAction = 0;

    // --- HJÄLPFUNKTIONER ---


    
    function debounce(fn, delay) {
        let timeout;
        return (...args) => {
            clearTimeout(timeout);
            timeout = setTimeout(() => fn(...args), delay);
        };
    }

    const sendSlider = debounce(async (id, val) => {
        await fetch(`${API_BASE}/robots/${id}/command`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ command: String(val) })
        });
    }, 40);

    async function sendMatrixCommand(val) {
        lastUserAction = Date.now();
        await fetch(`${API_BASE}/robots/${ROBOT_IDS.matrix}/command`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ command: String(val) })
        });
    }

    // --- DATA FETCHING ---

    async function updateAllData() {
        try {
            // Hämta distans
            const dRes = await fetch(`${API_BASE}/streams/data/latest/${DISTANCE_STREAM_ID}/1`);
            if (dRes.ok) {
                const dData = await dRes.json();
                if (dData?.[0]) distanceOutput = `${dData[0].number_value.toFixed(1)} cm`;
            }

            // Synka matrix — bara om användaren inte klickat nyss
            if (Date.now() - lastUserAction > 50) {
                const mRes = await fetch(`${API_BASE}/streams/data/latest/${MATRIX_STREAM_ID}/1`);
                if (mRes.ok) {
                    const mData = await mRes.json();
                    if (mData?.[0]?.string_value) {
                        const str = mData[0].string_value; // "0111001100101..."
                        let newSet = new Set();
                        for (let i = 0; i < str.length; i++) {
                            if (str[i] === "1") newSet.add(i);
                        }
                        toggledButtons = newSet;
                    }
                }
            }
        } catch (e) { /* tyst fel */ }
    }

    // --- HANDLERS ---

    function handlePixelClick(index) {
        const row = Math.floor(index / 8);
        const col = index % 8;

        if (toggledButtons.has(index)) toggledButtons.delete(index);
        else toggledButtons.add(index);
        toggledButtons = toggledButtons; // trigga reaktivitet

        sendMatrixCommand(`${row},${col}`);
    }

    async function clearMatrix() {
        toggledButtons = new Set();
        await sendMatrixCommand("clear");
    }

    // --- EFFECTS ---
    $effect(() => { sendSlider(ROBOT_IDS.red,   sliderRed);   });
    $effect(() => { sendSlider(ROBOT_IDS.green, sliderGreen); });
    $effect(() => { sendSlider(ROBOT_IDS.blue,  sliderBlue);  });

    onMount(() => {
        const interval = setInterval(updateAllData, 200);
        return () => clearInterval(interval);
    });
</script>

<div class="app">
    <header>
        <h1>Robot Controller</h1>
        <p class="ip">Backend: {API_BASE}</p>
    </header>

    <div class="grid-main">
        <section class="card">
            <h2>RGB Lighting</h2>
            <div class="control">
                <label for="red">Röd: {sliderRed}</label>
                <input type="range" id="red" min="0" max="255" bind:value={sliderRed} class="r-slider" />
            </div>
            <div class="control">
                <label for="green">Grön: {sliderGreen}</label>
                <input type="range" id="green" min="0" max="255" bind:value={sliderGreen} class="g-slider" />
            </div>
            <div class="control">
                <label for="blue">Blå: {sliderBlue}</label>
                <input type="range" id="blue" min="0" max="255" bind:value={sliderBlue} class="b-slider" />
            </div>
        </section>

        <section class="card sensor">
            <h2>Distance Sensor</h2>
            <div class="dist-val">{distanceOutput}</div>
        </section>

        <section class="card matrix-section">
            <h2>8x8 LED Matrix</h2>
            <div class="led-grid">
                {#each Array.from({ length: 64 }) as _, i}
                    <button
                        class="led {toggledButtons.has(i) ? 'active' : ''}"
                        onclick={() => handlePixelClick(i)}>
                    </button>
                {/each}
            </div>
            <button class="btn-clear" onclick={clearMatrix}>Clear Matrix</button>
        </section>
    </div>
</div>

<style>
    :global(body) {
        background: #0b0e14;
        color: #fff;
        font-family: 'Inter', system-ui, sans-serif;
        margin: 0;
    }

    .app { max-width: 1100px; margin: 0 auto; padding: 2rem; }

    header { text-align: center; margin-bottom: 2rem; }
    .ip { color: #555; font-size: 0.8rem; }

    .grid-main {
        display: grid;
        grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
        gap: 1.5rem;
    }

    .card {
        background: #151921;
        padding: 1.5rem;
        border-radius: 12px;
        border: 1px solid #222;
    }

    h2 { font-size: 1rem; color: #888; margin-top: 0; text-transform: uppercase; letter-spacing: 1px; }

    .control { margin-bottom: 1rem; }
    label { display: block; margin-bottom: 0.5rem; font-size: 0.9rem; }

    input[type="range"] { width: 100%; height: 6px; border-radius: 5px; appearance: none; background: #333; }
    .r-slider { accent-color: #ff4d4d; }
    .g-slider { accent-color: #4dff88; }
    .b-slider { accent-color: #4d94ff; }

    .dist-val {
        font-size: 3.5rem;
        font-weight: 800;
        text-align: center;
        color: #00ffcc;
        margin: 1rem 0;
    }

    .led-grid {
        display: grid;
        grid-template-columns: repeat(8, 1fr);
        gap: 4px;
        margin-bottom: 1rem;
    }

    .led {
        aspect-ratio: 1;
        background: #222;
        border: none;
        border-radius: 2px;
        cursor: pointer;
        transition: 0.1s;
    }

    .led.active {
        background: #ffaa00;
        box-shadow: 0 0 10px #ffaa00;
    }

    .btn-clear {
        width: 100%;
        padding: 0.75rem;
        background: #222;
        border: 1px solid #444;
        color: #ccc;
        cursor: pointer;
        border-radius: 6px;
    }

    .btn-clear:hover { background: #333; color: white; }
</style>
