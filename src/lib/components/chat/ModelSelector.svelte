<script lang="ts">
    import { models, showSettings, settings, user, mobile, config } from '$lib/stores';
    import { onMount, tick, getContext } from 'svelte';
    import { toast } from 'svelte-sonner';
    import Selector from './ModelSelector/Selector.svelte';
    import Tooltip from '../common/Tooltip.svelte';
    import { setDefaultModels } from '$lib/apis/configs';
    import { updateUserSettings } from '$lib/apis/users';

    const i18n = getContext('i18n');

    export let selectedModels = [''];
    export let disabled = false;
    export let showSetDefault = true;

    // Initialize default model from settings or selectedModels
    let defaultModel = $settings?.models?.[0] || selectedModels[0] || '';
    
    // Initialize comparison models array
    let comparisonModels: string[] = $settings?.models?.slice(1) || [];

    // Track if the dropdown should be shown
	let showDropdown = false;
	let dropdownDisplayText = $i18n.t('Select models to compare');

    $: dropdownDisplayText = comparisonModels.length === 0 
        ? $i18n.t('Select model to compare')  // Use i18n if you want translation
        : "";
		
	// Add click outside handler to close dropdown
	function handleClickOutside(event: MouseEvent) {
		const target = event.target as HTMLElement;
		if (!target.closest('.comparison-selector')) {
			showDropdown = false;
		}
	}

	onMount(() => {
		// Add click outside listener
		document.addEventListener('click', handleClickOutside);
		
		if ($settings?.models?.length > 0) {
			defaultModel = $settings.models[0];
			comparisonModels = [];
		}

		console.log("comparisonModels", $settings?.models )

		return () => {
			// Clean up listener
			document.removeEventListener('click', handleClickOutside);
		};
	});
    // When default model changes, remove it from comparison models if present
    $: if (defaultModel) {
        comparisonModels = comparisonModels.filter(model => model !== defaultModel);
    }

    const saveDefaultModel = async () => {
        if (!defaultModel) {
            toast.error($i18n.t('Choose a default model before saving...'));
            return;
        }
        settings.set({ ...$settings, models: [defaultModel, ...comparisonModels] });
        await updateUserSettings(localStorage.token, { ui: $settings });
        toast.success($i18n.t('Default model updated'));
    };

    // Update selectedModels when either default or comparison models change
	$: {
    selectedModels = [defaultModel, ...comparisonModels].filter(model => model !== '');
	}
    const removeComparisonModel = (modelId: string) => {
        comparisonModels = comparisonModels.filter(id => id !== modelId);
    };

	const addComparisonModel = (modelId: string) => {
    if (!comparisonModels.includes(modelId)) {
        comparisonModels = [...comparisonModels, modelId];
    }
    showDropdown = false;
};
    const resetComparison = () => {
		comparisonModels = [];
		showDropdown = false;
    };

    // Available models for comparison (excluding default and already selected)
    $: availableModels = $models.filter(model => 
        model.id !== defaultModel && !comparisonModels.includes(model.id) && model.id
    ).sort((a, b) => a.name.localeCompare(b.name));
</script>

<div class="flex flex-col space-y-4">
    <!-- Default Model Selection -->
    <div class="flex items-center space-x-3">
        <div class="model-label">Default model:</div>
        <div class="flex flex-col items-start">
            <div class="overflow-hidden w-full">
                <div class="mr-1 max-w-full">
                    <Selector
                        id="default-model"
                        placeholder={$i18n.t('Select default model')}
                        items={$models.map((model) => ({
                            value: model.id,
                            label: model.name,
                            model: model
                        }))}
                        showTemporaryChatControl={$user.role === 'user'
                            ? ($config?.permissions?.chat?.temporary ?? true)
                            : true}
                        bind:value={defaultModel}
                    />
                </div>
            </div>
            {#if showSetDefault}
                <div class="text-left mt-1 text-[0.9rem] text-gray-500 font-primary">
                    <button on:click={saveDefaultModel}>Set as default</button>
                </div>
            {/if}
        </div>
    </div>

    <!-- Comparison Models Selection -->
	<div class="flex items-center space-x-3">
		<div class="model-label">More model(s):</div>
		<div class="flex flex-col items-start">
			<div class="overflow-visible w-full max-w-fit comparison-selector">
				<div class="mr-1 max-w-full">
					<div 
						class="min-h-[40px] w-[200px] p-2 border rounded-3xl bg-gray-50 cursor-pointer flex items-center justify-between"
						on:mousedown|preventDefault={() => {
							showDropdown = !showDropdown;
						}}
						role="button"
						tabindex="0"
						style="outline: none;" 
					>
						<div class="flex flex-wrap gap-2 items-center flex-1">
							{#if !comparisonModels.length}
                            	<span class="text-gray-500">Activate one or more model(s)</span>
							{:else}
								<div class="flex flex-wrap gap-2">
									{#each comparisonModels as modelId}
										{#if $models.find(m => m.id === modelId)}
											<div class="inline-flex items-center bg-gray-100 rounded-full px-3 py-1 text-sm group">
												<span class="text-white">
													{$models.find(m => m.id === modelId)?.name}
												</span>
												<button
													class="ml-2 text-white hover:text-white"
													on:mousedown|preventDefault|stopPropagation={() => removeComparisonModel(modelId)}
												>
													<svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" viewBox="0 0 20 20" fill="currentColor">
														<path fill-rule="evenodd" d="M4.293 4.293a1 1 0 011.414 0L10 8.586l4.293-4.293a1 1 0 111.414 1.414L11.414 10l4.293 4.293a1 1 0 01-1.414 1.414L10 11.414l-4.293 4.293a1 1 0 01-1.414-1.414L8.586 10 4.293 5.707a1 1 0 010-1.414z" clip-rule="evenodd" />
													</svg>
												</button>
											</div>
										{/if}
									{/each}
								</div>
							{/if}
						</div>
						<div class="flex-none ml-2">
							<svg class="h-5 w-5 text-gray-100" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor">
								<path fill-rule="evenodd" d="M5.293 7.293a1 1 0 011.414 0L10 10.586l3.293-3.293a1 1 0 111.414 1.414l-4 4a1 1 0 01-1.414 0l-4-4a1 1 0 010-1.414z" clip-rule="evenodd" />
							</svg>
						</div>
					</div>
	
					{#if showDropdown}
						<div 
							class="fixed inset-0 z-30"
							on:mousedown|preventDefault={() => showDropdown = false}
						/>
						<div 
							class="absolute z-50 w-[300px] mt-1 bg-white border rounded-md shadow-lg"
							style="max-height: 240px; overflow-y: auto;"
						>
							{#each availableModels as model}
								<div
									class="px-4 py-2 hover:bg-gray-100 hover:text-white cursor-pointer dark:hover:bg-gray-750 dark:text-gray-100"
									on:mousedown|preventDefault={() => {
										addComparisonModel(model.id);
									}}
								>
									{model.name}
								</div>
							{/each}
						</div>
					{/if}
				</div>
			</div>
	
			{#if comparisonModels.length > 0}
				<div class="mt-2">
					<button
						class="text-gray-500 text-[0.9rem] hover:text-gray-700 dark:text-gray-400 
                           dark:hover:text-gray-200"
						on:mousedown|preventDefault={() => {
							resetComparison();
						}}
					>
						Reset selection
					</button>
				</div>
			{/if}
		</div>
	</div>
	

</div>

<style>
    /* Add any additional styles here */
    .group:hover button {
        opacity: 1;
    }
	
</style>
