<script lang="ts">
	import type { WidgetProps } from "../../shared/types";

	import { onMount } from "svelte";

	import WidgetFileInput from "../../shared/WidgetFileInput/WidgetFileInput.svelte";
	import WidgetDropzone from "../../shared/WidgetDropzone/WidgetDropzone.svelte";
	import WidgetOutputText from "../../shared/WidgetOutputText/WidgetOutputText.svelte";
	import WidgetWrapper from "../../shared/WidgetWrapper/WidgetWrapper.svelte";
	import { getResponse, getBlobFromUrl, getDemoInputs } from "../../shared/helpers";

	export let apiToken: WidgetProps["apiToken"];
	export let apiUrl: WidgetProps["apiUrl"];
	export let callApiOnMount: WidgetProps["callApiOnMount"];
	export let model: WidgetProps["model"];
	export let noTitle: WidgetProps["noTitle"];
	export let includeCredentials: WidgetProps["includeCredentials"];

	let computeTime = "";
	let error: string = "";
	let isLoading = false;
	let imgSrc = "";
	let modelLoading = {
		isLoading: false,
		estimatedTime: 0,
	};
	let output = "";
	let outputJson: string;
	let warning: string = "";

	function onSelectFile(file: File | Blob) {
		imgSrc = URL.createObjectURL(file);
		getOutput(file);
	}

	async function getOutput(file: File | Blob, { withModelLoading = false, isOnLoadCall = false } = {}) {
		if (!file) {
			return;
		}

		// Reset values
		computeTime = "";
		error = "";
		warning = "";
		output = "";
		outputJson = "";

		const requestBody = { file };

		isLoading = true;

		const res = await getResponse(
			apiUrl,
			model.id,
			requestBody,
			apiToken,
			parseOutput,
			withModelLoading,
			includeCredentials,
			isOnLoadCall
		);

		isLoading = false;
		modelLoading = { isLoading: false, estimatedTime: 0 };

		if (res.status === "success") {
			computeTime = res.computeTime;
			output = res.output;
			outputJson = res.outputJson;
			if (output.length === 0) {
				warning = "No classes were detected";
			}
		} else if (res.status === "loading-model") {
			modelLoading = {
				isLoading: true,
				estimatedTime: res.estimatedTime,
			};
			getOutput(file, { withModelLoading: true });
		} else if (res.status === "error") {
			error = res.error;
		}
	}

	function parseOutput(body: unknown): string {
		if (Array.isArray(body) && body.length) {
			const firstEntry = body[0];
			return firstEntry["generated_text"];
		}
		throw new TypeError("Invalid output: output must be of type Array & non-empty");
	}

	async function applyInputSample(sample: Record<string, any>) {
		imgSrc = sample.src;
		const blob = await getBlobFromUrl(imgSrc);
		getOutput(blob);
	}

	function previewInputSample(sample: Record<string, any>) {
		imgSrc = sample.src;
		output = "";
		outputJson = "";
	}

	onMount(() => {
		(async () => {
			const [src] = getDemoInputs(model, ["src"]);
			if (callApiOnMount && src) {
				imgSrc = src;
				const blob = await getBlobFromUrl(imgSrc);
				getOutput(blob, { isOnLoadCall: true });
			}
		})();
	});
</script>

<WidgetWrapper
	{apiUrl}
	{includeCredentials}
	{applyInputSample}
	{computeTime}
	{error}
	{isLoading}
	{model}
	{modelLoading}
	{noTitle}
	{outputJson}
	{previewInputSample}
>
	<svelte:fragment slot="top">
		<form>
			<WidgetDropzone classNames="hidden md:block" {isLoading} {imgSrc} {onSelectFile} onError={e => (error = e)}>
				{#if imgSrc}
					<img src={imgSrc} class="pointer-events-none mx-auto max-h-44 shadow" alt="" />
				{/if}
			</WidgetDropzone>
			<!-- Better UX for mobile/table through CSS breakpoints -->
			{#if imgSrc}
				{#if imgSrc}
					<div class="mb-2 flex justify-center bg-gray-50 dark:bg-gray-900 md:hidden">
						<img src={imgSrc} class="pointer-events-none max-h-44" alt="" />
					</div>
				{/if}
			{/if}
			<WidgetFileInput
				accept="image/*"
				classNames="mr-2 md:hidden"
				{isLoading}
				label="Browse for image"
				{onSelectFile}
			/>
			{#if warning}
				<div class="alert alert-warning mt-2">{warning}</div>
			{/if}
		</form>
	</svelte:fragment>
	<svelte:fragment slot="bottom">
		{#if model?.pipeline_tag !== "text-generation"}
			<WidgetOutputText classNames="mt-4" {output} />
		{/if}
	</svelte:fragment>
</WidgetWrapper>
