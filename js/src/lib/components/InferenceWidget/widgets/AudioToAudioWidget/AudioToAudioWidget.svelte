<script lang="ts">
	import type { WidgetProps } from "../../shared/types";

	import { onMount } from "svelte";

	import WidgetAudioTrack from "../../shared/WidgetAudioTrack/WidgetAudioTrack.svelte";
	import WidgetFileInput from "../../shared/WidgetFileInput/WidgetFileInput.svelte";
	import WidgetRecorder from "../../shared/WidgetRecorder/WidgetRecorder.svelte";
	import WidgetSubmitBtn from "../../shared/WidgetSubmitBtn/WidgetSubmitBtn.svelte";
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
	let file: Blob | File | null = null;
	let filename: string = "";
	let fileUrl: string;
	let isLoading = false;
	let isRecording = false;
	let modelLoading = {
		isLoading: false,
		estimatedTime: 0,
	};
	let output: AudioItem[] = [];
	let outputJson: string;
	let selectedSampleUrl = "";

	interface AudioItem {
		"blob": string;
		"label": string;
		"src"?: string;
		"content-type": string;
	}

	function onRecordStart() {
		file = null;
		filename = "";
		fileUrl = "";
		isRecording = true;
	}

	function onRecordError(err: string) {
		error = err;
	}

	function onSelectFile(updatedFile: Blob | File) {
		isRecording = false;
		selectedSampleUrl = "";
		if (updatedFile.size !== 0) {
			const date = new Date();
			const time = date.toLocaleTimeString("en-US");
			filename = "name" in updatedFile ? updatedFile.name : `Audio recorded from browser [${time}]`;
			file = updatedFile;
			fileUrl = URL.createObjectURL(file);
		}
	}

	async function getOutput({ withModelLoading = false, isOnLoadCall = false } = {}) {
		if (!file && !selectedSampleUrl) {
			error = "You must select or record an audio file";
			return;
		}

		if (!file && selectedSampleUrl) {
			file = await getBlobFromUrl(selectedSampleUrl);
		}
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
		// Reset values
		computeTime = "";
		error = "";
		modelLoading = { isLoading: false, estimatedTime: 0 };
		output = [];

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
			error = res.error;
		}
	}

	function isValidOutput(arg: any): arg is AudioItem[] {
		return (
			Array.isArray(arg) &&
			arg.every(x => typeof x.blob === "string" && typeof x.label === "string" && typeof x["content-type"] === "string")
		);
	}

	function parseOutput(body: unknown): AudioItem[] {
		if (isValidOutput(body)) {
			for (const item of body) {
				item.src = `data:${item["content-type"]};base64,${item.blob}`;
			}
			return body;
		}
		throw new TypeError("Invalid output: output must be of type Array<blob:string, label:string, content-type:string>");
	}

	function applyInputSample(sample: Record<string, any>) {
		file = null;
		filename = sample.example_title;
		fileUrl = sample.src;
		selectedSampleUrl = sample.src;
		getOutput();
	}

	function previewInputSample(sample: Record<string, any>) {
		filename = sample.example_title;
		fileUrl = sample.src;
		output = [];
		outputJson = "";
	}

	onMount(() => {
		const [exampleTitle, src] = getDemoInputs(model, ["example_title", "src"]);
		if (callApiOnMount && src) {
			filename = exampleTitle ?? "";
			fileUrl = src;
			selectedSampleUrl = src;
			getOutput({ isOnLoadCall: true });
		}
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
			<div class="flex flex-wrap items-center">
				<WidgetFileInput accept="audio/*" classNames="mt-1.5 mr-2" {onSelectFile} />
				<span class="mt-1.5 mr-2">or</span>
				<WidgetRecorder classNames="mt-1.5" {onRecordStart} onRecordStop={onSelectFile} onError={onRecordError} />
			</div>
			{#if fileUrl}
				<WidgetAudioTrack classNames="mt-3" label={filename} src={fileUrl} />
			{/if}
			<WidgetSubmitBtn
				classNames="mt-2"
				isDisabled={isRecording}
				{isLoading}
				onClick={() => {
					getOutput();
				}}
			/>
		</form>
	</svelte:fragment>
	<svelte:fragment slot="bottom">
		{#each output as item}
			<div class="mt-2 flex items-center">
				<span class="mr-2">{item.label}:</span>
				<WidgetAudioTrack classNames="" src={item.src} />
			</div>
		{/each}
	</svelte:fragment>
</WidgetWrapper>
