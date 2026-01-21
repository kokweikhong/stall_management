<script lang="ts">
	import { onMount } from 'svelte';

	let payedOrders: Order[] = $state([]);

	// fetch paid orders from local storage on mount
	onMount(() => {
		payedOrders = JSON.parse(localStorage.getItem('paidOrders') || '[]');
	});

	interface FoodItem {
		id: number;
		name: string;
		price: number;
		category: 'main' | 'sauce';
	}

	const menu: FoodItem[] = [
		{ id: 1, name: '猪肠粉', price: 120, category: 'main' },
		{ id: 2, name: '水料', price: 130, category: 'main' },
		{ id: 3, name: '炸料', price: 180, category: 'main' },
		{ id: 4, name: '酿料', price: 250, category: 'main' },
		{ id: 5, name: '甜酱', price: 0, category: 'sauce' },
		{ id: 6, name: '红酱', price: 0, category: 'sauce' },
		{ id: 7, name: '咖喱', price: 70, category: 'sauce' },
		{ id: 8, name: '咖喱猪皮', price: 220, category: 'sauce' },
		{ id: 9, name: '酱清', price: 0, category: 'sauce' },
		{ id: 10, name: '香菇肉碎', price: 500, category: 'sauce' }
	];

	const mains = menu.filter((item) => item.category === 'main');
	function clearPaidOrders() {
		if (confirm('Are you sure you want to clear all paid orders? This cannot be undone.')) {
			localStorage.removeItem('paidOrders');
			payedOrders = [];
		}
	}
	const sauces = menu.filter((item) => item.category === 'sauce');

	interface OrderItem extends FoodItem {
		quantity: number;
		totalPrice: number;
	}

	type OrderStatus = 'pending' | 'served' | 'preparing';

	interface Order {
		date: string;
		tableNumber: number;
		status: OrderStatus;
		items: OrderItem[];
		totalPrice: number;
	}

	const defaultTableNumbers: number[] = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15];

	function initializeOrders(): Order[] {
		const orders = defaultTableNumbers.map((tableNumber) => ({
			date: new Date().toISOString().split('T')[0],
			tableNumber,
			status: 'pending' as OrderStatus,
			totalPrice: 0,
			items: []
		}));

		return orders;
	}

	let orders: Order[] = $state(initializeOrders());

	// add item to order and the quantity is 1 by 1
	// if category is sauce, only allow one sauce per order
	function addItemToOrder(tableNumber: number, item: FoodItem) {
		orders = orders.map((order) => {
			if (order.tableNumber === tableNumber) {
				const existingItem = order.items.find((orderItem) => orderItem.id === item.id);
				if (existingItem) {
					if (item.category === 'sauce') {
						// Do not increase quantity for sauce
					} else {
						existingItem.quantity += 1;
						existingItem.totalPrice = existingItem.quantity * existingItem.price;
					}
				} else {
					order.items.push({ ...item, quantity: 1, totalPrice: item.price });
				}
				// Always update totalPrice after any change
				order.totalPrice = order.items.reduce((total, item) => total + item.totalPrice, 0);
				// if order totalPrice is 0, set status to pending
				// if order totalPrice > 0 and status is pending, set status to preparing
				if (order.totalPrice === 0) {
					order.status = 'pending';
				} else if (order.totalPrice > 0 && order.status === 'pending') {
					order.status = 'preparing';
				}
				return order;
			}
			return order;
		});
	}

	// reduce item quantity from order only for main
	function reduceItemQuantityFromOrder(tableNumber: number, itemId: number) {
		orders = orders.map((order) => {
			if (order.tableNumber === tableNumber) {
				const existingItem = order.items.find((orderItem) => orderItem.id === itemId);
				if (existingItem && existingItem.category === 'main') {
					existingItem.quantity -= 1;
					existingItem.totalPrice = existingItem.quantity * existingItem.price;
					if (existingItem.quantity <= 0) {
						order.items = order.items.filter((item) => item.id !== itemId);
					}
				}
				order.totalPrice = order.items.reduce((total, item) => total + item.totalPrice, 0);
				if (order.totalPrice === 0) {
					order.status = 'pending';
				} else if (order.totalPrice > 0 && order.status === 'pending') {
					order.status = 'preparing';
				}
				return order;
			}
			return order;
		});
	}

	// remove sauce from order
	function removeSauceFromOrder(tableNumber: number, sauceId: number) {
		orders = orders.map((order) => {
			if (order.tableNumber === tableNumber) {
				order.items = order.items.filter((item) => item.id !== sauceId);
				order.totalPrice = order.items.reduce((total, item) => total + item.totalPrice, 0);
				return order;
			}
			return order;
		});
	}

	function payForOrder(tableNumber: number) {
		orders = orders.map((order) => {
			if (order.tableNumber === tableNumber) {
				// Clone the order before resetting
				const paidOrder = { ...order, items: [...order.items] };
				payedOrders = [...payedOrders, paidOrder];
				localStorage.setItem('paidOrders', JSON.stringify(payedOrders));
				// Now reset the order
				order.items = [];
				order.totalPrice = 0;
				order.status = 'pending';
				return order;
			}
			return order;
		});
	}

	let selectedTableNumber: number | null = $state(1);

	let showPaidOrdersModal = $state(false);
	let copySuccess = $state(false);

	function copyPaidOrdersToClipboard() {
		if (!payedOrders.length) return;
		const text = payedOrders
			.map(
				(order, i) =>
					`#${i + 1} | Table: ${order.tableNumber} | Date: ${order.date} | Total: RM ${(order.totalPrice / 100).toFixed(2)}`
			)
			.join('\n');
		navigator.clipboard.writeText(text).then(() => {
			copySuccess = true;
			setTimeout(() => (copySuccess = false), 1500);
		});
	}
