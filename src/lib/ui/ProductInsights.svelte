<script lang="ts">
	// import { json } from '@sveltejs/kit';
	import Card from './Card.svelte';
	import { GoogleGenAI } from '@google/genai';

	type Props = {
		product: any;
	};

	let { product }: Props = $props();

	// ── Tab state ──────────────────────────────────────────────────────────────
	type Tab = 'summary' | 'deepdive' | 'scores' | 'suggestions';
	let activeTab = $state<Tab>('summary');
	let aiLoading = $state(false);
	// ── Alternatives state ─────────────────────────────────────────────────────
	let alternatives = $state<any[]>([]);
	let pairings = $state<any[]>([]);
	let altLoading = $state(false);
	let altError = $state<string | null>(null);
	let aiError = $state<any>([]);
	let aiSummary = $state<any>();
	let aiDeepDive = $state<any>();

	// ── Scores derived ─────────────────────────────────────────────────────────
	let scores = $derived(
		product
			? {
					nutriscore: product.nutriscore_grade?.toUpperCase() ?? null,
					nova: product.nova_group ?? null,
					ecoscore: product.ecoscore_grade?.toUpperCase() ?? null,
					sugar: product.nutriments?.sugars_100g ?? null,
					fat: product.nutriments?.fat_100g ?? null,
					saturatedFat: product.nutriments?.['saturated-fat_100g'] ?? null,
					salt: product.nutriments?.salt_100g ?? null,
					fiber: product.nutriments?.fiber_100g ?? null,
					proteins: product.nutriments?.proteins_100g ?? null,
					energy: product.nutriments?.energy_kcal_100g ?? null
				}
			: null
	);

	// ── Nutrigrade badge colour ────────────────────────────────────────────────
	function nutriscoreBadge(grade: string | null) {
		const map: Record<string, string> = {
			A: 'bg-green-500 text-white',
			B: 'bg-lime-400 text-white',
			C: 'bg-yellow-400 text-black',
			D: 'bg-orange-500 text-white',
			E: 'bg-red-600 text-white'
		};
		return grade
			? (map[grade] ?? 'bg-base-300 text-base-content')
			: 'bg-base-300 text-base-content';
	}

	function novaBadge(nova: number | null) {
		const map: Record<number, string> = {
			1: 'bg-green-500 text-white',
			2: 'bg-lime-400 text-white',
			3: 'bg-orange-400 text-white',
			4: 'bg-red-600 text-white'
		};
		return nova ? (map[nova] ?? 'bg-base-300') : 'bg-base-300';
	}

	function ecoscoreBadge(grade: string | null) {
		return nutriscoreBadge(grade); // same colour logic
	}

	// ── Nutrient bar helpers ───────────────────────────────────────────────────
	function nutrientBarWidth(value: number | null, max: number): string {
		if (value == null) return '0%';
		return Math.min(100, (value / max) * 100) + '%';
	}

	function nutrientBarColour(value: number | null, lowThreshold: number, highThreshold: number) {
		if (value == null) return 'bg-base-300';
		if (value <= lowThreshold) return 'bg-green-500';
		if (value <= highThreshold) return 'bg-orange-400';
		return 'bg-red-500';
	}

	// ── Fetch alternatives ─────────────────────────────────────────────────────
	async function fetchAlternatives() {
		if (!product?.categories_tags?.length) return;
		altLoading = true;
		altError = null;

		const ingredients =
			product.ingredients_tags?.slice(0, 3).map((i: string) => i.replace('en:', '')) ?? [];
		const query = ingredients.join(' ');
		console.log(query);
		if (!query) {
			altLoading = false;
			return;
		}

		try {
			const res = await fetch(
				`https://world.openfoodfacts.org/api/v2/search?q=${encodeURIComponent(query)}&fields=product_name,code,nutriscore_grade,nova_group,image_front_small_url&sort_by=nutriscore_score&size=8`
			);
			if (!res.ok) throw new Error(res.statusText);
			const data = await res.json();

			const all = (data.products ?? []).filter((p: any) => p.code !== product.code);
			alternatives = all.slice(0, 4).reverse();

			// Use the next batch as "pairings" (different category proxy for prototype)
			pairings = all.slice(4, 7);
		} catch (e) {
			altError = 'Could not load suggestions.';
			console.error('Alternatives fetch error:', e);
		} finally {
			altLoading = false;
		}
	}

	// __ Fetch AI Summary_______________________________________________________

	async function getAiSummary() {
		if (!product) return;

		const ai = new GoogleGenAI({ apiKey: import.meta.env.VITE_GEMINI_API_KEY });
		aiLoading = true;

		const productContext = JSON.stringify({
			name: product.product_name,
			brands: product.brands,
			nutriscore: product.nutriscore_grade,
			nova_group: product.nova_group,
			ecoscore: product.ecoscore_grade,
			ingredients: product.ingredients_text?.slice(0, 500),
			nutriments: product.nutriments
		});

		const summaryPrompt = `You are a bilingual (English/French) nutrition expert for OpenFoodFacts Canada. 
Respond ONLY in English.
Given this product data: ${productContext}

Return ONLY a JSON object (no markdown, no preamble) with exactly:
{
  "summary": "2-3 sentence plain-language summary of this product and it's health profile for a Canadian consumer",
  "deepdive": "3-4 sentence detailed analysis covering: processing level (NOVA), nutritional highlights/concerns, environmental impact (ecoscore), and one practical tip"
}`;
		try {
			const res = await ai.models.generateContent({
				model: 'gemini-3-flash-preview',
				contents: summaryPrompt
			});

			console.log('AI Text', res.text);
			const text = res.text;
			const clean = text?.replace(/```json|```/g, '').trim();
			const response = clean && JSON.parse(clean);
			// const clean = {
			// 	summary:
			// 		'Nutella is a popular hazelnut and cocoa spread with a poor nutritional profile, receiving a Nutri-Score E due to its very high sugar and fat content. For Canadian consumers, it is important to treat this as an occasional indulgence rather than a breakfast staple, as more than half of the product consists of added sugar.',
			// 	deepdive:
			// 		'Classified as NOVA 4, this is an ultra-processed food containing refined ingredients such as palm oil and emulsifiers which provide high caloric density but very little fiber. The primary nutritional concerns are the high levels of saturated fat (10.6g/100g) and sugar (56.3g/100g), while the environmental impact remains unknown due to an unavailable Eco-score. To mitigate the health impact, limit your serving to one tablespoon and pair it with fiber-rich foods like whole-grain toast or sliced fruit to help stabilize blood sugar levels.'
			// };
			console.log('Parsed clean', response.summary);
			// if (!res.ok) throw new Error(`API error ${res.status}`);
			// const data = await res.json();

			// const parsed = JSON.parse(clean);
			aiSummary = response.summary;
			aiDeepDive = response.deepdive;
		} catch (e) {
			aiError = 'Could not load AI insights. Please try again.';
			console.error('AI fetch error:', e);
		} finally {
			aiLoading = false;
		}
	}

	// ── Reactive effects ───────────────────────────────────────────────────────
	$effect(() => {
		if (product?.code) {
			getAiSummary();
			fetchAlternatives();
		}
	});
