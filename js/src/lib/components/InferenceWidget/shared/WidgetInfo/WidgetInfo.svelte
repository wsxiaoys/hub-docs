<script lang="ts">
	import type { WidgetProps, ModelLoadInfo, LoadingStatus } from "../types";

	import IconAzureML from "../../../Icons/IconAzureML.svelte";

	export let model: WidgetProps["model"];
	export let computeTime: string;
	export let error: string;
	export let modelLoadInfo: ModelLoadInfo = { status: "unknown" };

	const status = {
		error: "⚠️ This model could not be loaded by the inference API. ⚠️",
		loaded: "This model is currently loaded and running on the Inference API.",
		unknown: "This model can be loaded on the Inference API on-demand.",
	} as const;

	const azureStatus = {
		error: "⚠️ This model could not be loaded.",
		loaded: "This model is loaded and running on AzureML Managed Endpoint",
		unknown: "This model can be loaded loaded on AzureML Managed Endpoint",
	} as const;

	function getStatusReport(
		modelLoadInfo: ModelLoadInfo,
		statuses: Record<LoadingStatus, string>,
		isAzure = false
	): string {
		if (modelLoadInfo.compute_type === "cpu" && modelLoadInfo.status === "loaded" && !isAzure) {
			return `The model is loaded and running on <a class="hover:underline" href="https://huggingface.co/intel" target="_blank">Intel Xeon 3rd Gen Scalable CPU</a>`;
		}
		return statuses[modelLoadInfo.status];
	}

	function getComputeTypeMsg(): string {
		const computeType = modelLoadInfo?.compute_type ?? "cpu";
		if (computeType === "cpu") {
			return "Intel Xeon 3rd Gen Scalable cpu";
		}
		return computeType;
	}
</script>

<div class="mt-2">
	<div class="text-xs text-gray-400">
		{#if model.id === "bigscience/bloom"}
			<div class="flex items-baseline">
				<div class="flex items-center whitespace-nowrap text-gray-700">
					<IconAzureML classNames="mr-1 flex-none" /> Powered by&nbsp;
					<a
						class="underline hover:text-gray-800"
						href="https://azure.microsoft.com/products/machine-learning"
						target="_blank">AzureML</a
					>
				</div>
				<div class="border-dotter mx-2 flex flex-1 -translate-y-px border-b border-gray-100" />
				<div>
					{@html getStatusReport(modelLoadInfo, azureStatus, true)}
				</div>
			</div>
		{:else if computeTime}
			Computation time on {getComputeTypeMsg()}: {computeTime}
		{:else}
			{@html getStatusReport(modelLoadInfo, status)}
		{/if}
	</div>
	{#if error}
		<div class="alert alert-error mt-3">{error}</div>
	{/if}
</div>