</script>

<div class="responsive-container my-6 text-center text-orange-500">
	<h1 class="text-2xl font-bold">酱美丽 | 猪肠粉</h1>
</div>

<!-- Table Numbers -->
<div class="responsive-container">
	<div class="flex flex-wrap gap-4">
		{#each orders as order}
			<button
				class="touch-target cursor-pointer rounded-lg px-4 py-2 font-semibold text-white shadow transition focus:ring-2 focus:outline-none
				{selectedTableNumber === order.tableNumber
					? 'bg-blue-600 hover:bg-blue-700 focus:ring-blue-400'
					: order.status === 'pending'
						? 'bg-gray-600 hover:bg-gray-700 focus:ring-gray-400'
						: order.status === 'preparing'
							? 'bg-orange-600 hover:bg-orange-700 focus:ring-orange-400'
							: 'bg-green-600 hover:bg-green-700 focus:ring-green-400'}"
				onclick={() => (selectedTableNumber = order.tableNumber)}
			>
				<span class="flex flex-col gap-1">
					<span>{order.tableNumber}</span>
				</span>
			</button>
		{/each}
	</div>
</div>

<div class="responsive-container mt-6 flex justify-center">
	<button
		class="mb-4 rounded-lg bg-green-600 px-6 py-2 font-semibold text-white shadow transition hover:bg-green-700"
		onclick={() => (showPaidOrdersModal = true)}
		aria-label="Show total paid orders"
	>
		Show Paid Orders
	</button>

	{#if showPaidOrdersModal}
		<div class="fixed inset-0 z-50 flex items-center justify-center bg-black/40">
			<div class="animate-fade-in relative w-full max-w-md rounded-2xl bg-white p-8 shadow-2xl">
				<div
					class="mx-auto my-4 max-w-2xl rounded-2xl border border-gray-200 bg-white/90 p-6 shadow-xl sm:p-8"
				>
					<h2 class="mb-4 flex items-center gap-2 text-xl font-bold text-yellow-700">
						<svg
							xmlns="http://www.w3.org/2000/svg"
							class="h-6 w-6 text-yellow-400"
							fill="none"
							viewBox="0 0 24 24"
							stroke="currentColor"
							><path
								stroke-linecap="round"
								stroke-linejoin="round"
								stroke-width="2"
								d="M12 8c-1.657 0-3 .895-3 2s1.343 2 3 2 3 .895 3 2-1.343 2-3 2zm0 0c-4.418 0-8 1.79-8 4v2h16v-2c0-2.21-3.582-4-8-4z"
							/></svg
						>
						Today's Total Sales
					</h2>
					<div class="mt-6 flex flex-col items-center gap-2 text-center text-lg font-semibold">
						<span class="text-3xl text-yellow-700"
							>RM
							{payedOrders.reduce((total, order) => total + order.totalPrice, 0) / 100}</span
						>
					</div>
				</div>
				<button
					class="absolute top-3 right-3 text-2xl font-bold text-gray-400 hover:text-gray-700 focus:outline-none"
					onclick={() => (showPaidOrdersModal = false)}
					aria-label="Close modal">&times;</button
				>
				<h2 class="mb-4 flex items-center gap-2 text-xl font-bold text-green-700">
					<svg
						xmlns="http://www.w3.org/2000/svg"
						class="h-6 w-6 text-green-400"
						fill="none"
						viewBox="0 0 24 24"
						stroke="currentColor"
						><path
							stroke-linecap="round"
							stroke-linejoin="round"
							stroke-width="2"
							d="M17 9V7a5 5 0 00-10 0v2a2 2 0 00-2 2v5a2 2 0 002 2h12a2 2 0 002-2v-5a2 2 0 00-2-2z"
						/></svg
					>
					Total Paid Orders
				</h2>
				<div class="mb-4 flex flex-col items-center gap-2 text-center text-lg font-semibold">
					<span class="text-3xl text-green-700">{payedOrders.length}</span>
					<button
						class="mt-2 inline-flex items-center gap-2 rounded bg-gray-200 px-3 py-1 text-sm font-medium text-gray-700 transition hover:bg-gray-300 active:bg-gray-400"
						onclick={copyPaidOrdersToClipboard}
						aria-label="Copy paid orders to clipboard"
						disabled={payedOrders.length === 0}
					>
						<svg
							xmlns="http://www.w3.org/2000/svg"
							class="h-4 w-4"
							fill="none"
							viewBox="0 0 24 24"
							stroke="currentColor"
							><path
								stroke-linecap="round"
								stroke-linejoin="round"
								stroke-width="2"
								d="M8 16h8M8 12h8m-7 8h6a2 2 0 002-2V7a2 2 0 00-2-2H7a2 2 0 00-2 2v13a2 2 0 002 2z"
							/></svg
						>
						{copySuccess ? 'Copied!' : 'Copy List'}
					</button>
				</div>
				<div class="mb-4 flex justify-center">
					<button
						class="rounded bg-red-600 px-4 py-2 font-semibold text-white hover:bg-red-700"
						onclick={clearPaidOrders}
						aria-label="Clear all paid orders"
						disabled={payedOrders.length === 0}
					>
						Clear Paid Orders
					</button>
				</div>
				<hr class="my-4" />
				<!-- Paid Orders List -->
				<div class="max-h-60 overflow-y-auto">
					{#if payedOrders.length === 0}
						<p class="text-center text-gray-500">No paid orders yet.</p>
					{:else}
						<ul class="divide-y divide-gray-200">
							{#each payedOrders as order, i}
								<li
									class="flex flex-col gap-1 px-2 py-2 sm:flex-row sm:items-center sm:justify-between"
								>
									<span class="font-medium text-gray-700">Table {order.tableNumber}</span>
									<span class="text-xs text-gray-400">{order.date}</span>
									<span class="font-bold text-green-700">RM {order.totalPrice / 100}</span>
								</li>
							{/each}
						</ul>
					{/if}
				</div>
			</div>
		</div>
	{/if}
</div>

<!-- Order Details -->
{#if selectedTableNumber !== null}
	<div class="responsive-container mt-8">
		<div class="mb-6">
			<div
				class="mx-auto max-w-2xl rounded-2xl border border-gray-200 bg-white/90 p-6 shadow-xl sm:p-8"
			>
				<h2 class="mb-4 flex items-center gap-2 text-xl font-bold text-balance text-blue-700">
					<svg
						xmlns="http://www.w3.org/2000/svg"
						class="h-6 w-6 text-blue-400"
						fill="none"
						viewBox="0 0 24 24"
						stroke="currentColor"
						><path
							stroke-linecap="round"
							stroke-linejoin="round"
							stroke-width="2"
							d="M8 17l4 4 4-4m0-5V3m-8 9v6a2 2 0 002 2h4a2 2 0 002-2v-6"
						/></svg
					>
					Order Details for Table <span class="ml-1 text-blue-900">{selectedTableNumber}</span>
				</h2>
				<div class="mt-6">
					<div class="flex flex-col gap-2 sm:flex-row sm:items-center sm:justify-between">
						<h3 class="mb-2 text-lg font-semibold">Current Order</h3>
						{#if orders.find((order) => order.tableNumber === selectedTableNumber)?.totalPrice! > 0}
							<button
								class="touch-target mb-4 rounded-lg bg-green-600 px-6 py-2 font-semibold text-white shadow transition hover:bg-green-700"
								onclick={() => payForOrder(selectedTableNumber!)}
							>
								Pay RM {(orders.find((order) => order.tableNumber === selectedTableNumber)
									?.totalPrice || 0) / 100}
							</button>
						{/if}
					</div>
					{#if orders.find((order) => order.tableNumber === selectedTableNumber)?.items.length === 0}
						<p class="py-8 text-center text-gray-500 italic">No items in the order.</p>
					{:else}
						<ul class="divide-y divide-gray-200 overflow-hidden rounded-lg bg-gray-50">
							{#each orders.find((order) => order.tableNumber === selectedTableNumber)?.items as item}
								<li class="flex flex-wrap items-center justify-between gap-2 px-4 py-3">
									<span class="font-semibold text-gray-900">{item.name}</span>
									<span class="text-gray-700">x {item.quantity}</span>
									<span class="font-medium text-green-700">RM {item.totalPrice / 100}</span>
								</li>
							{/each}
						</ul>
						<p class="mt-6 border-t pt-4 text-right text-lg font-bold">
							Total Price: <span class="text-green-700"
								>RM {(orders.find((order) => order.tableNumber === selectedTableNumber)
									?.totalPrice || 0) / 100}</span
							>
						</p>
					{/if}
				</div>
			</div>
		</div>
		<!-- Display order details here -->
		<div class="mb-4">
			<h3 class="mb-2 text-lg font-semibold">Mains</h3>
			<div class="grid w-full grid-cols-1 gap-4 sm:grid-cols-2 md:grid-cols-3">
				{#each mains as main}
					<div
						class="grid grid-cols-[1fr_auto] items-center justify-between gap-4 rounded-lg border p-4 shadow md:grid"
					>
						<div class="flex w-full flex-col items-start justify-between gap-4">
							<h4 class="font-semibold text-balance">{main.name}</h4>
							<div class="grid w-full grid-cols-[auto_1fr] items-center justify-start gap-4">
								<span class="flex-1">
									<span
										class="rounded-full bg-gray-200 px-2 py-1 text-sm font-medium text-gray-700"
									>
										RM {main.price / 100}
									</span>
									<span class="mx-2"> x </span>
									<span
										class="rounded-full bg-gray-200 px-2 py-1 text-sm font-medium text-gray-700"
									>
										{#if orders
											.find((order) => order.tableNumber === selectedTableNumber)
											?.items.find((item) => item.id === main.id)}
											{orders
												.find((order) => order.tableNumber === selectedTableNumber)
												?.items.find((item) => item.id === main.id)?.quantity}
										{:else}
											0
										{/if}
									</span>
									<span class="mx-2"> = </span>
									<span
										class="rounded-full bg-green-200 px-2 py-1 text-sm font-medium text-green-800"
									>
										RM
										{#if orders
											.find((order) => order.tableNumber === selectedTableNumber)
											?.items.find((item) => item.id === main.id)}
											{orders
												.find((order) => order.tableNumber === selectedTableNumber)
												?.items.find((item) => item.id === main.id)?.totalPrice! / 100}
										{:else}
											0
										{/if}
									</span>
								</span>
							</div>
						</div>
						<div class="flex w-full flex-col items-center justify-between gap-4">
							<button
								class="touch-target rounded bg-green-500 px-3 py-1 text-white hover:bg-green-600"
								onclick={() => addItemToOrder(selectedTableNumber!, main)}
								aria-label="Add one {main.name} to order"
							>
								<svg
									xmlns="http://www.w3.org/2000/svg"
									class="h-5 w-5"
									viewBox="0 0 20 20"
									fill="currentColor"
								>
									<path
										fill-rule="evenodd"
										d="M10 5a1 1 0 011 1v3h3a1 1 0 110 2h-3v3a1 1 0 11-2 0v-3H6a1 1 0 110-2h3V6a1 1 0 011-1z"
										clip-rule="evenodd"
									/>
								</svg>
							</button>
							<button
								class="touch-target rounded bg-red-500 px-3 py-1 text-white hover:bg-red-600"
								onclick={() => reduceItemQuantityFromOrder(selectedTableNumber!, main.id)}
								aria-label="Remove one {main.name} from order"
							>
								<svg
									xmlns="http://www.w3.org/2000/svg"
									class="h-5 w-5"
									viewBox="0 0 20 20"
									fill="currentColor"
								>
									<path
										fill-rule="evenodd"
										d="M5 10a1 1 0 011-1h8a1 1 0 110 2H6a1 1 0 01-1-1z"
										clip-rule="evenodd"
									/>
								</svg>
							</button>
						</div>
					</div>
				{/each}
			</div>
		</div>
		<div class="mb-4">
			<h3 class="mt-6 mb-2 text-lg font-semibold">Sauces</h3>
			<div class="flex flex-wrap gap-4">
				{#each sauces as sauce}
					<button
						class={orders
							.find((order) => order.tableNumber === selectedTableNumber)
							?.items.find((item) => item.id === sauce.id)
							? 'touch-target rounded-lg bg-red-600 px-4 py-2 font-semibold text-white hover:bg-red-700'
							: 'touch-target rounded-lg bg-gray-300 px-4 py-2 text-black hover:bg-gray-500'}
						onclick={orders
							.find((order) => order.tableNumber === selectedTableNumber)
							?.items.find((item) => item.id === sauce.id)
							? () => removeSauceFromOrder(selectedTableNumber!, sauce.id)
							: () => addItemToOrder(selectedTableNumber!, sauce)}
					>
						<span class="flex flex-col items-center justify-center gap-2">
							<span>{sauce.name}</span>
							<span>RM {sauce.price / 100}</span>
						</span>
					</button>
				{/each}
			</div>
		</div>
	</div>
{/if}

<style lang="postcss">
	@reference "tailwindcss";
</style>
