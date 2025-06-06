<script lang="ts">
	import { writable, get } from 'svelte/store';
	import ISO6391 from 'iso-639-1';
	import { _ } from '$lib/i18n';

	import {
		getOrDefault,
		ProductsApi,
		type SelectedImage,
		type Taxonomy,
		type Product,
		type Nutriments
	} from '$lib/api';
	import { preferences } from '$lib/settings';
	import Card from '$lib/ui/Card.svelte';

	import type { PageData } from './$types';
	import TagsString from './TagsString.svelte';
	import { PRODUCT_IMAGE_URL } from '$lib/const';
	import TraceabilityCodes from './TraceabilityCodes.svelte';

	interface Props {
		data: PageData;
	}

	let { data }: Props = $props();

	function getNames(taxo: Taxonomy) {
		return Object.values(taxo)
			.map((t) => getOrDefault(t.name, $preferences.lang))
			.filter((t): t is string => t !== undefined);
	}

	function getLanguage(code: string) {
		return ISO6391.getName(code);
	}

	let categoryNames = $derived(getNames(data.categories));
	let labelNames = $derived(getNames(data.labels));
	let brandNames = $derived(getNames(data.brands));
	let storeNames = $derived(getNames(data.stores));
	let originNames = $derived(getNames(data.origins));
	let countriesNames = $derived(getNames(data.countries));
	let productStore = $derived(writable<Product>(data.state.product));
	let comment = writable('');
	const languageCodes = ISO6391.getAllCodes();
	let languageSearch = $state('');
	let filteredLanguages = $derived(
		languageCodes.filter((code) => {
			if ($productStore.languages_codes[code] !== undefined) {
				return false;
			}
			const language = getLanguage(code);
			return language.toLowerCase().includes(languageSearch.toLowerCase());
		})
	);

	let isSubmitting = $state(false);

	// Initialize nutriments object if it doesn't exist
	function ensureNutriments() {
		productStore.update((store) => {
			if (!store.nutriments) {
				store.nutriments = {} as Nutriments;
			}
			return store;
		});
	}

	// Handle nutriment value changes
	function updateNutriment(key: string, value: number | null) {
		ensureNutriments();
		productStore.update((store) => {
			if (value === null) {
				// @ts-expect-error - We know this is a valid key for nutriments
				delete store.nutriments[key];
			} else {
				// @ts-expect-error - We know this is a valid key for nutriments
				store.nutriments[key] = value;
			}
			return store;
		});
	}

	function handleNutrimentInput(e: Event, key: string) {
		const target = e.currentTarget as HTMLInputElement;
		updateNutriment(key, target.value ? Number(target.value) : null);
	}

	async function submit() {
		isSubmitting = true;
		const product = get(productStore);
		const commentValue = get(comment);

		console.group('Product added/edited');
		console.debug('Submitting', product);
		const ok = await new ProductsApi(fetch).addOrEditProductV2({
			...product,
			comment: commentValue
		});
		console.debug('Submitted', ok);
		console.groupEnd();
		if (ok) {
			window.location.href = '/products/' + product.code;
		} else {
			isSubmitting = false;
		}
	}

	function addLanguage(code: string) {
		productStore.update((store) => {
			store.languages_codes = { ...store.languages_codes, [code]: 0 };
			return store;
		});
	}

	function getIngredientsImage(language: string) {
		const paddedBarcode = get(productStore).code.toString().padStart(13, '0');
		const match = paddedBarcode.match(/^(.{3})(.{3})(.{3})(.*)$/);
		if (!match) {
			throw new Error('Invalid barcode format');
		}
		const path = `${match[1]}/${match[2]}/${match[3]}/${match[4]}`;
		const imageName = 'ingredients_' + language;
		const image = get(productStore).images[imageName];
		if (!image) {
			return '';
		}
		const rev = (image as SelectedImage).rev;
		const filename = `${imageName}.${rev}.400.jpg`;
		return PRODUCT_IMAGE_URL(`${path}/${filename}`);
	}

	$effect(() => {
		productStore.subscribe((it) => {
			console.debug('Product store changed', it);
		});
	});
</script>

