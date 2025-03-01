<script lang="ts">
	import type { WidgetProps } from "../../shared/types";

	import { onMount } from "svelte";

	import WidgetQuickInput from "../../shared/WidgetQuickInput/WidgetQuickInput.svelte";
	import WidgetWrapper from "../../shared/WidgetWrapper/WidgetWrapper.svelte";
	import { addInferenceParameters, getDemoInputs, getResponse, getSearchParams, updateUrl } from "../../shared/helpers";

	export let apiToken: WidgetProps["apiToken"];
	export let apiUrl: WidgetProps["apiUrl"];
	export let callApiOnMount: WidgetProps["callApiOnMount"];
	export let model: WidgetProps["model"];
	export let noTitle: WidgetProps["noTitle"];
	export let shouldUpdateUrl: WidgetProps["shouldUpdateUrl"];
	export let includeCredentials: WidgetProps["includeCredentials"];

	let computeTime = "";
	let error: string = "";
	let isLoading = false;
	let modelLoading = {
		isLoading: false,
		estimatedTime: 0,
	};
	let output = "";
	let outputJson = "";
	let text = "";

	onMount(() => {
		const [textParam] = getSearchParams(["text"]);
		if (textParam) {
			text = textParam;
			getOutput({ useCache: true });
		} else {
			const [demoText] = getDemoInputs(model, ["text"]);
			text = (demoText as string) ?? "";
			if (text && callApiOnMount) {
				getOutput({ isOnLoadCall: true, useCache: true });
			}
		}
	});

	async function getOutput({ withModelLoading = false, isOnLoadCall = false, useCache = false } = {}) {
		const trimmedText = text.trim();

		if (!trimmedText) {
			error = "You need to input some text";
			output = "";
			return;
		}

		if (shouldUpdateUrl && !isOnLoadCall) {
			updateUrl({ text: trimmedText });
		}

		const requestBody = { inputs: trimmedText };
		addInferenceParameters(requestBody, model);

		isLoading = true;

		const res = await getResponse(
			apiUrl,
			model.id,
			requestBody,
			apiToken,
			parseOutput,
			withModelLoading,
			includeCredentials,
			isOnLoadCall,
			useCache
		);

		isLoading = false;
		// Reset values
		computeTime = "";
		error = "";
		modelLoading = { isLoading: false, estimatedTime: 0 };
		output = "";
		outputJson = "";

		if (res.status === "success") {
			computeTime = res.computeTime;
			output = res.output;
			outputJson = res.outputJson;
		} else if (res.status === "loading-model") {
			modelLoading = {
				isLoading: true,
				estimatedTime: res.estimatedTime,
			};
			getOutput({ withModelLoading: true });
		} else if (res.status === "error") {
			error = res.error || `Error encountered on input "${trimmedText}"`;
		}
	}

	function parseOutput(body: unknown): string {
		if (body && typeof body === "object" && body instanceof Blob) {
			return URL.createObjectURL(body);
		}
		throw new TypeError("Invalid output: output must be of type object & of instance Blob");
	}

	function previewInputSample(sample: Record<string, any>) {
		text = sample.text;
	}

	function applyInputSample(sample: Record<string, any>) {
		text = sample.text;
		getOutput();
	}
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
			<WidgetQuickInput bind:value={text} {isLoading} onClickSubmitBtn={() => getOutput()} />
		</form>
	</svelte:fragment>
	<svelte:fragment slot="bottom">
		{#if output.length}
			<div class="mt-4 flex justify-center bg-gray-50 dark:bg-gray-925">
				<img class="max-w-sm object-contain" src={output} alt="" />
			</div>
		{/if}
	</svelte:fragment>
</WidgetWrapper>
