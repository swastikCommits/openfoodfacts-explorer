<script lang="ts">
	import type { ProductReduced } from '$lib/api';
	import { navigating } from '$app/state';

	interface Props {
		product: ProductReduced;
	}

	let { product }: Props = $props();
</script>

<a
	href={`/products/${product.code}`}
	class="btn btn-ghost text-primary dark:bg-base-300 pointer-events-none h-auto justify-normal rounded-2xl bg-white p-4 text-start shadow-md"
	class:pointer-events-none={navigating.to}
>
	<div class="flex flex-row items-center">
		<div class="mr-4 flex w-16 shrink-0 items-center justify-center">
			{#if navigating.to?.params?.barcode === product.code}
				<span class="loading loading-ring loading-lg mx-auto my-auto"></span>
			{:else if product.image_front_small_url}
				<img
					src={product.image_front_small_url}
					class="h-16 rounded-lg object-cover"
					alt="Product front"
				/>
			{/if}
		</div>
		<div>
			<p class="text-lg">
				{product.product_name ?? product.code}
			</p>
			<p class="mt-2 text-sm font-light">
				{product.brands} - {product.quantity}
			</p>
		</div>
	</div>
</a>
