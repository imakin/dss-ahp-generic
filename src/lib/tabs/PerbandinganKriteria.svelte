<script>
	import { createEventDispatcher, onMount } from 'svelte';
	const dispatch = createEventDispatcher();

	// Props
	export let criteria = {}; // { C1: { name, type }, ... }
	export let initialMatrix = null; // optional: { [idA]: { [idB]: number } }

	// Derived
	$: ids = Object.keys(criteria);
	$: n = ids.length;

	// Saaty scale options
	const scale = [
		{ value: 1/9, label: '1/9' },
		{ value: 1/8, label: '1/8' },
		{ value: 1/7, label: '1/7' },
		{ value: 1/6, label: '1/6' },
		{ value: 1/5, label: '1/5' },
		{ value: 1/4, label: '1/4' },
		{ value: 1/3, label: '1/3' },
		{ value: 1/2, label: '1/2' },
		{ value: 1,   label: 'sama penting' },
		{ value: 2,   label: 'sedikit lebih penting' },
		{ value: 3,   label: 'sedikit lebih penting' },
		{ value: 4,   label: 'lebih penting' },
		{ value: 5,   label: 'lebih penting' },
		{ value: 6,   label: 'sangat lebih penting' },
		{ value: 7,   label: 'sangat lebih penting' },
		{ value: 8,   label: 'ekstrem lebih penting' },
		{ value: 9,   label: 'ekstrem lebih penting' }
	];

	let matrix = {};

	function initMatrix() {
		const next = {};
		ids.forEach((ia) => {
			next[ia] = {};
			ids.forEach((ib) => {
				if (ia === ib) next[ia][ib] = 1;
				else next[ia][ib] = 1;
			});
		});
		// If initial provided, merge
		if (initialMatrix && typeof initialMatrix === 'object') {
			ids.forEach((ia) => {
				ids.forEach((ib) => {
					const v = initialMatrix?.[ia]?.[ib];
					if (typeof v === 'number' && isFinite(v) && v > 0) {
						next[ia][ib] = v;
					}
				});
			});
		}
		// enforce reciprocity & diagonals
		ids.forEach((ia, i) => {
			ids.forEach((ib, j) => {
				if (i === j) next[ia][ib] = 1;
				else if (i > j) next[ia][ib] = 1 / next[ib][ia];
			});
		});
		matrix = next;
		emit();
	}

	onMount(() => {
		initMatrix();
	});

	// Re-init when criteria changes in size
	$: if (n && Object.keys(matrix).length !== n) {
		initMatrix();
	}

	function setPair(a, b, value) {
		const v = Number(value);
		if (!isFinite(v) || v <= 0) return;
		if (a === b) return;
		// update both sides
		matrix = { ...matrix, [a]: { ...matrix[a], [b]: v } };
		matrix = { ...matrix, [b]: { ...matrix[b], [a]: 1 / v } };
		emit();
	}

	function emit() {
		const weights = computeWeightsNormalized(matrix, ids);
		dispatch('matrixChange', matrix);
		dispatch('weightsChange', weights);
	}

	function resetMatrix() {
		initMatrix();
	}

	function formatCell(v) {
		// Try show as fraction if close to 1/k or k
		const reciprocals = [2,3,4,5,6,7,8,9];
		for (const k of reciprocals) {
			if (Math.abs(v - 1/k) < 1e-6) return `1/${k}`;
			if (Math.abs(v - k) < 1e-6) return `${k}`;
		}
		if (Math.abs(v - 1) < 1e-6) return '1';
		return Number(v).toFixed(3);
	}

	function computeWeightsGM(A, ids) {
		const m = ids.length;
		if (!m) return {};
		// Geometric mean method
		const gms = ids.map((i) => {
			let prod = 1;
			ids.forEach((j) => {
				const val = A?.[i]?.[j] ?? 1;
				prod *= val;
			});
			return Math.pow(prod, 1 / m);
		});
		const sum = gms.reduce((a,b)=>a+b,0) || 1;
		const weights = {};
		ids.forEach((id, idx) => { weights[id] = gms[idx] / sum; });
		return weights;
	}

	function multiplyMatrixVector(A, ids, w) {
		const out = {};
		ids.forEach(i => {
			let s = 0;
			ids.forEach(j => {
				const aij = A?.[i]?.[j] ?? 1;
				const wj = w?.[j] ?? 0;
				s += aij * wj;
			});
			out[i] = s;
		});
		return out;
	}

	// reactive derived values to avoid recomputing in template
	$: weights = computeWeightsNormalized(matrix, ids);
	$: Aw = multiplyMatrixVector(matrix, ids, weights);
    $: eigenvalue = (() => {
        // λmax = (Aw)i / wi
        const lambda = ids.map(i => {
            const awi = Aw[i] ?? 0;
            const wi = weights[i] ?? 1;
            return wi ? (awi / wi) : 0;
        });
        const avg = lambda.reduce((a,b)=>a+b,0) / (lambda.length || 1);
        return avg;
    })();
    $: consistencyindex = (() => {
        if (n > 1) {
            return (((eigenvalue ?? 0) - n) / (n - 1));
        }
        else {
            return 0;
        }
    })();
    $: consistencyratio = (() => {
        if (n > 1) {
            if ((randomindexes[n] ?? 0) > 0) {
                return consistencyindex / (randomindexes[n] ?? 1)
            }
            else {
                return 0;
            }
        }
        else {
            return 0;
        }
    })();

    let randomindexes = [0, 0, 0.58, 0.90, 1.12, 1.24, 1.32, 1.41, 1.45, 1.49, 1.51, 1.48, 1.56, 1.57, 1.59];

    function computeWeightsNormalized(A, ids) {
        const n = ids.length;
        if (!n) return {};
        // hitung jumlah kolom
        const colSums = {};
        ids.forEach(j => {
            let s = 0;
            ids.forEach(i => { s += (A?.[i]?.[j] ?? 1); });
            colSums[j] = s || 1;
        });
        // normalisasi kolom dan rata-rata baris
        const weightsRaw = {};
        ids.forEach(i => {
            let sumRow = 0;
            ids.forEach(j => {
                sumRow += (A?.[i]?.[j] ?? 1) / colSums[j];
            });
            weightsRaw[i] = sumRow / n;
        });
        // normalisasi akhir agar jumlah 1
        const total = Object.values(weightsRaw).reduce((a,b)=>a+b,0) || 1;
        const w = {};
        ids.forEach(i => { w[i] = weightsRaw[i] / total; });
        return w;
    }
