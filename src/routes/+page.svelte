<script lang="ts">
	import Card from '$lib/components/ui/card/card.svelte';
	import * as InputGroup from '$lib/components/ui/input-group';
	import Input from '$lib/components/ui/input/input.svelte';
	import Label from '$lib/components/ui/label/label.svelte';
	import {
		createCollection,
		localStorageCollectionOptions,
		useLiveQuery
	} from '@tanstack/svelte-db';

	import CheckIcon from '@lucide/svelte/icons/check';
	import ClipboardIcon from '@lucide/svelte/icons/clipboard';
	import { createHotkey } from '@tanstack/svelte-hotkeys';

	let balanceInput = $state<HTMLInputElement | null>(null);
	let entryInput = $state<HTMLInputElement | null>(null);
	let stopInput = $state<HTMLInputElement | null>(null);
	let exitInput = $state<HTMLInputElement | null>(null);
	let rMultipleInput = $state<HTMLInputElement | null>(null);
	let maxSharesInput = $state<HTMLInputElement | null>(null);
	let didCopyMaxShares = $state(false);
	let copyResetTimeout: ReturnType<typeof setTimeout> | undefined;

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
		rMultiple: 0.1
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
	const maxShares = $derived.by(() => {
		if (!settings) {
			return 0;
		}

		const riskPerShare = settings.entry - settings.stop;
		const riskAmount = (settings.balance / 100) * (settings.rMultiple ?? 0);

		if (riskPerShare <= 0 || riskAmount <= 0) {
			return 0;
		}

		const shares = riskAmount / (riskPerShare / 100);
		return Number.isFinite(shares) ? Math.floor(shares) : 0;
	});

	$effect(() => {
		if (settingsQuery.isReady && settingsQuery.data.length === 0) {
			settingsCollection.insert(defaultSettings);
		}
	});

	function formatCents(cents: number): string {
		return (cents / 100).toFixed(2);
	}

	function formatDecimal(value: number): string {
		if (!Number.isFinite(value)) {
			return '0.00';
		}

		const formatted = String(value);
		const decimalIndex = formatted.indexOf('.');

		if (decimalIndex === -1) {
			return `${formatted}.00`;
		}

		const decimalPlaces = formatted.length - decimalIndex - 1;
		return decimalPlaces < 2 ? value.toFixed(2) : formatted;
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

	function updateStop(event: Event): void {
		const input = event.currentTarget as HTMLInputElement;
		const stop = parseDigits(input.value);

		settingsCollection.update('settings', (draft) => {
			draft.stop = stop;
			draft.exit = draft.entry + (draft.entry - stop) * 4;
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
			didCopyMaxShares = true;
			clearTimeout(copyResetTimeout);
			copyResetTimeout = setTimeout(() => {
				didCopyMaxShares = false;
			}, 1200);
		} catch {
			maxSharesInput?.select();
		}
	}
</script>

<h1 class="sr-only">Position size calculator</h1>

<Card class="w-full max-w-lg bg-card-fg p-4 sm:p-8">
	{#if settingsQuery.isLoading || !settings}
		<p aria-live="polite">Loading settings…</p>
	{:else}
		<div class="space-y-1.5">
			<Label for="balance" class="mb-1">Balance</Label>
			<Input
				bind:ref={balanceInput}
				id="balance"
				type="text"
				inputmode="numeric"
				autocomplete="off"
				class="h-11 sm:h-9"
				value={formatCents(settings.balance ?? 0)}
				oninput={(event) => updatePrice('balance', event)}
			/>
		</div>

		<div class="space-y-1.5">
			<Label for="entry" class="mb-1">Entry</Label>
			<Input
				bind:ref={entryInput}
				id="entry"
				type="text"
				inputmode="numeric"
				autocomplete="off"
				class="h-11 sm:h-9"
				value={formatCents(settings.entry)}
				oninput={(event) => updatePrice('entry', event)}
			/>
		</div>

		<div class="space-y-1.5">
			<Label for="stop" class="mb-1">Stop</Label>
			<Input
				bind:ref={stopInput}
				id="stop"
				type="text"
				inputmode="numeric"
				autocomplete="off"
				class="h-11 sm:h-9"
				value={formatCents(settings.stop)}
				oninput={updateStop}
			/>
		</div>

		<div class="space-y-1.5">
			<Label for="exit" class="mb-1">Exit</Label>
			<Input
				bind:ref={exitInput}
				id="exit"
				type="text"
				inputmode="numeric"
				autocomplete="off"
				class="h-11 sm:h-9"
				value={formatCents(settings.exit)}
				oninput={(event) => updatePrice('exit', event)}
			/>
		</div>

		<div class="space-y-1.5">
			<Label for="r-multiple" class="mb-1">R Multiple</Label>
			<Input
				bind:ref={rMultipleInput}
				id="r-multiple"
				type="number"
				inputmode="decimal"
				autocomplete="off"
				step="0.1"
				min="0"
				class="h-11 sm:h-9"
				value={formatDecimal(settings.rMultiple ?? 0.1)}
				oninput={(event) => updateNumber('rMultiple', event)}
			/>
		</div>
	{/if}
</Card>

<Card class="w-full max-w-lg bg-card-fg p-4 sm:p-8">
	<div class="space-y-1.5">
		<Label for="maxshares" class="mb-1">Max Shares</Label>
		<InputGroup.Root>
			<InputGroup.Input
				bind:ref={maxSharesInput}
				id="maxshares"
				type="number"
				readonly
				class="h-11 sm:h-9"
				value={maxShares}
			/>
			<InputGroup.Button
				aria-label="Copy max shares"
				size="icon-sm"
				data-align="inline-end"
				onclick={copyMaxShares}
				class="size-11 cursor-pointer sm:size-8"
			>
				{#if didCopyMaxShares}
					<CheckIcon class="text-green-600" />
				{:else}
					<ClipboardIcon />
				{/if}
			</InputGroup.Button>
		</InputGroup.Root>
	</div>
</Card>
