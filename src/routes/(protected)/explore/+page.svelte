<script lang="ts">
    import { getContext } from "svelte";
    import ListItem from "$lib/components/list-item.svelte";
    import { Button } from "$lib/components/ui/button/index.js";
    import { Skeleton } from "$lib/components/ui/skeleton/index.js";
    import * as Popover from "$lib/components/ui/popover/index.js";
    import { Badge } from "$lib/components/ui/badge/index.js";
    import * as Calendar from "$lib/components/ui/calendar/index.js";
    import { Input } from "$lib/components/ui/input/index.js";
    import Filter from "@lucide/svelte/icons/filter";
    import X from "@lucide/svelte/icons/x";
    import CalendarIcon from "@lucide/svelte/icons/calendar";
    import { CalendarDate, type DateValue } from "@internationalized/date";

    let { data } = $props();

    const searchStore: any = getContext("searchStore");
    let loadMoreTrigger = $state<HTMLDivElement | null>(null);
    let lastQuery = $state("");

    // Genre mapping for display
    const GENRES: Record<number, string> = {
        28: "Action",
        12: "Adventure",
        16: "Animation",
        35: "Comedy",
        80: "Crime",
        99: "Documentary",
        18: "Drama",
        10751: "Family",
        14: "Fantasy",
        36: "History",
        27: "Horror",
        10402: "Music",
        9648: "Mystery",
        10749: "Romance",
        878: "Sci-Fi",
        53: "Thriller",
        10752: "War",
        37: "Western"
    };

    // Generate year options (1900 to current year + 5)
    const currentYear = new Date().getFullYear();
    const yearOptions = Array.from(
        { length: currentYear - 1900 + 6 },
        (_, i) => currentYear + 5 - i
    );

    // Derived values from data
    const currentQuery = $derived(data.form?.data?.query || "");
    const currentType = $derived(data.form?.data?.type || "both");
    const activeFilterCount = $derived(
        (searchStore.filters.dateFrom || searchStore.filters.dateTo ? 1 : 0) +
            (searchStore.filters.genres.length > 0 ? 1 : 0) +
            (searchStore.filters.language ? 1 : 0)
    );

    // Date picker state
    let dateFromOpen = $state(false);
    let dateToOpen = $state(false);

    // Text input state for dates (synced with store)
    let dateFromInput = $state(searchStore.filters.dateFrom || "");
    let dateToInput = $state(searchStore.filters.dateTo || "");

    // Calendar value state (derived from store)
    let dateFromValue = $state<DateValue | undefined>(undefined);
    let dateToValue = $state<DateValue | undefined>(undefined);

    // Sync calendar values when store changes
    $effect(() => {
        const from = searchStore.filters.dateFrom;
        if (from) {
            const [y, m, d] = from.split("-").map(Number);
            dateFromValue = new CalendarDate(y, m, d);
            dateFromInput = from;
        } else {
            dateFromValue = undefined;
            dateFromInput = "";
        }
    });

    $effect(() => {
        const to = searchStore.filters.dateTo;
        if (to) {
            const [y, m, d] = to.split("-").map(Number);
            dateToValue = new CalendarDate(y, m, d);
            dateToInput = to;
        } else {
            dateToValue = undefined;
            dateToInput = "";
        }
    });

    // Derived display values from store
    const dateFromDisplay = $derived(
        searchStore.filters.dateFrom
            ? new Date(searchStore.filters.dateFrom + "T00:00:00").toLocaleDateString("en-US", {
                  month: "short",
                  day: "numeric",
                  year: "numeric"
              })
            : null
    );

    const dateToDisplay = $derived(
        searchStore.filters.dateTo
            ? new Date(searchStore.filters.dateTo + "T00:00:00").toLocaleDateString("en-US", {
                  month: "short",
                  day: "numeric",
                  year: "numeric"
              })
            : null
    );

    function isValidDate(dateStr: string): boolean {
        if (!/^\d{4}-\d{2}-\d{2}$/.test(dateStr)) return false;
        const date = new Date(dateStr + "T00:00:00");
        return !isNaN(date.getTime());
    }

    function handleDateFromInput(e: Event) {
        const value = (e.target as HTMLInputElement).value;
        dateFromInput = value;
        if (value === "") {
            searchStore.setDateRange(null, searchStore.filters.dateTo);
        } else if (isValidDate(value)) {
            searchStore.setDateRange(value, searchStore.filters.dateTo);
        }
    }

    function handleDateToInput(e: Event) {
        const value = (e.target as HTMLInputElement).value;
        dateToInput = value;
        if (value === "") {
            searchStore.setDateRange(searchStore.filters.dateFrom, null);
        } else if (isValidDate(value)) {
            searchStore.setDateRange(searchStore.filters.dateFrom, value);
        }
    }

    // Track previous values to detect user calendar selections
    let prevDateFromValue: DateValue | undefined = undefined;
    let prevDateToValue: DateValue | undefined = undefined;

    // Watch for calendar selection changes (only when popover is open = user interaction)
    $effect(() => {
        if (dateFromOpen && dateFromValue && dateFromValue !== prevDateFromValue) {
            const iso = `${dateFromValue.year}-${String(dateFromValue.month).padStart(2, "0")}-${String(dateFromValue.day).padStart(2, "0")}`;
            if (iso !== searchStore.filters.dateFrom) {
                searchStore.setDateRange(iso, searchStore.filters.dateTo);
                dateFromOpen = false;
            }
        }
        prevDateFromValue = dateFromValue;
    });

    $effect(() => {
        if (dateToOpen && dateToValue && dateToValue !== prevDateToValue) {
            const iso = `${dateToValue.year}-${String(dateToValue.month).padStart(2, "0")}-${String(dateToValue.day).padStart(2, "0")}`;
            if (iso !== searchStore.filters.dateTo) {
                searchStore.setDateRange(searchStore.filters.dateFrom, iso);
                dateToOpen = false;
            }
        }
        prevDateToValue = dateToValue;
    });

    function clearDateFrom() {
        searchStore.setDateRange(null, searchStore.filters.dateTo);
    }

    function clearDateTo() {
        searchStore.setDateRange(searchStore.filters.dateFrom, null);
    }

    // Handle query changes
    $effect(() => {
        if (currentQuery !== lastQuery) {
            lastQuery = currentQuery;

            if (currentQuery) {
                searchStore.setSearch(currentQuery, data.parsed);
                searchStore.setMediaType(currentType);
                searchStore.searchDebounced();
            } else {
                searchStore.clear();
            }
        }
    });

    // Setup intersection observer for infinite scroll
    $effect(() => {
        if (!loadMoreTrigger) return;

        const observer = new IntersectionObserver(
            (entries) => {
                if (entries[0].isIntersecting && searchStore.hasMore && !searchStore.loading) {
                    searchStore.loadMore();
                }
            },
            { threshold: 0.1 }
        );

        observer.observe(loadMoreTrigger);

        return () => {
            observer.disconnect();
            searchStore.cancelPendingRequests();
        };
    });
