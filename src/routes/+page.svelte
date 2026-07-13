<script lang="ts">
	import Card from '$lib/components/ui/card/card.svelte';
	import Input from '$lib/components/ui/input/input.svelte';
	import Label from '$lib/components/ui/label/label.svelte';

	let entry = $state(0);
	let stop = $state(0);
	let exit = $state(0);

	function formatCents(cents: number) {
		return (cents / 100).toFixed(2);
	}

	function parseDigits(value: string): number {
		const digits = value.replace(/\D/g, '');
		return digits ? Number.parseInt(digits, 10) : 0;
	}
</script>

<Card class="w-lg p-8 bg-card-fg">
	<div>
		<Label for="entry" class="mb-1">Entry</Label>
		<Input
			id="entry"
			type="text"
			inputmode="numeric"
			autocomplete="off"
			value={formatCents(entry)}
			oninput={(e) => {
				entry = parseDigits(e.currentTarget.value);
			}}
		/>
	</div>
	<div>
		<Label for="stop" class="mb-1">Stop</Label>
		<Input
			id="stop"
			type="text"
			inputmode="numeric"
			autocomplete="off"
			value={formatCents(stop)}
			oninput={(e) => {
				stop = parseDigits(e.currentTarget.value);
			}}
		/>
	</div>
	<div>
		<Label for="exit" class="mb-1">Exit</Label>
		<Input
			id="exit"
			type="text"
			inputmode="numeric"
			autocomplete="off"
			value={formatCents(exit)}
			oninput={(e) => {
				exit = parseDigits(e.currentTarget.value);
			}}
		/>
	</div>
	<output>
		<Label for="maxshares" class="mb-1">Max Shares</Label>
		<Input id="maxshares" type="number" readonly value={entry + stop + exit} />
	</output>
</Card>
