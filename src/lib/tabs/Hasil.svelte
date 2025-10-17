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
<div class="inclusive-sans hasil-container">
    <div class="page-header">
        <h1 class="page-title">{nama}</h1>
        {#if deskripsi}
            <p class="page-description">{deskripsi}</p>
        {/if}
    </div>

    <div class="results-grid">
        <div class="results-main">
            <!-- Criteria Weights Card -->
            <div class="card">
                <div class="card-header">
                    <h3>Bobot Kriteria</h3>
                </div>
                <div class="card-body">
                    <div class="weights-grid">
                        {#each cids as id}
                            <div class="weight-item">
                                <div class="weight-label">{criteria[id]?.name || id}</div>
                                <div class="weight-value">{Number(criteriaWeights[id] ?? 0).toFixed(4)}</div>
                                <div class="weight-bar">
                                    <div class="weight-bar-fill" style="width: {(Number(criteriaWeights[id] ?? 0) * 100).toFixed(1)}%"></div>
                                </div>
                            </div>
                        {/each}
                    </div>
                </div>
            </div>

            <!-- Priority Matrix Card -->
            <div class="card">
                <div class="card-header">
                    <h3>Matrix Prioritas</h3>
                </div>
                <div class="card-body">
                    <div class="table-responsive">
                        <table class="table table-sm">
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
                                        <td><strong>{alt}</strong></td>
                                        {#each cids as k}
                                            <td>{Number(scores?.[alt]?.[k] ?? 0).toFixed(3)}</td>
                                        {/each}
                                    </tr>
                                {/each}
                            </tbody>
                        </table>
                    </div>
                </div>
            </div>

            <!-- Final Results Card -->
            <div class="card ranking-card">
                <div class="card-header">
                    <h3> Hasil Akhir (Ranking AHP)</h3>
                </div>
                <div class="card-body">
                    {#if !alternatives?.length || !Object.keys(criteria)?.length}
                        <div class="empty-state">
                            <span class="material-symbols-outlined">info</span>
                            <p>Lengkapi data alternatif dan kriteria untuk melihat hasil.</p>
                        </div>
                    {:else}
                        <div class="ranking-list">
                            {#each ranking as row, i}
                                <div class="ranking-item rank-{i + 1}">
                                    <div class="rank-badge">#{i + 1}</div>
                                    <div class="rank-info">
                                        <div class="rank-name">{row.alt}</div>
                                        <div class="rank-scores">
                                            <span class="score-raw">Skor: {row.total.toFixed(4)}</span>
                                            <span class="score-norm">Normalized: {(row.norm * 100).toFixed(2)}%</span>
                                        </div>
                                    </div>
                                    <div class="rank-bar">
                                        <div class="rank-bar-fill" style="width: {(row.norm * 100).toFixed(1)}%"></div>
                                    </div>
                                </div>
                            {/each}
                        </div>
                    {/if}
                </div>
            </div>
        </div>

        <div class="results-sidebar">
            <!-- Import/Export Card -->
            <div class="card">
                <div class="card-header">
                    <h3><span class="material-symbols-outlined">import_export</span> Import/Export</h3>
                </div>
                <div class="card-body">
                    <div class="action-buttons">
                        <button type="button" class="btn btn-primary btn-sm" on:click={exportYAML}>
                            <span class="material-symbols-outlined">download</span>
                            Export YAML
                        </button>
                        <label class="btn btn-outline btn-sm">
                            <span class="material-symbols-outlined">upload</span>
                            Import YAML
                            <input type="file" accept=".yaml,.yml,text/yaml" on:change={onImportYAML} style="display: none;" />
                        </label>
                    </div>
                    
                    {#if selectedFileName}
                        <div class="file-info">
                            <span class="material-symbols-outlined">description</span>
                            {selectedFileName}
                        </div>
                    {/if}
                    
                    {#if importMessage}
                        <div class="import-status" class:success={importMessage.includes('berhasil')} class:error={importMessage.includes('gagal')}>
                            <span class="material-symbols-outlined">
                                {importMessage.includes('berhasil') ? 'check_circle' : 'error'}
                            </span>
                            {importMessage}
                        </div>
                    {/if}
                </div>
            </div>

            <!-- Config Preview Card -->
            <div class="card">
                <div class="card-header">
                    <h3><span class="material-symbols-outlined">code</span> Config Preview</h3>
                </div>
                <div class="card-body">
                    <div class="config-editor">
                        <textarea rows="15" readonly class="form-control code-textarea">{toYAMLSpecific()}</textarea>
                    </div>
                </div>
            </div>
        </div>
    </div>
</div>

<style lang="scss">
@use '../css/_color.scss' as color;
@use '../css/_reusable.scss' as *;

.hasil-container {
    max-width: 1400px;
    margin: 0 auto;
}

.page-header {
    text-align: center;
    margin-bottom: 32px;
    padding: 32px 0;
    background: linear-gradient(135deg, rgba(59, 130, 246, 0.1), rgba(139, 92, 246, 0.1));
    border-radius: 16px;
    border: 1px solid rgba(59, 130, 246, 0.2);
    
    .page-title {
        font-size: 2.5rem;
        font-weight: 700;
        background: linear-gradient(135deg, color.$primary, color.$secondary);
        -webkit-background-clip: text;
        -webkit-text-fill-color: transparent;
        background-clip: text;
        margin: 0 0 12px;
    }
    
    .page-description {
        font-size: 1.1rem;
        color: color.$gray-dark;
        margin: 0;
        max-width: 600px;
        margin: 0 auto;
        line-height: 1.6;
    }
}

.results-grid {
    display: grid;
    grid-template-columns: 2fr 1fr;
    gap: 24px;
    
    @media (max-width: 1024px) {
        grid-template-columns: 1fr;
        gap: 20px;
    }
}

.results-main {
    display: flex;
    flex-direction: column;
    gap: 24px;
}

.results-sidebar {
    display: flex;
    flex-direction: column;
    gap: 20px;
}

// Weight visualization
.weights-grid {
    display: grid;
    gap: 16px;
    
    .weight-item {
        display: flex;
        flex-direction: column;
        gap: 8px;
        
        .weight-label {
            font-weight: 600;
            color: color.$gray-dark;
            font-size: 14px;
        }
        
        .weight-value {
            font-size: 18px;
            font-weight: 700;
            color: color.$primary;
        }
        
        .weight-bar {
            height: 6px;
            background: color.$gray-light;
            border-radius: 3px;
            overflow: hidden;
            
            .weight-bar-fill {
                height: 100%;
                background: linear-gradient(90deg, color.$primary, color.$secondary);
                border-radius: 3px;
                transition: width 0.3s ease;
            }
        }
    }
}

// Ranking visualization
.ranking-card {
    .ranking-list {
        display: flex;
        flex-direction: column;
        gap: 16px;
        
        .ranking-item {
            display: flex;
            align-items: center;
            gap: 16px;
            padding: 16px;
            background: linear-gradient(135deg, #f8fafc, #f1f5f9);
            border-radius: 12px;
            border: 1px solid color.$gray-light;
            transition: all 0.2s ease;
            
            &:hover {
                transform: translateY(-2px);
                box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
            }
            
            &.rank-1 {
                background: linear-gradient(135deg, color.$success-light, rgba(16, 185, 129, 0.1));
                border-color: color.$success;
                
                .rank-badge {
                    background: color.$success;
                    color: white;
                }
                
                .rank-bar-fill {
                    background: linear-gradient(90deg, color.$success, #059669);
                }
            }
            
            &.rank-2 {
                background: linear-gradient(135deg, color.$info-light, rgba(6, 182, 212, 0.1));
                border-color: color.$info;
                
                .rank-badge {
                    background: color.$info;
                    color: white;
                }
                
                .rank-bar-fill {
                    background: linear-gradient(90deg, color.$info, #0891b2);
                }
            }
            
            &.rank-3 {
                background: linear-gradient(135deg, color.$warning-light, rgba(245, 158, 11, 0.1));
                border-color: color.$warning;
                
                .rank-badge {
                    background: color.$warning;
                    color: white;
                }
                
                .rank-bar-fill {
                    background: linear-gradient(90deg, color.$warning, #d97706);
                }
            }
        }
        
        .rank-badge {
            width: 48px;
            height: 48px;
            border-radius: 50%;
            background: color.$gray-medium;
            color: white;
            display: flex;
            align-items: center;
            justify-content: center;
            font-weight: 700;
            font-size: 16px;
            flex-shrink: 0;
        }
        
        .rank-info {
            flex: 1;
            min-width: 0;
            
            .rank-name {
                font-size: 18px;
                font-weight: 600;
                color: color.$gray-darker;
                margin-bottom: 4px;
            }
            
            .rank-scores {
                display: flex;
                gap: 16px;
                font-size: 12px;
                color: color.$gray-dark;
                
                .score-raw, .score-norm {
                    padding: 2px 8px;
                    background: white;
                    border-radius: 4px;
                    border: 1px solid color.$gray-light;
                }
            }
        }
        
        .rank-bar {
            width: 120px;
            height: 8px;
            background: color.$gray-light;
            border-radius: 4px;
            overflow: hidden;
            flex-shrink: 0;
            
            .rank-bar-fill {
                height: 100%;
                background: linear-gradient(90deg, color.$primary, color.$secondary);
                border-radius: 4px;
                transition: width 0.3s ease;
            }
        }
    }
}

// Table responsive
.table-responsive {
    overflow-x: auto;
    border-radius: 8px;
    border: 1px solid color.$gray-light;
}

// Action buttons
.action-buttons {
    display: flex;
    flex-direction: column;
    gap: 12px;
    margin-bottom: 16px;
}

.file-info {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 12px;
    background: color.$info-light;
    border-radius: 8px;
    color: color.$info;
    font-size: 14px;
    margin-top: 12px;
}

.import-status {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 12px;
    border-radius: 8px;
    font-size: 14px;
    margin-top: 12px;
    
    &.success {
        background: color.$success-light;
        color: color.$success;
    }
    
    &.error {
        background: color.$danger-light;
        color: color.$danger;
    }
}

// Config editor
.config-editor {
    .code-textarea {
        font-family: 'JetBrains Mono', 'Fira Code', monospace;
        background: #1e293b;
        color: #e2e8f0;
        border: none;
        resize: vertical;
        line-height: 1.6;
        font-size: 13px;
        
        &:focus {
            box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
            border-color: color.$primary;
        }
    }
}

// Empty state
.empty-state {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 16px;
    padding: 48px 24px;
    text-align: center;
    color: color.$gray-dark;
    
    .material-symbols-outlined {
        font-size: 48px;
        opacity: 0.5;
    }
    
    p {
        margin: 0;
        font-size: 16px;
    }
}

@media (max-width: 768px) {
    .page-header {
        padding: 24px 16px;
        margin-bottom: 24px;
        
        .page-title {
            font-size: 2rem;
        }
        
        .page-description {
            font-size: 1rem;
        }
    }
    
    .ranking-item {
        flex-direction: column;
        text-align: center;
        gap: 12px;
        
        .rank-info {
            order: 1;
        }
        
        .rank-bar {
            order: 2;
            width: 100%;
        }
        
        .rank-badge {
            order: 0;
        }
    }
    
    .action-buttons {
        .btn {
            justify-content: center;
        }
    }
}

</style>