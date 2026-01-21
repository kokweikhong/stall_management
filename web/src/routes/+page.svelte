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

	// calculate total price by table number
	function calculateTotalPrice(tableNumber: number): number {
		const order = orders.find((order) => order.tableNumber === tableNumber);
		if (!order) return 0;
		return order.items.reduce((total, item) => total + item.totalPrice, 0);
	}

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
				// push to local storage
				payedOrders.push(order);
				localStorage.setItem('paidOrders', JSON.stringify(payedOrders));
				// reset order
				order.items = [];
				order.totalPrice = 0;
				order.status = 'pending';
				return order;
			}
			return order;
		});
	}

	let selectedTableNumber: number | null = $state(1);
</script>

<!-- Table Numbers -->
<div class="grid grid-cols-3 gap-4 md:grid-cols-6 lg:grid-cols-9">
	{#each orders as order}
		<button
			class="cursor-pointer rounded-lg px-4 py-2 font-semibold text-white shadow transition focus:ring-2 focus:outline-none
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

<!-- Order Details -->
{#if selectedTableNumber !== null}
	<div class="mt-8">
		<div>
			<!-- total paidorders -->
			<p class="mb-4 text-lg font-semibold">
				Total Paid Orders: <span class="text-green-700">{payedOrders.length}</span>
			</p>
		</div>
		<div class="mb-6">
			<h2 class="mb-4 text-xl font-bold">Order Details for Table {selectedTableNumber}</h2>
			<div class="mt-6">
				<div class="flex items-center justify-between">
					<h3 class="mb-2 text-lg font-semibold">Current Order</h3>
					{#if orders.find((order) => order.tableNumber === selectedTableNumber)?.totalPrice > 0}
						<button
							class="mb-4 rounded bg-green-600 px-4 py-2 font-semibold text-white hover:bg-green-700"
							onclick={() => payForOrder(selectedTableNumber!)}
						>
							Pay RM {(orders.find((order) => order.tableNumber === selectedTableNumber)
								?.totalPrice || 0) / 100}
						</button>
					{/if}
				</div>
				{#if orders.find((order) => order.tableNumber === selectedTableNumber)?.items.length === 0}
					<p class="text-gray-500">No items in the order.</p>
				{:else}
					<ul class="divide-y divide-gray-200">
						{#each orders.find((order) => order.tableNumber === selectedTableNumber)?.items as item}
							<li class="flex items-center justify-between py-2">
								<span class="font-semibold">{item.name}</span>
								<span class="text-gray-700">x {item.quantity}</span>
								<span class="font-medium text-green-700">RM {item.totalPrice / 100}</span>
							</li>
						{/each}
					</ul>
					<p class="mt-4 border-t pt-4 text-right text-lg font-bold">
						Total Price: <span class="text-green-700"
							>RM {(orders.find((order) => order.tableNumber === selectedTableNumber)?.totalPrice ||
								0) / 100}</span
						>
					</p>
				{/if}
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
							<h4 class="font-semibold">{main.name}</h4>

							<div class="grid w-full grid-cols-[auto_1fr] items-center justify-start gap-4">
								<span class="flex-1">
									<!-- css like bullet -->
									<span class="rounded-full bg-gray-200 px-2 py-1 text-sm font-medium text-gray-700"
										>RM {main.price / 100}</span
									>
									<span class="mx-2"> x </span>
									<!-- quantity -->
									<!-- bullet css -->
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
									<!-- total item price -->
									<!-- bullet with different color -->
									<span
										class="rounded-full bg-green-200 px-2 py-1 text-sm font-medium text-green-800"
									>
										RM
										{#if orders
											.find((order) => order.tableNumber === selectedTableNumber)
											?.items.find((item) => item.id === main.id)}
											{orders
												.find((order) => order.tableNumber === selectedTableNumber)
												?.items.find((item) => item.id === main.id)?.totalPrice / 100}
										{:else}
											0
										{/if}
									</span>
								</span>
							</div>
						</div>

						<div class="flex w-full flex-col items-center justify-between gap-4">
							<button
								class="rounded bg-green-500 px-3 py-1 text-white hover:bg-green-600"
								onclick={() => addItemToOrder(selectedTableNumber!, main)}
							>
								<!-- svg + -->
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

							<!-- button for - -->
							<button
								class="rounded bg-red-500 px-3 py-1 text-white hover:bg-red-600"
								onclick={() => reduceItemQuantityFromOrder(selectedTableNumber!, main.id)}
							>
								<!-- svg - -->
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
					<!-- if selected sauce then remove button -->

					<button
						class={orders
							.find((order) => order.tableNumber === selectedTableNumber)
							?.items.find((item) => item.id === sauce.id)
							? 'rounded-lg bg-red-600 px-4 py-2 font-semibold text-white hover:bg-red-700'
							: 'rounded-lg bg-gray-300 px-4 py-2 text-black hover:bg-gray-500'}
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
