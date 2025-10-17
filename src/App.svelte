<script>
    import Alternatives from './lib/tabs/Alternatives.svelte';
    import Criteria from './lib/tabs/Criteria.svelte';
    import PerbandinganKriteria from './lib/tabs/PerbandinganKriteria.svelte';
    import PerbandinganAlternative from './lib/tabs/PerbandinganAlternative.svelte';
    import Hasil from './lib/tabs/Hasil.svelte';

    const tabs = [
        'Dashboard dan Hasil',
        'Alternatives',
        'Kriteria',
        'Perbandingan Antar Kriteria',
        'Perbandingan Antar Alternatif',
    ];
    let activeTab = tabs[0];
    function changeTab(t){
        activeTab = t;
    }
    // app meta
    let nama = 'SPPK Pemilihan Framework Frontend';
    let deskripsi = 'Pembantu memilih framework frontend yang paling cocok untuk proyek dengan metode AHP dan transparasi perhitungan';

    // shared state across tabs
    let alternatives = [
        'Vue.js',
        'React',
        'Svelte'
    ];

    let criteria = {
        C1: { name: 'Kurva Belajar',            type: 'benefit' },
        C2: { name: 'Relevansi Industri',       type: 'benefit' },
        C3: { name: 'Ekosistem & Komunitas',    type: 'benefit' },
        C4: { name: 'Struktur & Kerapian Kode', type: 'benefit' },
        C5: { name: 'Performa Aplikasi',        type: 'benefit' }
    };

    let scores = {
    };
    alternatives.forEach(a => {
        scores[a] = {};
        Object.keys(criteria).forEach(c => {
            scores[a][c] = 0;
        });
    });
    
    // AHP pairwise matrix for criteria (optional persistence)
    let criteriaMatrix = null;
    let criteriaWeights = null; // { [criterionId]: number }
    // AHP pairwise matrices of alternatives per criterion (optional persistence)
    let altMatrices = null; // { [criterionId]: { [altA]: { [altB]: number } } }

    // load from local storage if available
    if (typeof localStorage !== 'undefined') {
        const saved = localStorage.getItem('sppk-data');
        if (saved) {
            try {
                const data = JSON.parse(saved);
                if (Array.isArray(data.alternatives)) alternatives = data.alternatives;
                if (data.criteria && typeof data.criteria === 'object') criteria = data.criteria;
                if (data.scores && typeof data.scores === 'object') scores = data.scores;
                if (data.criteriaMatrix && typeof data.criteriaMatrix === 'object') criteriaMatrix = data.criteriaMatrix;
                if (data.criteriaWeights && typeof data.criteriaWeights === 'object') criteriaWeights = data.criteriaWeights;
                if (data.altMatrices && typeof data.altMatrices === 'object') altMatrices = data.altMatrices;
                if (typeof data.nama === 'string') nama = data.nama;
                if (typeof data.deskripsi === 'string') deskripsi = data.deskripsi;
            } catch (e) {
                console.warn('Gagal memuat data dari local storage', e);
            }
        }
    }

    function saveLocal() {
        const data = { alternatives, criteria, scores, criteriaMatrix, criteriaWeights, altMatrices, nama, deskripsi };
        localStorage.setItem('sppk-data', JSON.stringify(data));
        console.log('Data disimpan ke local storage');
    }

    function updateAlternatives(e) {
        alternatives = e.detail;
        saveLocal();
    }

    function updateCriteria(e) {
        criteria = e.detail;
        saveLocal();
    }

    function onCriteriaMatrixChange(e) {
        criteriaMatrix = e.detail;
        saveLocal();
    }

    function rebuildScores() {
        const next = {};
        alternatives.forEach(a => {
            next[a] = {};
            Object.keys(criteria).forEach(c => { next[a][c] = 0; });
        });
        scores = next;
    }

</script>

