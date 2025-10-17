<script lang="ts">
    import { createEventDispatcher } from 'svelte';
    const dispatch = createEventDispatcher<{ganti: string[]}>();
    export let alternatives: string[] = [];
    export let scores = {};

    function altReady(name: string) {
        try {
            let target_score = scores[name];
            let ready = true;
            Object.keys(target_score).forEach(k => {
                if (isNaN(Number(target_score[k]))) ready = false;
                if (target_score[k]=0) ready = false;
            });
            return ready;
        } catch {
            return false;
        }
    }

    function remove(name: string) {
        const next = alternatives.filter(a => a !== name);
        dispatch('ganti', next);
    }

    let newName = '';
    let isAdding = false;
    function toggleAdd() {
        isAdding = !isAdding;
    }
    function addAlt() {
        if (isAdding) {
            const name = newName.trim() || `Alternatif ${alternatives.length + 1}`;
            const next = [...alternatives, name];
            dispatch('ganti', next);
            newName = '';
        }
    }

    let isDeleting = false;
    function toggleDelete() {
        isDeleting = !isDeleting;
    }
</script>

<div class="alternatives-container inclusive-sans" id="tab-alternatives">
    <div class="page-header">
        <h2><span class="material-symbols-outlined">category</span> Kelola Alternatif</h2>
        <p>Tambah, edit, atau hapus alternatif yang akan dibandingkan dalam analisis AHP</p>
    </div>
    <div class="controls-panel">
        <div class="add-control" class:active={isAdding}>
            <div class="control-header">
                <span class="material-symbols-outlined">add_circle</span>
                <span>Tambah Alternatif</span>
            </div>
            {#if isAdding}
                <div class="add-form">
                    <input
                        type="text"
                        placeholder="Masukkan nama alternatif"
                        bind:value={newName}
                        class="form-control"
                    />
                    <div class="form-actions">
                        <button type="button" class="btn btn-primary btn-sm" on:click={addAlt}>
                            <span class="material-symbols-outlined">save</span>
                            Simpan
                        </button>
                        <button type="button" class="btn btn-outline btn-sm" on:click={toggleAdd}>
                            Batal
                        </button>
                    </div>
                </div>
            {:else}
                <button type="button" class="btn btn-primary" on:click={toggleAdd}>
                    <span class="material-symbols-outlined">add</span>
                    Tambah Baru
                </button>
            {/if}
        </div>

        <div class="delete-control" class:active={isDeleting}>
            <button
                type="button"
                class="btn {isDeleting ? 'btn-danger' : 'btn-outline'}"
                on:click={toggleDelete}
            >
                <span class="material-symbols-outlined">
                    {isDeleting ? 'close' : 'delete'}
                </span>
                {isDeleting ? 'Selesai Hapus' : 'Mode Hapus'}
            </button>
            {#if isDeleting}
                <p class="delete-hint">Klik alternatif yang ingin dihapus</p>
            {/if}
        </div>
    </div>
    <div class="alternatives-grid">
        {#each alternatives as a, i}
            <div
                class="alternative-card"
                class:ready={altReady(a)}
                class:notready={!altReady(a)}
                class:deleting={isDeleting}
            >
                <div class="card-content">
                    <div class="alternative-icon">
                        <span class="material-symbols-outlined">
                            {altReady(a) ? 'check_circle' : 'pending'}
                        </span>
                    </div>
                    <div class="alternative-info">
                        <h4 class="alternative-name">{a}</h4>
                        <p class="alternative-status">
                            {altReady(a) ? 'Siap untuk analisis' : 'Perlu melengkapi data'}
                        </p>
                    </div>
                    {#if isDeleting}
                        <button 
                            class="delete-btn" 
                            on:click={() => remove(a)}
                            title="Hapus {a}"
                        >
                            <span class="material-symbols-outlined">delete</span>
                        </button>
                    {/if}
                </div>
                <div class="readiness-indicator" class:ready={altReady(a)}></div>
            </div>
        {/each}
    </div>
    
    {#if alternatives.length === 0}
        <div class="empty-state">
            <span class="material-symbols-outlined">category</span>
            <h3>Belum ada alternatif</h3>
            <p>Tambahkan alternatif pertama untuk memulai analisis AHP</p>
        </div>
    {/if}
</div>

<style lang="scss">
@use '../css/_color.scss' as color;
@use '../css/_reusable.scss' as *;

.alternatives-container {
    max-width: 1200px;
    margin: 0 auto;
}

.page-header {
    text-align: center;
    margin-bottom: 32px;
    padding: 24px;
    background: linear-gradient(135deg, rgba(59, 130, 246, 0.1), rgba(139, 92, 246, 0.1));
    border-radius: 12px;
    border: 1px solid rgba(59, 130, 246, 0.2);
    
    h2 {
        display: flex;
        align-items: center;
        justify-content: center;
        gap: 12px;
        font-size: 1.8rem;
        font-weight: 700;
        color: color.$gray-darker;
        margin: 0 0 8px;
        
        .material-symbols-outlined {
            font-size: 2rem;
            color: color.$primary;
        }
    }
    
    p {
        color: color.$gray-dark;
        margin: 0;
        font-size: 1rem;
    }
}

.controls-panel {
    display: grid;
    grid-template-columns: 1fr auto;
    gap: 24px;
    margin-bottom: 32px;
    
    @media (max-width: 768px) {
        grid-template-columns: 1fr;
        gap: 16px;
    }
}

.add-control {
    background: white;
    border-radius: 12px;
    padding: 20px;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
    border: 1px solid color.$gray-light;
    transition: all 0.2s ease;
    
    &.active {
        border-color: color.$primary;
        box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
    }
    
    .control-header {
        display: flex;
        align-items: center;
        gap: 12px;
        font-weight: 600;
        color: color.$gray-darker;
        margin-bottom: 16px;
        
        .material-symbols-outlined {
            color: color.$primary;
        }
    }
    
    .add-form {
        display: flex;
        flex-direction: column;
        gap: 16px;
        
        .form-actions {
            display: flex;
            gap: 12px;
            
            @media (max-width: 480px) {
                flex-direction: column;
            }
        }
    }
}

.delete-control {
    display: flex;
    flex-direction: column;
    gap: 12px;
    align-items: flex-end;
    
    @media (max-width: 768px) {
        align-items: stretch;
    }
    
    .delete-hint {
        font-size: 14px;
        color: color.$danger;
        margin: 0;
        text-align: right;
        
        @media (max-width: 768px) {
            text-align: center;
        }
    }
}

.alternatives-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
    gap: 20px;
    
    @media (max-width: 640px) {
        grid-template-columns: 1fr;
    }
}

.alternative-card {
    background: white;
    border-radius: 12px;
    overflow: hidden;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
    border: 1px solid color.$gray-light;
    transition: all 0.2s ease;
    position: relative;
    
    &:hover {
        transform: translateY(-2px);
        box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
    }
    
    &.ready {
        border-color: color.$success;
        
        .alternative-icon .material-symbols-outlined {
            color: color.$success;
        }
        
        .readiness-indicator {
            background: color.$success;
        }
    }
    
    &.notready {
        border-color: color.$warning;
        
        .alternative-icon .material-symbols-outlined {
            color: color.$warning;
        }
        
        .readiness-indicator {
            background: color.$warning;
        }
    }
    
    &.deleting {
        .card-content {
            padding-right: 60px;
        }
        
        .delete-btn {
            display: flex;
        }
    }
    
    .card-content {
        display: flex;
        align-items: center;
        gap: 16px;
        padding: 20px;
        position: relative;
    }
    
    .alternative-icon {
        width: 48px;
        height: 48px;
        border-radius: 50%;
        background: color.$gray-light;
        display: flex;
        align-items: center;
        justify-content: center;
        flex-shrink: 0;
        
        .material-symbols-outlined {
            font-size: 24px;
        }
    }
    
    .alternative-info {
        flex: 1;
        min-width: 0;
        
        .alternative-name {
            font-size: 18px;
            font-weight: 600;
            color: color.$gray-darker;
            margin: 0 0 4px;
        }
        
        .alternative-status {
            font-size: 14px;
            color: color.$gray-dark;
            margin: 0;
        }
    }
    
    .delete-btn {
        position: absolute;
        right: 16px;
        top: 50%;
        transform: translateY(-50%);
        width: 40px;
        height: 40px;
        border-radius: 50%;
        border: none;
        background: color.$danger;
        color: white;
        cursor: pointer;
        display: none;
        align-items: center;
        justify-content: center;
        transition: all 0.2s ease;
        
        &:hover {
            background: #dc2626;
            transform: translateY(-50%) scale(1.1);
        }
        
        .material-symbols-outlined {
            font-size: 20px;
        }
    }
    
    .readiness-indicator {
        height: 4px;
        width: 100%;
        background: color.$gray-light;
        transition: background-color 0.2s ease;
        
        &.ready {
            background: color.$success;
        }
    }
}

.empty-state {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 16px;
    padding: 64px 24px;
    text-align: center;
    color: color.$gray-dark;
    background: white;
    border-radius: 12px;
    border: 2px dashed color.$gray-medium;
    
    .material-symbols-outlined {
        font-size: 64px;
        opacity: 0.5;
        color: color.$gray-medium;
    }
    
    h3 {
        margin: 0;
        font-size: 1.5rem;
        color: color.$gray-darker;
    }
    
    p {
        margin: 0;
        font-size: 1rem;
        max-width: 400px;
    }
}

</style>