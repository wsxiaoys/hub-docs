<script lang="ts">
	import type { WidgetProps } from "../../shared/types";

	import { onMount } from "svelte";

	import WidgetCheckbox from "../../shared/WidgetCheckbox/WidgetCheckbox.svelte";
	import WidgetOutputChart from "../../shared/WidgetOutputChart/WidgetOutputChart.svelte";
	import WidgetSubmitBtn from "../../shared/WidgetSubmitBtn/WidgetSubmitBtn.svelte";
	import WidgetTextarea from "../../shared/WidgetTextarea/WidgetTextarea.svelte";
	import WidgetTextInput from "../../shared/WidgetTextInput/WidgetTextInput.svelte";
	import WidgetWrapper from "../../shared/WidgetWrapper/WidgetWrapper.svelte";
	import { addInferenceParameters, getDemoInputs, getResponse, getSearchParams, updateUrl } from "../../shared/helpers";

	export let apiToken: WidgetProps["apiToken"];
	export let apiUrl: WidgetProps["apiUrl"];
	export let callApiOnMount: WidgetProps["callApiOnMount"];
	export let model: WidgetProps["model"];
	export let noTitle: WidgetProps["noTitle"];
	export let shouldUpdateUrl: WidgetProps["shouldUpdateUrl"];
	export let includeCredentials: WidgetProps["includeCredentials"];

	let candidateLabels = "";
	let computeTime = "";
	let error: string = "";
	let isLoading = false;
	let modelLoading = {
		isLoading: false,
		estimatedTime: 0,
	};
	let multiClass = false;
	let output: Array<{ label: string; score: number }> = [];
	let outputJson: string;
	let text = "";
	let warning: string = "";
	let setTextAreaValue: (text: string) => void;

	onMount(() => {
		const [candidateLabelsParam, multiClassParam, textParam] = getSearchParams([
			"candidateLabels",
			"multiClass",
			"text",
		]);
		if (candidateLabelsParam && !!multiClassParam && textParam) {
			candidateLabels = candidateLabelsParam;
			multiClass = multiClassParam === "true";
			setTextAreaValue(textParam);
			getOutput();
		} else {
			const [demoCandidateLabels, demoMultiClass, demoText] = getDemoInputs(model, [
				"candidate_labels",
				"multi_class",
				"text",
			]);
			candidateLabels = (demoCandidateLabels as string) ?? "";
			multiClass = demoMultiClass === "true";
			setTextAreaValue(demoText ?? "");
			if (candidateLabels && text && callApiOnMount) {
				getOutput({ isOnLoadCall: true });
			}
		}
	});

	async function getOutput({ withModelLoading = false, isOnLoadCall = false } = {}) {
		const trimmedText = text.trim();
		const trimmedCandidateLabels = candidateLabels.trim().split(",").join(",");

		if (!trimmedText) {
			error = "You need to input some text";
			output = [];
			outputJson = "";
			return;
		}

		if (!trimmedCandidateLabels) {
			error = "You need to input at least one label";
			output = [];
			outputJson = "";
			return;
		}

		if (shouldUpdateUrl && !isOnLoadCall) {
			updateUrl({
				candidateLabels: trimmedCandidateLabels,
				multiClass: multiClass ? "true" : "false",
				text: trimmedText,
			});
		}

		const requestBody = {
			inputs: trimmedText,
			parameters: {
				candidate_labels: trimmedCandidateLabels,
				multi_class: multiClass,
			},
		};
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
			isOnLoadCall
		);

		isLoading = false;
		// Reset values
		computeTime = "";
		error = "";
		warning = "";
		modelLoading = { isLoading: false, estimatedTime: 0 };
		output = [];
		outputJson = "";

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
			getOutput({ withModelLoading: true });
		} else if (res.status === "error") {
			error = res.error;
		}
	}

	function parseOutput(body: unknown): Array<{ label: string; score: number }> {
		if (body && typeof body === "object" && Array.isArray(body["labels"]) && Array.isArray(body["scores"])) {
			return body["labels"]
				.filter((_, i) => body["scores"][i] !== null || body["scores"][i] !== undefined)
				.map((x, i) => ({
					label: x ?? "",
					score: body["scores"][i] ?? 0,
				}));
		}
		throw new TypeError("Invalid output: output must be of type <labels:Array; scores:Array>");
	}

	function previewInputSample(sample: Record<string, any>) {
		candidateLabels = sample.candidate_labels;
		multiClass = sample.multi_class;
		setTextAreaValue(sample.text);
	}

	function applyInputSample(sample: Record<string, any>) {
		candidateLabels = sample.candidate_labels;
		multiClass = sample.multi_class;
		setTextAreaValue(sample.text);
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
		<form class="flex flex-col space-y-2">
			<WidgetTextarea bind:value={text} bind:setValue={setTextAreaValue} placeholder="Text to classify..." />
			<WidgetTextInput
				bind:value={candidateLabels}
				label="Possible class names (comma-separated)"
				placeholder="Possible class names..."
			/>
			<WidgetCheckbox bind:checked={multiClass} label="Allow multiple true classes" />
			<WidgetSubmitBtn
				{isLoading}
				onClick={() => {
					getOutput();
				}}
			/>
			{#if warning}
				<div class="alert alert-warning mt-2">{warning}</div>
			{/if}
		</form>
	</svelte:fragment>
	<svelte:fragment slot="bottom">
		{#if output.length}
			<WidgetOutputChart classNames="pt-4" {output} />
		{/if}
	</svelte:fragment>
</WidgetWrapper>
