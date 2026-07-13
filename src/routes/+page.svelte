<script lang="ts">
	import Card from '$lib/components/ui/card/card.svelte';
	import Input from '$lib/components/ui/input/input.svelte';
	import Label from '$lib/components/ui/label/label.svelte';
	import {
		createCollection,
		localStorageCollectionOptions,
		useLiveQuery
	} from '@tanstack/svelte-db';

	import { createHotkey } from '@tanstack/svelte-hotkeys';

	let balanceInput = $state<HTMLInputElement | null>(null);
	let entryInput = $state<HTMLInputElement | null>(null);
	let stopInput = $state<HTMLInputElement | null>(null);
	let exitInput = $state<HTMLInputElement | null>(null);
	let rMultipleInput = $state<HTMLInputElement | null>(null);
	let maxSharesInput = $state<HTMLInputElement | null>(null);

	createHotkey(
		'E',
		() => {
			entryInput!.select();
		},
		{ ignoreInputs: false }
	);
	createHotkey('B', () => balanceInput!.select(), { ignoreInputs: false });
	createHotkey('S', () => stopInput!.select(), { ignoreInputs: false });
	createHotkey('X', () => exitInput!.select(), { ignoreInputs: false });
	createHotkey('R', () => rMultipleInput!.select(), { ignoreInputs: false });
	createHotkey('Control+C', () => copyMaxShares(), { ignoreInputs: false });

	interface AppSettings {
		id: 'settings';
		balance: number;
		entry: number;
		stop: number;
		exit: number;
		rMultiple: number;
	}

	type PriceField = 'balance' | 'entry' | 'stop' | 'exit';
	type NumberField = 'rMultiple';

	const defaultSettings: AppSettings = {
		id: 'settings',
		balance: 0,
		entry: 0,
		stop: 0,
		exit: 0,
		rMultiple: 0
	};

	const settingsCollection = createCollection(
		localStorageCollectionOptions<AppSettings>({
			id: 'settings',
			storageKey: 'app-settings',
			getKey: (item) => item.id
		})
	);

	const settingsQuery = useLiveQuery((q) => q.from({ settings: settingsCollection }));

	const settings = $derived(settingsQuery.data[0]);

	$effect(() => {
		if (settingsQuery.isReady && settingsQuery.data.length === 0) {
			settingsCollection.insert(defaultSettings);
		}
	});

	function formatCents(cents: number): string {
		return (cents / 100).toFixed(2);
	}

	function parseDigits(value: string): number {
		const digits = value.replace(/\D/g, '');
		return digits ? Number.parseInt(digits, 10) : 0;
	}

	function parseNumber(value: string): number {
		const parsed = Number.parseFloat(value);
		return Number.isFinite(parsed) ? parsed : 0;
	}

	function updatePrice(field: PriceField, event: Event): void {
		const input = event.currentTarget as HTMLInputElement;
		const value = parseDigits(input.value);

		settingsCollection.update('settings', (draft) => {
			draft[field] = value;
		});
	}

	function updateNumber(field: NumberField, event: Event): void {
		const input = event.currentTarget as HTMLInputElement;
		const value = parseNumber(input.value);

		settingsCollection.update('settings', (draft) => {
			draft[field] = value;
		});
	}

	async function copyMaxShares(): Promise<void> {
		const value = maxSharesInput?.value ?? '';

		if (!value) {
			return;
		}

		try {
			await navigator.clipboard.writeText(value);
		} catch {
			maxSharesInput?.select();
		}
	}
</script>

<Card class="w-lg bg-card-fg p-8">
	{#if settingsQuery.isLoading || !settings}
		<p>Loading settings…</p>
	{:else}
		<div>
			<Label for="balance" class="mb-1">Balance</Label>
			<Input
				bind:ref={balanceInput}
				id="balance"
				type="text"
				inputmode="numeric"
				autocomplete="off"
				value={formatCents(settings.balance ?? 0)}
				oninput={(event) => updatePrice('balance', event)}
			/>
		</div>

		<div>
			<Label for="entry" class="mb-1">Entry</Label>
			<Input
				bind:ref={entryInput}
				id="entry"
				type="text"
				inputmode="numeric"
				autocomplete="off"
				autofocus
				value={formatCents(settings.entry)}
				oninput={(event) => updatePrice('entry', event)}
			/>
		</div>

		<div>
			<Label for="stop" class="mb-1">Stop</Label>
			<Input
				bind:ref={stopInput}
				id="stop"
				type="text"
				inputmode="numeric"
				autocomplete="off"
				value={formatCents(settings.stop)}
				oninput={(event) => updatePrice('stop', event)}
			/>
		</div>

		<div>
			<Label for="exit" class="mb-1">Exit</Label>
			<Input
				bind:ref={exitInput}
				id="exit"
				type="text"
				inputmode="numeric"
				autocomplete="off"
				value={formatCents(settings.exit)}
				oninput={(event) => updatePrice('exit', event)}
			/>
		</div>

		<div>
			<Label for="r-multiple" class="mb-1">R Multiple</Label>
			<Input
				bind:ref={rMultipleInput}
				id="r-multiple"
				type="number"
				inputmode="decimal"
				autocomplete="off"
				step="0.1"
				min="0"
				value={settings.rMultiple ?? 0}
				oninput={(event) => updateNumber('rMultiple', event)}
			/>
		</div>

		<div>
			<Label for="maxshares" class="mb-1">Max Shares</Label>
			<Input bind:ref={maxSharesInput} id="maxshares" type="number" readonly value={67} />
		</div>
	{/if}
</Card>