<div class="space-y-4">
	<div class="collapse-arrow dark:bg-base-200 collapse bg-white p-2 shadow-md">
		<input type="checkbox" />
		<div class="collapse-title font-semibold">{$_('product.edit.add_language')}</div>
		<div class="collapse-content text-sm">
			<label class="input w-full">
				<span class="icon-[mdi--search] h-5 w-5"></span>
				<input
					type="search"
					placeholder={$_('product.edit.search_languages')}
					bind:value={languageSearch}
				/>
			</label>
			{#if filteredLanguages.length === 0}
				<p class="mt-4 text-center opacity-70">{$_('product.edit.no_languages_found')}</p>
			{:else}
				<div
					class="mt-2 grid max-h-96 grid-cols-[repeat(auto-fill,minmax(200px,1fr))] gap-2 overflow-auto"
				>
					{#each filteredLanguages as code (code)}
						<button class="btn btn-ghost" onclick={() => addLanguage(code)}>
							{getLanguage(code)}
						</button>
					{/each}
				</div>
			{/if}
		</div>
	</div>

	<div class="tabs tabs-box">
		{#each Object.keys($productStore.languages_codes) as code (code)}
			<input
				type="radio"
				name="name_tabs"
				class="tab"
				aria-label={getLanguage(code)}
				defaultChecked={code === $productStore.lang}
			/>
			<div class="tab-content form-control p-6">
				<label for="">{$_('product.edit.name')} ({getLanguage(code)})</label>
				<input
					type="text"
					class="input input-bordered w-full"
					bind:value={$productStore[`product_name_${code}`]}
				/>
			</div>
		{/each}
	</div>

	<Card>
		<div class="form-control mb-4">
			<label for="">{$_('product.edit.quantity')}</label>
			<input type="text" class="input input-bordered w-full" bind:value={$productStore.quantity} />
		</div>

		<div class="form-control mb-4">
			<label for="">{$_('product.edit.emb_code')}</label>
			<input type="text" class="input input-bordered w-full" bind:value={$productStore.emb_codes} />
		</div>

		<div class="form-control mb-4">
			<label for="">{$_('product.edit.packaging')}</label>
			<input type="text" class="input input-bordered w-full" bind:value={$productStore.packaging} />
		</div>

		<div class="form-control mb-4">
			<label for="">{$_('product.edit.manufacturing_places')}</label>
			<input
				type="text"
				class="input input-bordered w-full"
				bind:value={$productStore.manufacturing_places}
			/>
		</div>

		<div class="form-control mb-4">
			<label for="">{$_('product.edit.categories')}</label>
			<TagsString bind:tagsString={$productStore.categories} autocomplete={categoryNames} />
		</div>
		<div class="mb-4">
			<label for="">{$_('product.edit.labels')}</label>
			<TagsString bind:tagsString={$productStore.labels} autocomplete={labelNames} />
		</div>
		<div class="mb-4">
			<label for="">{$_('product.edit.brands')}</label>
			<TagsString bind:tagsString={$productStore.brands} autocomplete={brandNames} />
		</div>
		<div class="mb-4">
			<label for="">{$_('product.edit.stores')}</label>
			<TagsString bind:tagsString={$productStore.stores} autocomplete={storeNames} />
		</div>
		<div class="mb-4">
			<label for="">{$_('product.edit.origins')}</label>
			<TagsString bind:tagsString={$productStore.origins} autocomplete={originNames} />
		</div>
		<div class="mb-4">
			<label for="">{$_('product.edit.countries')}</label>
			<TagsString bind:tagsString={$productStore.countries} autocomplete={countriesNames} />
		</div>

		<div class="mb-4">
			<TraceabilityCodes bind:traceabilityCodes={$productStore.emb_codes} autocomplete={[]} />
		</div>

		<div class="mb-4">
			<label for="">{$_('product.edit.website_url')}</label>
			<input type="text" class="input input-bordered w-full" bind:value={$productStore.link} />
		</div>
	</Card>

	<Card>
		<h3 class="mb-4 text-3xl font-bold">{$_('product.edit.ingredients')}</h3>
		<div class="tabs tabs-box">
			{#each Object.keys($productStore.languages_codes) as code (code)}
				<input
					type="radio"
					name="ingredients_tabs"
					class="tab"
					aria-label={getLanguage(code)}
					defaultChecked={code === $productStore.lang}
				/>
				<div class="tab-content form-control p-6">
					{#if getIngredientsImage(code)}
						<img src={getIngredientsImage(code)} alt="Ingredients" class="mb-4" />
					{:else}
						<p class="alert alert-warning mb-4">{$_('product.edit.no_ingredients_image')}</p>
					{/if}
					<label for="">{$_('product.edit.ingredients_list')} ({getLanguage(code)})</label>
					<div class="form-control mb-4">
						<textarea
							class="textarea textarea-bordered h-40 w-full"
							bind:value={$productStore[`ingredients_text_${code}`]}
						></textarea>
					</div>
				</div>
			{/each}
		</div>
	</Card>

	<Card>
		<h3 class="mb-4 text-3xl font-bold">{$_('product.edit.nutritional_information')}</h3>
		<div class="form-control mb-4">
			<label for="">{$_('product.edit.serving_size')}</label>
			<input
				type="text"
				class="input input-bordered w-full"
				bind:value={$productStore.serving_size}
			/>
		</div>

		<div class="form-control mb-4">
			<label class="label cursor-pointer justify-start gap-2">
				<input type="checkbox" class="checkbox" bind:checked={$productStore.no_nutrition_data} />
				<span class="label-text">{$_('product.edit.no_nutrition_data')}</span>
			</label>
		</div>

		{#if !$productStore.no_nutrition_data}
			<div class="grid grid-cols-2 gap-4">
				<div class="form-control mb-4">
					<label for="">Energy (kJ)</label>
					<input
						type="number"
						class="input input-bordered w-full"
						value={$productStore.nutriments?.['energy-kj_100g'] ?? ''}
						oninput={(e) => handleNutrimentInput(e, 'energy-kj_100g')}
					/>
				</div>
				<div class="form-control mb-4">
					<label for="">Energy (kcal)</label>
					<input
						type="number"
						class="input input-bordered w-full"
						value={$productStore.nutriments?.['energy-kcal_100g'] ?? ''}
						oninput={(e) => handleNutrimentInput(e, 'energy-kcal_100g')}
					/>
				</div>
				<div class="form-control mb-4">
					<label for="">Fat (g)</label>
					<input
						type="number"
						class="input input-bordered w-full"
						value={$productStore.nutriments?.fat_100g ?? ''}
						oninput={(e) => handleNutrimentInput(e, 'fat_100g')}
					/>
				</div>
				<div class="form-control mb-4">
					<label for="">Saturated Fat (g)</label>
					<input
						type="number"
						class="input input-bordered w-full"
						value={$productStore.nutriments?.['saturated-fat_100g'] ?? ''}
						oninput={(e) => handleNutrimentInput(e, 'saturated-fat_100g')}
					/>
				</div>
				<div class="form-control mb-4">
					<label for="">Carbohydrates (g)</label>
					<input
						type="number"
						class="input input-bordered w-full"
						value={$productStore.nutriments?.carbohydrates_100g ?? ''}
						oninput={(e) => handleNutrimentInput(e, 'carbohydrates_100g')}
					/>
				</div>
				<div class="form-control mb-4">
					<label for="">Sugars (g)</label>
					<input
						type="number"
						class="input input-bordered w-full"
						value={$productStore.nutriments?.sugars_100g ?? ''}
						oninput={(e) => handleNutrimentInput(e, 'sugars_100g')}
					/>
				</div>
				<div class="form-control mb-4">
					<label for="">Proteins (g)</label>
					<input
						type="number"
						class="input input-bordered w-full"
						value={$productStore.nutriments?.proteins_100g ?? ''}
						oninput={(e) => handleNutrimentInput(e, 'proteins_100g')}
					/>
				</div>
				<div class="form-control mb-4">
					<label for="">Salt (g)</label>
					<input
						type="number"
						class="input input-bordered w-full"
						value={$productStore.nutriments?.salt_100g ?? ''}
						oninput={(e) => handleNutrimentInput(e, 'salt_100g')}
					/>
				</div>
				<div class="form-control mb-4">
					<label for="">Sodium (g)</label>
					<input
						type="number"
						class="input input-bordered w-full"
						value={$productStore.nutriments?.sodium_100g ?? ''}
						oninput={(e) => handleNutrimentInput(e, 'sodium_100g')}
					/>
				</div>
			</div>
		{:else}
			<div class="alert alert-info">{$_('product.edit.no_nutrition_specified')}</div>
		{/if}
	</Card>

	<Card>
		<div class="form-control mb-4">
			<label for="comment">{$_('product.edit.comment')}</label>
			<input
				id="comment"
				type="text"
				class="input input-bordered w-full"
				placeholder={$_('product.edit.comment_placeholder')}
				bind:value={$comment}
			/>
		</div>
	</Card>

	<div class="sticky bottom-2.5 z-1 rounded-md">
		<button class="btn btn-primary w-full" onclick={submit} disabled={isSubmitting}>
			{#if isSubmitting}
				<span class="loading loading-spinner loading-sm"></span>
			{/if}
			{$_('product.edit.save_btn')}
		</button>
	</div>
</div>

<details>
	<summary>{$_('product.edit.debug')}</summary>
	<pre>{JSON.stringify(data, null, 2)}</pre>
</details>
