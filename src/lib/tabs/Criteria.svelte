<script>
    import { createEventDispatcher } from 'svelte';
    const dispatch = createEventDispatcher();
    export let criteria = {};

    function setType(id, t) {
        const next = { ...criteria, [id]: { ...criteria[id], type: t } };
        dispatch('criteriaChange', next);
    }

    function clamp(x) { return Math.max(0, Math.min(1, x)); }
    
    
    let newName = '';
    let isAdding = false;
    function toggleAdd() {
        isAdding = !isAdding;
    }

</script>

<div class="criteria-container inclusive-sans">
    <div class="page-header">
        <h2><span class="material-symbols-outlined">rule</span> Kelola Kriteria</h2>
        <p>Definisikan kriteria yang akan digunakan dalam proses pengambilan keputusan</p>
    </div>

    <div class="add-criteria-panel" class:active={isAdding}>
        <div class="panel-header">
            <h3><span class="material-symbols-outlined">add_circle</span> Tambah Kriteria Baru</h3>
        </div>
        <div class="panel-content">
            {#if isAdding}
                <div class="add-form">
                    <input 
                        type="text"
                        placeholder="Masukkan nama kriteria"
                        class="form-control"
                        bind:value={newName}
                    />
                    <div class="form-actions">
                        <button
                            type="button"
                            class="btn btn-primary"
                            on:click={() => {
                                const nextId = `C${Object.keys(criteria).length + 1}`;
                                const next = { ...criteria, [nextId]: { name: newName, type: 'benefit', weight: 0 } };
                                dispatch('criteriaChange', next);
                                newName = '';
                                isAdding = false;
                            }}
                        >
                            <span class="material-symbols-outlined">save</span>
                            Simpan Kriteria
                        </button>
                        <button 
                            type="button"
                            class="btn btn-outline"
                            on:click={toggleAdd}
                        >
                            Batal
                        </button>
                    </div>
                </div>
            {:else}
                <button 
                    type="button"
                    class="btn btn-primary btn-lg"
                    on:click={toggleAdd}
                >
                    <span class="material-symbols-outlined">add</span>
                    Tambah Kriteria
                </button>
            {/if}
        </div>
    </div>

    <div class="criteria-list">
        {#each Object.keys(criteria) as key, i}
            <div class="criteria-card">
                <div class="criteria-info">
                    <div class="criteria-header">
                        <span class="criteria-code">{key}</span>
                    </div>
                    <h4 class="criteria-name">{criteria[key].name}</h4>
                </div>
                <div class="criteria-actions">
                    <button
                        type="button"
                        class="btn-delete"
                        on:click={() => {
                            const next = { ...criteria };
                            delete next[key];
                            dispatch('criteriaChange', next);
                        }}
                        title="Hapus kriteria {criteria[key].name}"
                    >
                        <span class="material-symbols-outlined">delete</span>
                    </button>
                </div>
            </div>
        {/each}
    </div>
    
    {#if Object.keys(criteria).length === 0}
        <div class="empty-state">
            <span class="material-symbols-outlined">rule</span>
            <h3>Belum ada kriteria</h3>
            <p>Tambahkan kriteria pertama untuk memulai analisis AHP</p>
        </div>
    {/if}
</div>

<style lang="scss">
@use '../css/_color.scss' as color;
@use '../css/_reusable.scss' as *;

.criteria-container {
    max-width: 1000px;
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

.add-criteria-panel {
    background: white;
    border-radius: 12px;
    padding: 24px;
    margin-bottom: 32px;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
    border: 1px solid color.$gray-light;
    transition: all 0.2s ease;
    
    &.active {
        border-color: color.$primary;
        box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
    }
    
    .panel-header {
        h3 {
            display: flex;
            align-items: center;
            gap: 12px;
            font-size: 1.25rem;
            font-weight: 600;
            color: color.$gray-darker;
            margin: 0 0 20px;
            
            .material-symbols-outlined {
                color: color.$primary;
            }
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

.criteria-list {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    gap: 20px;
    
    @media (max-width: 640px) {
        grid-template-columns: 1fr;
    }
}

.criteria-card {
    background: white;
    border-radius: 12px;
    padding: 20px;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
    border: 1px solid color.$gray-light;
    transition: all 0.2s ease;
    display: flex;
    align-items: center;
    gap: 16px;
    
    &:hover {
        transform: translateY(-2px);
        box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
    }
    
    .criteria-icon {
        width: 48px;
        height: 48px;
        border-radius: 50%;
        background: linear-gradient(135deg, color.$primary-light, rgba(59, 130, 246, 0.2));
        display: flex;
        align-items: center;
        justify-content: center;
        flex-shrink: 0;
        
        .material-symbols-outlined {
            font-size: 24px;
            color: color.$primary;
        }
    }
    
    .criteria-info {
        flex: 1;
        min-width: 0;
        
        .criteria-header {
            display: flex;
            align-items: center;
            gap: 12px;
            margin-bottom: 4px;
            
            .criteria-code {
                font-size: 12px;
                font-weight: 600;
                color: color.$primary;
                background: color.$primary-light;
                padding: 2px 8px;
                border-radius: 4px;
            }
            
            .criteria-type {
                font-size: 11px;
                font-weight: 500;
                color: color.$success;
                background: color.$success-light;
                padding: 2px 6px;
                border-radius: 4px;
                text-transform: uppercase;
                letter-spacing: 0.05em;
            }
        }
        
        .criteria-name {
            font-size: 16px;
            font-weight: 600;
            color: color.$gray-darker;
            margin: 0;
            line-height: 1.4;
        }
    }
    
    .criteria-actions {
        .btn-delete {
            width: 40px;
            height: 40px;
            border-radius: 50%;
            border: none;
            background: color.$danger-light;
            color: color.$danger;
            cursor: pointer;
            display: flex;
            align-items: center;
            justify-content: center;
            transition: all 0.2s ease;
            
            &:hover {
                background: color.$danger;
                color: white;
                transform: scale(1.1);
            }
            
            .material-symbols-outlined {
                font-size: 20px;
            }
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

@media (max-width: 768px) {
    .criteria-card {
        flex-direction: column;
        text-align: center;
        
        .criteria-info .criteria-header {
            justify-content: center;
        }
    }
}

</style>