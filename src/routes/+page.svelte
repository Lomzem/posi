<script lang="ts">
	import Card from '$lib/components/ui/card/card.svelte';
	import Input from '$lib/components/ui/input/input.svelte';
	import Label from '$lib/components/ui/label/label.svelte';
	import {
		createCollection,
		localStorageCollectionOptions,
		useLiveQuery
	} from '@tanstack/svelte-db';

	interface AppSettings {
		id: 'settings';
		entry: number;
		stop: number;
		exit: number;
	}

	type PriceField = 'entry' | 'stop' | 'exit';

	const defaultSettings: AppSettings = {
		id: 'settings',
		entry: 0,
		stop: 0,
		exit: 0
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

	function updatePrice(field: PriceField, event: Event): void {
		const input = event.currentTarget as HTMLInputElement;
		const value = parseDigits(input.value);

		settingsCollection.update('settings', (draft) => {
			draft[field] = value;
		});
	}
</script>

<Card class="w-lg bg-card-fg p-8">
	{#if settingsQuery.isLoading || !settings}
		<p>Loading settings…</p>
	{:else}
		<div>
			<Label for="entry" class="mb-1">Entry</Label>
			<Input
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
				id="exit"
				type="text"
				inputmode="numeric"
				autocomplete="off"
				value={formatCents(settings.exit)}
				oninput={(event) => updatePrice('exit', event)}
			/>
		</div>

		<div>
			<Label for="maxshares" class="mb-1">Max Shares</Label>
			<Input id="maxshares" type="number" readonly value={0} />
		</div>
	{/if}
</Card>