</script>

<div class="card inclusive-sans">
	<h2>Perbandingan Antar Kriteria (AHP)</h2>
	{#if n === 0}
		<p>Tambahkan kriteria terlebih dahulu.</p>
	{:else}
		<!-- <div class="controls marginvpx10">
			<button type="button" on:click={resetMatrix} class="lineheight3">Reset Matriks</button>
		</div> -->
		<div class="table-wrapper">
			<table class="table ahp-table">
				<thead>
					<tr>
						<th></th>
						{#each ids as id}
							<th>{criteria[id]?.name || id}</th>
						{/each}
					</tr>
				</thead>
				<tbody>
					{#each ids as rowId, i}
						<tr>
							<th>{criteria[rowId]?.name || rowId}</th>
							{#each ids as colId, j}
								<td>
									{#if i === j}
										<div class="diag">1</div>
									{:else if i < j}
										<select
											class="pair-select"
											bind:value={matrix[rowId][colId]}
											on:change={(e)=> {
                                                setPair(rowId, colId, e.target.value);
                                            }}
										>
											{#each scale as s}
												<option value={s.value}
                                                    data-info={`${criteria[rowId].name} ${s.label} dibandingkan ${criteria[colId].name}`}
                                                    data-value={s.value%1==0? s.value : s.label}
                                                    title={`${criteria[rowId].name} ${s.label} dibandingkan ${criteria[colId].name}`}
                                                >
                                                    {#if (s.value%1==0) }
                                                        {s.value}
                                                    {:else}
                                                        {s.label}
                                                    {/if}
                                                </option>
											{/each}
										</select>
									{:else}
										<div class="mirror">{formatCell(matrix[rowId][colId])}</div>
									{/if}
								</td>
							{/each}
						</tr>
					{/each}
				</tbody>
			</table>
		</div>

        <div class="summary-perbandingan">
            <div class="weights">
                <h3>Bobot Kriteria</h3>
                <h4> (rata-rata baris ternormalisasi)</h4>
                <table class="textleft">
                    <thead >
                        <tr><th>Kriteria</th><th>Bobot</th></tr>
                    </thead>
                    <tbody>
                    {#each ids as id}
                        <tr>
                            <td><strong>{criteria[id]?.name || id}:</strong></td>
                            <td>{Number(weights[id] ?? 0).toFixed(4)}</td>
                        </tr>
                    {/each}
                    </tbody>
                </table>
            </div>

            <div class="aw">
                <h3>A × w</h3>
                <h4> (matriks sebelum normalisasi × bobot)</h4>
                <table class="textleft">
                    <thead>
                        <tr><th>Kriteria</th><th>Nilai</th></tr>
                    </thead>
                    <tbody>
                    {#each ids as id}
                        <tr>
                            <td><strong>{criteria[id]?.name || id}:</strong></td>
                            <td>{Number(Aw[id] ?? 0).toFixed(4)}</td>
                        </tr>
                    {/each}
                    </tbody>
                </table>
            </div>

            <div class="eigenvalue">
                <h3>Eigenvalue (λmax)</h3>
                {(eigenvalue).toFixed(4)}
            </div>

            <div class="consistency-index">
                <h3>Consistency Index (CI)</h3>
                {consistencyindex.toFixed(4)}
            </div>

            <div class="random-index">
                <h3>Random Index (RI)</h3>
                <p>Banyak kriteria (n) = {n}</p>
                <p>RI = {(randomindexes[n] ?? 0).toFixed(4)}</p>
            </div>

            <div class="consistency-ratio" class:warning={consistencyratio >= 0.1}>
                <h3>Consistency Ratio (CR)</h3>
                {(consistencyratio)}
            </div>
        </div>
	{/if}
</div>

<style lang="scss">
@use '../css/_color.scss' as color;
@use '../css/_reusable.scss' as *;
.card {
    font-family: Arial;
}

.table-wrapper { overflow: auto; }
.ahp-table {
	border-collapse: collapse;
	th, td { border-bottom: 4px solid color.$gray; padding: .5em; text-align: center; }
	thead th { position: sticky; top: 0; background: color.$gray; }
}
.pair-select {
	border: 0; background: color.$gray; padding: .5em; min-width: 110px;
}
.diag, .mirror { padding: .5em; }
.weights,.aw {
    ul { list-style: none; padding: 0; }
}
.summary-perbandingan {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
    gap: 1em;
    margin-top: 1em;
    border-top: solid 4px color.$gray;
    padding-top: 1em;
    h3 { margin: 0 0 0.2em; }
    h4 { margin: 0 0 0.5em; font-weight: normal; font-size: 0.9em; color: rgba(0,0,0,0.9); }
    div {
        background: color.$gray;
        padding: 1em;
        border-radius: 8px;
        box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    }
    .warning {
        background: #ffdddd;
        border: solid 2px red;
    }
}
</style>


