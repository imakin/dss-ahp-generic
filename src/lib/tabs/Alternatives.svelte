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

<div class="bgwhite  inclusive-sans" id="tab-alternatives">
    <h2>Alternatives</h2>
    <ul class="list">
            <li class:adding={isAdding}
                class="add textcenter percent40 inlineblock bglight marginvpx5 marginhpx5"
            >
                <input
                    type="text"
                    placeholder="Nama alternatif"
                    bind:value={newName}
                    class="lineheight3 inlineblock"
                />
                <button
                    type="button"
                    on:click={toggleAdd}
                    class="lineheight3 inlineblock"
                >
                    <span class="material-symbols-outlined">
                        add_box
                    </span>
                </button>
                <button
                    type="button"
                    on:click={addAlt}
                    class="save lineheight3 inlineblock"
                >
                    <span class="material-symbols-outlined">
                        save
                    </span>
                </button>
            </li><li class:deleting={isDeleting}
                class="delete textcenter percent40 inlineblock bglight marginvpx5 marginhpx5"
            >
                <button
                    type="button"
                    on:click={toggleDelete}
                    class="lineheight3"
                >
                    <span class:hidden={!isDeleting}>
                        Pilih yang dihapus
                    </span>
                    <span class="material-symbols-outlined">
                        delete
                    </span>
                </button>
            </li>
        </ul>
    <ul class="list flex">
        {#each alternatives as a, i}
            <li
                class="textcenter percent40 inlineblock lineheight3 bglight marginvpx5 marginhpx5"
                class:ready={altReady(a)}
                class:notready={!altReady(a)}
                title={altReady(a) ? 'Siap' : 'Matrix nilai belum siap'}
            >
                <span class="name">
                    {a}
                </span>
                <button on:click={() => remove(a)} class:hidden={!isDeleting} class="lidelete">
                    <span class="material-symbols-outlined">
                        backspace
                    </span>
                </button>
            </li>
        {/each}
    </ul>
    {#if alternatives.length === 0}
        <p>Belum ada alternatif</p>
    {/if}
</div>

<style lang="scss">
@use '../css/_color.scss' as color;
@use '../css/_reusable.scss' as *;
.ready {
    color: green;
}
.notready {
    color: red;
}

#tab-alternatives {
    ul {
        margin: 0;
        padding: 0;
        li {
            line-height: 3em;
        }
    }
    button {
        border: 0;
        background-color: transparent;
        cursor: pointer;
    }
    .add {
        input {
            transition: width 0.5s, min-width 0.7s, padding 0.5s, margin 0.5s;
            min-width: 0%;
            width: 0%;
            border: 0;
            padding: 0px 0%;
        }
        button {
            transition: width 0.5s, min-width 0.7s;
            width: 90%;
            cursor: pointer;
        }
        .save {
            display: none;
        }
        &.adding{
            input {
                width: 70%;
                padding: 0px 5%;
            }
            button {
                width: 20%;
            }
            .save {
                display: inline-block;
            }
        }
    }
    .delete {
        button {
            width: 100%;
            height: 100%;
        }
        &.deleting {
            background-color: color.$red;
        }
    }
    .lidelete {
        float: right;
        line-height: 3em;
    }
}

</style>