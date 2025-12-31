<script lang="ts">
	import { Calendar as CalendarPrimitive } from "bits-ui";
	import type { DateValue } from "@internationalized/date";
	import { cn } from "$lib/utils.js";
	import ChevronLeft from "@lucide/svelte/icons/chevron-left";
	import ChevronRight from "@lucide/svelte/icons/chevron-right";
	import { Button } from "$lib/components/ui/button/index.js";

	interface Props {
		value?: DateValue;
		onValueChange?: (value: DateValue | undefined) => void;
		class?: string;
	}

	let { value = $bindable(), onValueChange, class: className }: Props = $props();
</script>

<CalendarPrimitive.Root
	type="single"
	bind:value
	{onValueChange}
	weekdayFormat="short"
	class={cn("bg-background p-3", className)}
>
	{#snippet children({ months, weekdays })}
		<CalendarPrimitive.Header class="relative flex w-full items-center justify-between">
			<CalendarPrimitive.PrevButton>
				{#snippet child({ props })}
					<Button {...props} variant="outline" size="icon" class="size-7">
						<ChevronLeft class="size-4" />
					</Button>
				{/snippet}
			</CalendarPrimitive.PrevButton>
			<CalendarPrimitive.Heading class="text-sm font-medium" />
			<CalendarPrimitive.NextButton>
				{#snippet child({ props })}
					<Button {...props} variant="outline" size="icon" class="size-7">
						<ChevronRight class="size-4" />
					</Button>
				{/snippet}
			</CalendarPrimitive.NextButton>
		</CalendarPrimitive.Header>

		{#each months as month}
			<CalendarPrimitive.Grid class="mt-4 w-full border-collapse">
				<CalendarPrimitive.GridHead>
					<CalendarPrimitive.GridRow class="flex">
						{#each weekdays as weekday}
							<CalendarPrimitive.HeadCell
								class="text-muted-foreground w-8 rounded-md text-center text-xs font-normal"
							>
								{weekday.slice(0, 2)}
							</CalendarPrimitive.HeadCell>
						{/each}
					</CalendarPrimitive.GridRow>
				</CalendarPrimitive.GridHead>
				<CalendarPrimitive.GridBody>
					{#each month.weeks as weekDates}
						<CalendarPrimitive.GridRow class="mt-2 flex w-full">
							{#each weekDates as date}
								<CalendarPrimitive.Cell {date} month={month.value} class="p-0">
									<CalendarPrimitive.Day
										class={cn(
											"hover:bg-accent hover:text-accent-foreground inline-flex size-8 items-center justify-center rounded-md text-sm font-normal transition-colors",
											"focus-visible:ring-ring focus-visible:outline-none focus-visible:ring-1",
											"data-[disabled]:pointer-events-none data-[disabled]:opacity-50",
											"data-[outside-month]:text-muted-foreground/50 data-[outside-month]:pointer-events-none",
											"data-[selected]:bg-primary data-[selected]:text-primary-foreground data-[selected]:font-medium",
											"data-[today]:bg-accent data-[today]:text-accent-foreground"
										)}
									/>
								</CalendarPrimitive.Cell>
							{/each}
						</CalendarPrimitive.GridRow>
					{/each}
				</CalendarPrimitive.GridBody>
			</CalendarPrimitive.Grid>
		{/each}
	{/snippet}
</CalendarPrimitive.Root>