<div class="layout inclusive-sans">

    <nav class="sidebar bgdark">
        <h1>{nama || 'SPPK'}</h1>
        <ul>
            {#each tabs as t}
                <li class:selected={activeTab === t}>
                    <button on:click={
                            e => changeTab(t)
                    }>{t}</button>
                </li>
            {/each}
        </ul>
    </nav>
    <main>
        {#if activeTab === 'Alternatives'}
            <Alternatives {alternatives} scores={structuredClone(scores)} on:ganti={updateAlternatives} />
        {:else if activeTab === 'Kriteria'}
            <Criteria {criteria}
                on:criteriaChange={updateCriteria}
            />
        {:else if activeTab === 'Perbandingan Antar Kriteria'}
            <PerbandinganKriteria
                {criteria}
                initialMatrix={criteriaMatrix}
                on:matrixChange={onCriteriaMatrixChange}
                on:weightsChange={(e) => {
                    criteriaWeights = e.detail;
                    saveLocal();
                }}
            />
        {:else if activeTab === 'Perbandingan Antar Alternatif'}
            <PerbandinganAlternative
                {alternatives}
                {criteria}
                scores={structuredClone(scores)}
                initialMatrices={altMatrices}
                on:scoresChange={(e) => {
                    scores = e.detail;
                    saveLocal();
                }}
                on:matricesChange={(e) => {
                    altMatrices = e.detail;
                    saveLocal();
                }}
            />
        {:else if activeTab === 'Dashboard dan Hasil'}
            <Hasil
                {alternatives}
                {criteria}
                {scores}
                criteriaWeights={criteriaWeights || {}}
                {nama}
                {deskripsi}
                on:importConfig={(e)=>{
                    const cfg = e.detail;
                    // normalize alternatives
                    if (Array.isArray(cfg.alternatives)) {
                        alternatives = cfg.alternatives.filter(x => typeof x === 'string' && x.trim().length > 0);
                    }
                    // normalize criteria list to object with codes C1, C2, ...
                    if (Array.isArray(cfg.criteria)) {
                        const list = cfg.criteria.filter(x => typeof x === 'string' && x.trim().length > 0);
                        const obj = {};
                        list.forEach((name, idx) => {
                            const code = `C${idx+1}`;
                            obj[code] = { name, type: 'benefit' };
                        });
                        criteria = obj;
                    }
                    if (typeof cfg.nama === 'string') nama = cfg.nama;
                    if (typeof cfg.deskripsi === 'string') deskripsi = cfg.deskripsi;
                    // reset derived structures
                    criteriaMatrix = null;
                    criteriaWeights = null;
                    altMatrices = null;
                    rebuildScores();
                    saveLocal();
                }}
            />
        {/if}
    </main>
</div>

<style lang="scss">
@use './lib/css/_color.scss' as color;
@use './lib/css/_reusable.scss' as *;

body {
    font-size: 18px;
}
.sidebar {
    background: linear-gradient(180deg, color.$gray-darker 0%, color.$gray-dark 100%);
    padding: 24px;
    box-shadow: 2px 0 10px rgba(0, 0, 0, 0.1);
    
    h1 {
        font-size: 1.4em;
        font-weight: 700;
        margin: 0 0 32px;
        color: white;
        text-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
        padding-bottom: 16px;
        border-bottom: 1px solid rgba(255, 255, 255, 0.1);
    }
    
    ul {
        list-style: none;
        padding: 0;
        margin: 0;
    }
    
    button {
        font-size: 15px;
        width: 100%;
        text-align: left;
        padding: 16px 20px;
        background: transparent;
        border: none;
        border-radius: 8px;
        cursor: pointer;
        color: rgba(255, 255, 255, 0.8);
        font-weight: 500;
        transition: all 0.2s ease;
        position: relative;
        
        &:hover {
            background: rgba(255, 255, 255, 0.1);
            color: white;
            transform: translateX(4px);
        }
        
        &:active {
            transform: translateX(2px);
        }
    }
    
    li {
        margin: 4px 0;
        
        &.selected button {
            background: linear-gradient(135deg, color.$primary, color.$primary-dark);
            color: white;
            box-shadow: 0 4px 12px rgba(59, 130, 246, 0.3);
            transform: translateX(4px);
            
            &::before {
                content: '';
                position: absolute;
                left: -24px;
                top: 50%;
                transform: translateY(-50%);
                width: 4px;
                height: 100%;
                background: color.$primary;
                border-radius: 0 2px 2px 0;
            }
        }
    }
    
    @media screen and (max-width: 600px) {
        padding: 16px;
        
        h1 {
            font-size: 1.2em;
            margin-bottom: 20px;
        }
        
        button {
            font-size: 14px;
            padding: 12px 16px;
        }
    }
}

.layout {
    display: grid;
    grid-template-columns: 280px 1fr;
    min-height: 100vh;
    
    @media screen and (max-width: 768px) {
        grid-template-columns: 1fr;
        
        .sidebar {
            display: none; // Hide sidebar on mobile, could implement mobile menu later
        }
    }
}

main {
    background: linear-gradient(135deg, #f8fafc 0%, #e2e8f0 100%);
    position: relative;
    overflow-x: auto;
}

</style>