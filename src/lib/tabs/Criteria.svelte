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

<div class="card kriteria  inclusive-sans">
    <h2>Kriteria & weighting</h2>

    <div class="add percent40 inlineblock marginvpx5 marginhpx5"
        class:adding={isAdding}
    >
        <input type="text"
            placeholder="Nama kriteria"
            class="lineheight3"
            class:hidden={!isAdding}
            bind:value={newName}
        />
        <button
            type="button"
            on:click={() => {
                const nextId = `C${Object.keys(criteria).length + 1}`;
                const next = { ...criteria, [nextId]: { name: newName, type: 'benefit', weight: 0 } };
                dispatch('criteriaChange', next);
            }}
            class="lineheight3"
            class:hidden={!isAdding}
        >
            <span class="material-symbols-outlined">
                save
            </span>
        </button>
        <button 
            type="button"
            on:click={toggleAdd}
            class="lineheight3 inlineblock paddingh1"
        >{isAdding ? 'Batal' : 'Tambah Kriteria'}</button>
    </div>

    <table class="table">
        <thead>
            <tr><th>Kode</th><th>Nama</th><th></th><th></th></tr>
        </thead>
        <tbody>
            {#each Object.keys(criteria) as key, i}
                <tr>
                    <td data-label="Kode">{key}</td>
                    <td data-label="Nama">{criteria[key].name}</td>
                    <td data-label="">
                        <!-- <select
                            bind:value={criteria[key].type}
                            on:change={(e)=>setType(key, e.target.value)}
                        >
                            <option value="benefit">benefit</option>
                            <option value="cost">cost</option>
                        </select> -->
                    </td>
                    <td data-label="">
                        <button
                            type="button"
                            on:click={() => {
                                const next = { ...criteria };
                                delete next[key];
                                dispatch('criteriaChange', next);
                            }}
                        >
                            <span class="material-symbols-outlined">
                                delete
                            </span>
                        </button>
                    </td>
                </tr>
            {/each}
        </tbody>
    </table>
</div>

<style lang="scss">
@use '../css/_color.scss' as color;
@use '../css/_reusable.scss' as *;

.kriteria {
    select, input {
        border: 0;
        background-color: color.$gray;
        padding: 1em 1em;
        text-transform: capitalize;
        font-weight: 400;
        font-size: 0.8em;
    }
}

button {
    border: 0;
    background-color: rgba(218, 82, 111, 0.1);
    cursor: pointer;
    padding: 1em;
}
@media screen and (max-width: 1024px) {
    table thead {
        display: none;
    }
    table, tbody, tr{
        display: block;
        width: 100%;
    }
    table tr {
        padding-bottom: 1em;
        margin-bottom: 1em;
        border-bottom: solid 5px color.$gray;
    }
    table td {
        display: inline-block;
        width: 49%;
        @media screen and (max-width: 600px) {
            width: 100%;
        }
    }

    table td::before {
        content: attr(data-label); /* Mengambil teks dari atribut data-label */
        font-weight: bold;
        display: block;
        margin-bottom: 5px;
        color: #333;
    }
}

</style>