</script>

<svelte:head>
    <title>Search Results - Riven</title>
</svelte:head>

<div class="mt-14 flex flex-col gap-6 p-6 md:p-8 md:px-16">
    <div class="flex flex-col gap-4">
        <div class="flex flex-wrap items-center justify-between gap-4">
            <div class="flex flex-col gap-1">
                <h1 class="text-2xl font-bold md:text-3xl lg:text-4xl">Search Results</h1>
                {#if searchStore.rawSearchString}
                    <p class="text-muted-foreground text-sm">
                        Searching for: <span class="font-mono">{searchStore.rawSearchString}</span>
                    </p>
                {/if}
                {#if searchStore.totalResults > 0 && !searchStore.loading}
                    <p class="text-muted-foreground text-sm">
                        {#if searchStore.hasActiveFilters}
                            Showing {searchStore.results.length} of {searchStore.unfilteredResultsCount}
                            loaded
                        {:else}
                            Found {searchStore.totalResults} results
                        {/if}
                    </p>
                {/if}
            </div>

            <div class="flex flex-wrap items-center gap-2">
                <!-- Media Type Buttons -->
                <div class="flex gap-1">
                    <Button
                        variant={searchStore.mediaType === "both" ? "default" : "outline"}
                        size="sm"
                        onclick={() => searchStore.setMediaType("both")}>
                        All
                    </Button>
                    <Button
                        variant={searchStore.mediaType === "movie" ? "default" : "outline"}
                        size="sm"
                        onclick={() => searchStore.setMediaType("movie")}>
                        Movies
                    </Button>
                    <Button
                        variant={searchStore.mediaType === "tv" ? "default" : "outline"}
                        size="sm"
                        onclick={() => searchStore.setMediaType("tv")}>
                        TV Shows
                    </Button>
                </div>

                <!-- Filter Popover -->
                <Popover.Root>
                    <Popover.Trigger>
                        <Button variant="outline" size="sm" class="gap-2">
                            <Filter class="size-4" />
                            Filters
                            {#if activeFilterCount > 0}
                                <Badge variant="secondary" class="ml-1 size-5 p-0 text-xs">
                                    {activeFilterCount}
                                </Badge>
                            {/if}
                        </Button>
                    </Popover.Trigger>
                    <Popover.Content class="w-80">
                        <div class="flex flex-col gap-4">
                            <div class="flex items-center justify-between">
                                <h4 class="font-semibold">Filters</h4>
                                {#if searchStore.hasActiveFilters}
                                    <Button
                                        variant="ghost"
                                        size="sm"
                                        class="h-auto p-1 text-xs"
                                        onclick={() => searchStore.resetFilters()}>
                                        Clear all
                                    </Button>
                                {/if}
                            </div>

                            <!-- Date Filter -->
                            <div class="flex flex-col gap-3">
                                <span class="text-sm font-medium">Release / Air Date</span>

                                <div class="flex items-center gap-2">
                                    <div class="flex flex-1 flex-col gap-1">
                                        <div class="flex items-center justify-between">
                                            <span class="text-muted-foreground text-xs">From</span>
                                            {#if searchStore.filters.dateFrom}
                                                <button
                                                    class="text-muted-foreground hover:text-foreground text-xs"
                                                    onclick={clearDateFrom}>
                                                    Clear
                                                </button>
                                            {/if}
                                        </div>
                                        <div class="flex gap-1">
                                            <Input
                                                type="text"
                                                placeholder="YYYY-MM-DD"
                                                value={dateFromInput}
                                                oninput={handleDateFromInput}
                                                class="h-8 flex-1 font-mono text-xs" />
                                            <Popover.Root bind:open={dateFromOpen}>
                                                <Popover.Trigger>
                                                    <Button
                                                        variant="outline"
                                                        size="sm"
                                                        class="h-8 w-8 p-0">
                                                        <CalendarIcon class="size-4" />
                                                    </Button>
                                                </Popover.Trigger>
                                                <Popover.Content class="w-auto p-0" align="end">
                                                    <Calendar.Calendar
                                                        type="single"
                                                        bind:value={dateFromValue}
                                                        captionLayout="dropdown"
                                                        years={yearOptions} />
                                                </Popover.Content>
                                            </Popover.Root>
                                        </div>
                                    </div>
                                    <div class="flex flex-1 flex-col gap-1">
                                        <div class="flex items-center justify-between">
                                            <span class="text-muted-foreground text-xs">To</span>
                                            {#if searchStore.filters.dateTo}
                                                <button
                                                    class="text-muted-foreground hover:text-foreground text-xs"
                                                    onclick={clearDateTo}>
                                                    Clear
                                                </button>
                                            {/if}
                                        </div>
                                        <div class="flex gap-1">
                                            <Input
                                                type="text"
                                                placeholder="YYYY-MM-DD"
                                                value={dateToInput}
                                                oninput={handleDateToInput}
                                                class="h-8 flex-1 font-mono text-xs" />
                                            <Popover.Root bind:open={dateToOpen}>
                                                <Popover.Trigger>
                                                    <Button
                                                        variant="outline"
                                                        size="sm"
                                                        class="h-8 w-8 p-0">
                                                        <CalendarIcon class="size-4" />
                                                    </Button>
                                                </Popover.Trigger>
                                                <Popover.Content class="w-auto p-0" align="end">
                                                    <Calendar.Calendar
                                                        type="single"
                                                        bind:value={dateToValue}
                                                        captionLayout="dropdown"
                                                        years={yearOptions} />
                                                </Popover.Content>
                                            </Popover.Root>
                                        </div>
                                    </div>
                                </div>
                            </div>

                            <!-- Genre Filter -->
                            <div class="flex flex-col gap-2">
                                <label class="text-sm font-medium">Genres</label>
                                <div class="flex flex-wrap gap-1">
                                    {#each Object.entries(GENRES) as [id, name]}
                                        {@const genreId = parseInt(id)}
                                        {@const isSelected =
                                            searchStore.filters.genres.includes(genreId)}
                                        <Button
                                            variant={isSelected ? "default" : "outline"}
                                            size="sm"
                                            class="h-7 px-2 text-xs"
                                            onclick={() => searchStore.toggleGenre(genreId)}>
                                            {name}
                                        </Button>
                                    {/each}
                                </div>
                            </div>
                        </div>
                    </Popover.Content>
                </Popover.Root>
            </div>
        </div>

        <!-- Active Filters Display -->
        {#if searchStore.hasActiveFilters}
            <div class="flex flex-wrap items-center gap-2">
                <span class="text-muted-foreground text-sm">Active filters:</span>
                {#if dateFromDisplay || dateToDisplay}
                    <Badge variant="secondary" class="gap-1">
                        <CalendarIcon class="size-3" />
                        {#if dateFromDisplay && dateToDisplay}
                            {dateFromDisplay} - {dateToDisplay}
                        {:else if dateFromDisplay}
                            From {dateFromDisplay}
                        {:else}
                            Until {dateToDisplay}
                        {/if}
                        <button
                            class="hover:text-destructive ml-1"
                            onclick={() => searchStore.setDateRange(null, null)}>
                            <X class="size-3" />
                        </button>
                    </Badge>
                {/if}
                {#each searchStore.filters.genres as genreId}
                    <Badge variant="secondary" class="gap-1">
                        {GENRES[genreId] || genreId}
                        <button
                            class="hover:text-destructive ml-1"
                            onclick={() => searchStore.toggleGenre(genreId)}>
                            <X class="size-3" />
                        </button>
                    </Badge>
                {/each}
            </div>
        {/if}
    </div>

    {#if searchStore.warnings && searchStore.warnings.length > 0}
        <div
            class="rounded-lg border border-yellow-500 bg-yellow-500/10 p-4 text-yellow-600 dark:text-yellow-500">
            <p class="font-semibold">Warnings</p>
            <ul class="mt-1 list-disc pl-5 text-sm">
                {#each searchStore.warnings as warning}
                    <li>{warning}</li>
                {/each}
            </ul>
        </div>
    {/if}

    {#if searchStore.error}
        <div class="rounded-lg border border-red-500 bg-red-500/10 p-4 text-red-500">
            <p class="font-semibold">Error</p>
            <p class="text-sm">{searchStore.error}</p>
        </div>
    {/if}

    {#if !searchStore.rawSearchString}
        <div class="flex flex-col items-center justify-center gap-4 py-16">
            <p class="text-muted-foreground">
                Use the search bar above to discover movies and TV shows
            </p>
            <div class="text-muted-foreground text-sm">
                <p class="mb-2 font-semibold">Examples:</p>
                <ul class="list-disc space-y-1 pl-5 font-mono text-xs">
                    <li>inception - text search only</li>
                    <li>inception y:2024 g:sci-fi - hybrid search with filters</li>
                    <li>y:2024 g:action va:7 - pure filtering (no text)</li>
                    <li>test year:2025 eg:fantasy va:7</li>
                </ul>
            </div>
        </div>
    {:else if Array.isArray(searchStore.results) && searchStore.results.length > 0}
        <div class="flex flex-wrap items-center gap-4">
            {#each searchStore.results as item (item.id)}
                <ListItem data={item} indexer={item.indexer} type={item.media_type} />
            {/each}
            {#if searchStore.loading}
                {#each Array(6) as _}
                    <div class="w-full">
                        <Skeleton class="aspect-2/3 w-full rounded-sm" />
                        <Skeleton class="mt-2 h-4 w-full" />
                        <div class="mt-1 flex items-center justify-between">
                            <Skeleton class="h-4 w-12 rounded-full" />
                            <Skeleton class="h-4 w-12 rounded-full" />
                        </div>
                    </div>
                {/each}
            {/if}
        </div>
    {:else if searchStore.loading}
        <div
            class="grid grid-cols-2 gap-4 sm:grid-cols-3 md:grid-cols-4 lg:grid-cols-5 xl:grid-cols-6">
            {#each Array(12) as _}
                <div>
                    <Skeleton class="aspect-2/3 w-full rounded-sm" />
                    <Skeleton class="mt-2 h-4 w-full" />
                    <div class="mt-1 flex items-center justify-between">
                        <Skeleton class="h-4 w-12 rounded-full" />
                        <Skeleton class="h-4 w-12 rounded-full" />
                    </div>
                </div>
            {/each}
        </div>
    {:else if searchStore.hasActiveFilters}
        <div class="flex flex-col items-center justify-center gap-2 py-16">
            <p class="text-muted-foreground">No results match your filters</p>
            <Button variant="outline" size="sm" onclick={() => searchStore.resetFilters()}>
                Clear filters
            </Button>
        </div>
    {:else}
        <div class="flex flex-col items-center justify-center gap-2 py-16">
            <p class="text-muted-foreground">No results found</p>
            <p class="text-muted-foreground text-sm">Try adjusting your search query</p>
        </div>
    {/if}

    <div bind:this={loadMoreTrigger}></div>
</div>