</script>

<Card>
	<!-- Header ---->
	<div class="mb-5 flex items-center justify-between">
		<div class="flex items-center gap-2">
			<span class="text-primary text-xl">✦</span>
			<h2 class="text-lg font-bold">Insights</h2>
			<span
				class="badge badge-sm bg-primary text-primary-content border-0 px-2 py-0.5 text-[10px] font-semibold tracking-wide uppercase opacity-80"
			>
				Beta
			</span> --
		</div>

		<!-- Language toggle -->
		<div class="join border-base-300 overflow-hidden rounded-xl border">
			<button class="join-item btn btn-xs px-3 font-semibold transition-colors"> EN </button>
			<button class="join-item btn btn-xs px-3 font-semibold transition-colors"> FR </button>
		</div>
	</div>

	<!-- Tabs --->
	<div role="tablist" class="tabs tabs-box mb-5 w-full">
		{#each [['summary', '📋 Summary'], ['deepdive', '🔬 Deep Dive'], ['scores', '📊 Scores'], ['suggestions', '💡 Suggestions']] as [id, label]}
			<button
				role="tab"
				class="tab flex-1 text-xs font-semibold sm:text-sm {activeTab === id ? 'tab-active' : ''}"
				onclick={() => (activeTab = id as Tab)}
			>
				{label}
			</button>
		{/each}
	</div>

	<!-- ── Tab: Summary ----->
	{#if activeTab === 'summary'}
		<div class="min-h-[100px]">
			{#if aiLoading}
				<div class="flex flex-col gap-3">
					<div class="skeleton h-4 w-full rounded"></div>
					<div class="skeleton h-4 w-5/6 rounded"></div>
					<div class="skeleton h-4 w-4/6 rounded"></div>
				</div>
				<!-- {:else if aiError}
				<div class="alert alert-error text-sm">
					<span>⚠️ {aiError}</span>
					<button class="btn btn-xs ml-auto" onclick={() => getAiSummary()}> Retry </button>
				</div> -->
			{:else if aiSummary}
				<p class="text-base-content/90 leading-relaxed">{aiSummary}</p>

				Quick flags
				<div class="mt-4 flex flex-wrap gap-2">
					{#if scores?.nova === 4}
						<span class="badge badge-error gap-1 text-xs">⚠ Ultra-processed</span>
					{/if}
					{#if scores?.sugar != null && scores.sugar > 20}
						<span class="badge badge-warning gap-1 text-xs">🍬 High sugar</span>
					{/if}
					{#if scores?.saturatedFat != null && scores.saturatedFat > 5}
						<span class="badge badge-warning gap-1 text-xs">🧈 High saturated fat</span>
					{/if}
					{#if scores?.salt != null && scores.salt > 1.5}
						<span class="badge badge-warning gap-1 text-xs">🧂 High salt</span>
					{/if}
					{#if scores?.fiber != null && scores.fiber > 3}
						<span class="badge badge-success gap-1 text-xs">🌾 Good fiber</span>
					{/if}
					{#if scores?.nutriscore === 'A' || scores?.nutriscore === 'B'}
						<span class="badge badge-success gap-1 text-xs">✅ Good Nutri-Score</span>
					{/if}
				</div>
			{:else}
				<p class="text-base-content/50 text-sm italic">No insights available for this product.</p>
			{/if}
		</div>
	{/if}

	<!-- ── Tab: Deep Dive ------->
	{#if activeTab === 'deepdive'}
		<div class="min-h-[100px]">
			{#if aiLoading}
				<div class="flex flex-col gap-3">
					<div class="skeleton h-4 w-full rounded"></div>
					<div class="skeleton h-4 w-5/6 rounded"></div>
					<div class="skeleton h-4 w-full rounded"></div>
					<div class="skeleton h-4 w-3/4 rounded"></div>
				</div>
				<!-- {:else if aiError}
				<div class="alert alert-error text-sm">
					<span>⚠️ {aiError}</span>
					<button class="btn btn-xs ml-auto" onclick={() => getAiSummary()}> Retry </button>
				</div> -->
			{:else if aiDeepDive}
				<p class="text-base-content/90 leading-relaxed">{aiDeepDive}</p>

				{#if product.ingredients_text}
					<div class="bg-base-200 mt-5 rounded-xl p-4">
						<p class="text-secondary mb-1 text-xs font-bold tracking-wider uppercase">
							Ingredients
						</p>
						<p class="text-base-content/70 text-xs leading-relaxed">
							{product.ingredients_text.slice(0, 300)}{product.ingredients_text.length > 300
								? '…'
								: ''}
						</p>
					</div>
				{/if}
			{:else}
				<p class="text-base-content/50 text-sm italic">No deep dive available.</p>
			{/if}
		</div>
	{/if}

	<!-- ── Tab: Scores ------>
	{#if activeTab === 'scores'}
		{#if scores}
			<!-- Grade badges row ------->
			<div class="mb-6 grid grid-cols-3 gap-3">
				<!-- Nutri-Score ----->
				<div class="bg-base-200 flex flex-col items-center gap-1 rounded-xl p-3">
					<span class="text-secondary text-xs font-bold tracking-wider uppercase">Nutri-Score</span>
					<span
						class="flex h-10 w-10 items-center justify-center rounded-full text-lg font-black {nutriscoreBadge(
							scores.nutriscore
						)}"
					>
						{scores.nutriscore ?? '?'}
					</span>
				</div>

				<!-- NOVA ----->
				<div class="bg-base-200 flex flex-col items-center gap-1 rounded-xl p-3">
					<span class="text-secondary text-xs font-bold tracking-wider uppercase">NOVA</span>
					<span
						class="flex h-10 w-10 items-center justify-center rounded-full text-lg font-black {novaBadge(
							scores.nova
						)}"
					>
						{scores.nova ?? '?'}
					</span>
				</div>

				<!-- Eco-Score ----->
				<div class="bg-base-200 flex flex-col items-center gap-1 rounded-xl p-3">
					<span class="text-secondary text-xs font-bold tracking-wider uppercase">Eco-Score</span>
					<span
						class="flex h-10 w-10 items-center justify-center rounded-full text-lg font-black {ecoscoreBadge(
							scores.ecoscore
						)}"
					>
						{scores.ecoscore ?? '?'}
					</span>
				</div>
			</div>

			<!-- Nutrient bars -->
			<div class="flex flex-col gap-3">
				{#each [{ label: 'Sugar', value: scores.sugar, unit: 'g', max: 50, low: 5, high: 15 }, { label: 'Fat', value: scores.fat, unit: 'g', max: 50, low: 10, high: 25 }, { label: 'Saturated Fat', value: scores.saturatedFat, unit: 'g', max: 20, low: 1.5, high: 5 }, { label: 'Salt', value: scores.salt, unit: 'g', max: 5, low: 0.3, high: 1.5 }, { label: 'Fiber', value: scores.fiber, unit: 'g', max: 15, low: 0, high: 3 }, { label: 'Protein', value: scores.proteins, unit: 'g', max: 40, low: 0, high: 10 }] as nutrient}
					<div>
						<div class="mb-1 flex justify-between text-xs">
							<span class="font-semibold">{nutrient.label}</span>
							<span class="text-base-content/60">
								{nutrient.value != null ? `${nutrient.value} ${nutrient.unit} / 100g` : 'Unknown'}
							</span>
						</div>
						<div class="bg-base-300 h-2 w-full overflow-hidden rounded-full">
							<div
								class="h-full rounded-full transition-all duration-700 {nutrientBarColour(
									nutrient.value,
									nutrient.low,
									nutrient.high
								)}"
								style="width: {nutrientBarWidth(nutrient.value, nutrient.max)}"
							></div>
						</div>
					</div>
				{/each}
			</div>

			{#if scores.energy != null}
				<p class="text-base-content/50 mt-4 text-right text-xs">
					Energy: {scores.energy} kcal / 100g
				</p>
			{/if}
		{:else}
			<p class="text-base-content/50 text-sm italic">No score data available.</p>
		{/if}
	{/if}

	<!-- ── Tab: Suggestions ------>
	{#if activeTab === 'suggestions'}
		{#if altLoading}
			<div class="flex flex-col gap-3">
				{#each [1, 2, 3] as _}
					<div class="bg-base-200 flex items-center gap-3 rounded-xl p-3">
						<div class="skeleton h-10 w-10 rounded-lg"></div>
						<div class="flex flex-1 flex-col gap-1">
							<div class="skeleton h-3 w-3/4 rounded"></div>
							<div class="skeleton h-3 w-1/3 rounded"></div>
						</div>
					</div>
				{/each}
			</div>
		{:else if altError}
			<div class="alert alert-error text-sm">{altError}</div>
		{:else}
			<!-- Better alternatives------->
			{#if alternatives.length > 0}
				<div class="mb-5">
					<p class="text-secondary mb-2 text-xs font-bold tracking-wider uppercase">
						Better Alternatives
					</p>
					<div class="flex flex-col gap-2">
						{#each alternatives as alt}
							<a
								href="/products/{alt.code}"
								class="bg-base-200 hover:bg-base-300 flex items-center gap-3 rounded-xl p-3 transition-colors"
							>
								<div class="min-w-0 flex-1">
									<p class="truncate text-sm font-semibold">{alt.product_name ?? alt.code}</p>
									<p class="text-base-content/50 text-xs">
										Nutri-Score: {alt.nutriscore_grade?.toUpperCase() ?? '?'}
									</p>
								</div>
								<span
									class="flex h-7 w-7 shrink-0 items-center justify-center rounded-full text-sm font-black {nutriscoreBadge(
										alt.nutriscore_grade?.toUpperCase() ?? null
									)}"
								>
									{alt.nutriscore_grade?.toUpperCase() ?? '?'}
								</span>
							</a>
						{/each}
					</div>
				</div>
			{/if}

			{#if alternatives.length === 0 && pairings.length === 0}
				<p class="text-base-content/50 text-sm italic">No suggestions found for this product.</p>
			{/if}
		{/if}
	{/if}
</Card>
