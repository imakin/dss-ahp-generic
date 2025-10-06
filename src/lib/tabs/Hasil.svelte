<script>
    import { createEventDispatcher } from 'svelte';
    const dispatch = createEventDispatcher();
    export let alternatives = [];
    export let criteria = {};
    export let scores = {};
    export let criteriaWeights = {};
    export let nama = 'SPPK';
    export let deskripsi = '';

    $: cids = Object.keys(criteria);

    function computeFinalScores() {
        const result = [];
        const cw = criteriaWeights || {};
        for (const alt of alternatives) {
            const perCrit = scores?.[alt] || {};
            let total = 0;
            for (const cid of cids) {
                const wCrit = Number(cw[cid] ?? 0);
                const wAlt = Number(perCrit[cid] ?? 0);
                total += wCrit * wAlt;
            }
            result.push({ alt, total });
        }
        const sum = result.reduce((a, b) => a + b.total, 0);
        if (sum > 0) {
            for (const r of result) r.norm = r.total / sum;
        } else {
            for (const r of result) r.norm = 0;
        }
        result.sort((a, b) => b.total - a.total);
        return result;
    }

    $: ranking = computeFinalScores();

    // YAML import/export (minimal for our schema)
    function toYAML(obj) {
        const lines = [];
        const esc = (s) => String(s).replace(/"/g, '\\"');
        lines.push(`nama: "${esc(obj.nama || '')}"`);
        lines.push(`deskripsi: "${esc(obj.deskripsi || '')}"`);
        lines.push('alternatives:');
        (obj.alternatives || []).forEach(a => lines.push(`  - "${esc(a)}"`));
        lines.push('criteria:');
        (obj.criteria || []).forEach(c => lines.push(`  - "${esc(c)}"`));
        return lines.join('\n');
    }
    function toYAMLSpecific(){
        const cfg = {
            nama,
            deskripsi,
            alternatives,
            criteria: Object.values(criteria).map(c => c.name)
        };
        const text = toYAML(cfg);
        return text;
    }

    function fromYAML(text) {
        const lines = String(text || '').split(/\r?\n/);
        const out = { nama: '', deskripsi: '', alternatives: [], criteria: [] };
        let mode = null;
        const stripQuotes = (s) => s.replace(/^["']|["']$/g, '');
        for (const raw of lines) {
            const line = raw.replace(/^\uFEFF/, '').trimEnd(); // strip BOM
            if (!line.trim()) continue;
            if (/^nama\s*:/i.test(line)) { mode = null; out.nama = stripQuotes(line.replace(/^nama\s*:\s*/i, '').trim()); continue; }
            if (/^deskripsi\s*:/i.test(line)) { mode = null; out.deskripsi = stripQuotes(line.replace(/^deskripsi\s*:\s*/i, '').trim()); continue; }
            if (/^alternatives\s*:/i.test(line)) { mode = 'alternatives'; continue; }
            if (/^criteria\s*:/i.test(line)) { mode = 'criteria'; continue; }
            const m = line.match(/^\s*\-\s*(.+)$/);
            if (m && mode) {
                const val = stripQuotes(m[1].trim());
                if (val) out[mode].push(val);
            }
        }
        return out;
    }

    function exportYAML() {
        const text = toYAMLSpecific();
        const blob = new Blob([text], { type: 'text/yaml;charset=utf-8' });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = 'sppk-config.yaml';
        a.click();
        URL.revokeObjectURL(url);
    }

    let selectedFileName = '';
    let importMessage = '';

    function onImportYAML(e) {
        const input = e.currentTarget;
        const file = input?.files?.[0];
        if (!file) return;
        selectedFileName = file.name;
        console.log('Memulai import YAML:', selectedFileName);
        const reader = new FileReader();
        reader.onload = () => {
            try {
                const text = String(reader.result || '');
                const cfg = fromYAML(text);
                // Emit Svelte event so parent (App.svelte) receives on:importConfig
                dispatch('importConfig', cfg);
                console.log('Konfigurasi YAML diimport', cfg);
                importMessage = 'Import berhasil.';
            } catch (err) {
                console.error('Gagal parse YAML:', err);
                alert('Gagal import YAML. Pastikan format sesuai.');
                importMessage = 'Import gagal.';
            }
        };
        reader.readAsText(file);
    }
</script>
<div class="inclusive-sans">
    <h1 class="title">{nama}</h1>
    {#if deskripsi}
        <p class="desc">{deskripsi}</p>
    {/if}

    <div class="flex">
        <div class="percent50">
            <div class="weights">
                <h2>Bobot Kriteria</h2>
                <table class="textleft">
                    <thead >
                        <tr><th>Kriteria</th><th>Bobot</th></tr>
                    </thead>
                    <tbody>
                    {#each cids as id}
                        <tr>
                            <td><strong>{criteria[id]?.name || id}:</strong></td>
                            <td>{Number(criteriaWeights[id] ?? 0).toFixed(4)}</td>
                        </tr>
                    {/each}
                    </tbody>
                </table>
            </div>
            <div class="card hasil">
                <h2>Matrix Prioritas</h2>
                <table class="table">
                    <thead>
                        <tr>
                            <th>Alternatif</th>
                            {#each cids as k}
                                <th>{criteria[k]?.name || k}</th>
                            {/each}
                        </tr>
                    </thead>
                    <tbody>
                        {#each alternatives as alt}
                            <tr>
                                <td>{alt}</td>
                                {#each cids as k}
                                    <td>{Number(scores?.[alt]?.[k] ?? 0).toFixed(3)}</td>
                                {/each}
                            </tr>
                        {/each}
                    </tbody>
                </table>
            </div>
            <div class="card hasil">
                <h2>Hasil Akhir (Ranking AHP)</h2>
                {#if !alternatives?.length || !Object.keys(criteria)?.length}
                    <p>Lengkapi data alternatif dan kriteria.</p>
                {:else}
                    <table class="table">
                        <thead>
                            <tr>
                                <th>Peringkat</th>
                                <th>Alternatif</th>
                                <th>Skor</th>
                                <th>Skor (dinormalisasi)</th>
                            </tr>
                        </thead>
                        <tbody>
                            {#each ranking as row, i}
                                <tr>
                                    <td>{i + 1}</td>
                                    <td>{row.alt}</td>
                                    <td>{row.total.toFixed(6)}</td>
                                    <td>{row.norm.toFixed(6)}</td>
                                </tr>
                            {/each}
                        </tbody>
                    </table>
                {/if}
            </div>
        </div>

        <div class="percent50">
            <div class="controls marginvpx10">
                <button type="button" class="lineheight3" on:click={exportYAML}>Export YAML</button>
                <label class="lineheight3">
                    Import YAML
                    <input type="file" accept=".yaml,.yml,text/yaml" on:change={onImportYAML} />
                </label>
                {#if selectedFileName}
                    <span class="filename">{selectedFileName}</span>
                {/if}
                {#if importMessage}
                    <span class="import-msg">{importMessage}</span>
                {/if}
            </div>
            <div class="texteditor">
                <h2>Config file</h2>
                <textarea rows="20" class="fullwidth">
                    {toYAMLSpecific()}
                </textarea>
            </div>
        </div>
    </div>
</div>

<style lang="scss">
@use '../css/_color.scss' as color;
@use '../css/_reusable.scss' as *;

table {
    width: 100%;
    border-collapse: collapse;
    th, td { border-bottom: 4px solid color.$gray; padding: .5em; text-align: left; }
    thead th { background: color.$gray; }
    border: solid 2px gray;
    padding: 1em;
}

.texteditor{
    textarea {
        padding: 2em;
        background-color: rgb(48, 18, 1);
        color: white;
        line-height: 2em;
    }
}

</style>