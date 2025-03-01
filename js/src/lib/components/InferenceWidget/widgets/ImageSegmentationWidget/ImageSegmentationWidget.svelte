<script lang="ts">
	import type { WidgetProps, ImageSegment } from "../../shared/types";

	import { onMount } from "svelte";

	import { COLORS } from "../../shared/consts";
	import { clamp, mod, hexToRgb } from "../../../../utils/ViewUtils";
	import { getResponse, getBlobFromUrl, getDemoInputs } from "../../shared/helpers";
	import WidgetFileInput from "../../shared/WidgetFileInput/WidgetFileInput.svelte";
	import WidgetDropzone from "../../shared/WidgetDropzone/WidgetDropzone.svelte";
	import WidgetOutputChart from "../../shared/WidgetOutputChart/WidgetOutputChart.svelte";
	import WidgetWrapper from "../../shared/WidgetWrapper/WidgetWrapper.svelte";

	import Canvas from "./Canvas.svelte";

	export let apiToken: WidgetProps["apiToken"];
	export let apiUrl: WidgetProps["apiUrl"];
	export let callApiOnMount: WidgetProps["callApiOnMount"];
	export let model: WidgetProps["model"];
	export let noTitle: WidgetProps["noTitle"];
	export let includeCredentials: WidgetProps["includeCredentials"];

	const maskOpacity = Math.floor(255 * 0.6);
	const colorToRgb = COLORS.reduce((acc, clr) => {
		const [r, g, b]: number[] = hexToRgb(clr.hex);
		const clrWithRgb = { ...clr, r, g, b };
		return { ...acc, [clr.color]: clrWithRgb };
	}, {});

	let computeTime = "";
	let error: string = "";
	let highlightIndex = -1;
	let isLoading = false;
	let imgSrc = "";
	let imgEl: HTMLImageElement;
	let imgW = 0;
	let imgH = 0;
	let modelLoading = {
		isLoading: false,
		estimatedTime: 0,
	};
	let output: ImageSegment[] = [];
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
		output = [];
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
			const output_ = res.output;
			if (output_.length === 0) {
				warning = "No object was detected";
			} else {
				imgW = imgEl.naturalWidth;
				imgH = imgEl.naturalHeight;
				isLoading = true;
				output = (
					await Promise.all(output_.map((o, idx) => addOutputColor(o, idx)).map(o => addOutputCanvasData(o)))
				).filter(o => o !== undefined) as ImageSegment[];
				isLoading = false;
			}
			outputJson = res.outputJson;
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

	function isValidOutput(arg: any): arg is ImageSegment[] {
		return (
			Array.isArray(arg) &&
			arg.every(x => typeof x.label === "string" && typeof x.score === "number" && typeof x.mask === "string")
		);
	}
	function parseOutput(body: unknown): ImageSegment[] {
		if (isValidOutput(body)) {
			return body;
		}
		throw new TypeError("Invalid output: output must be of type Array<{label:string; score:number; mask: string}>");
	}

	function mouseout() {
		highlightIndex = -1;
	}

	function mouseover(index: number) {
		highlightIndex = index;
	}

	function mousemove(e: any, canvasW: number, canvasH: number) {
		let { layerX, layerY } = e;
		layerX = clamp(layerX, 0, canvasW);
		layerY = clamp(layerY, 0, canvasH);
		const row = Math.floor((layerX / canvasH) * imgH);
		const col = Math.floor((layerY / canvasW) * imgW);
		highlightIndex = -1;
		const index = (imgW * col + row) * 4;
		for (const [i, o] of output.entries()) {
			const pixel = o?.imgData?.data[index];
			if (pixel && pixel > 0) {
				highlightIndex = i;
			}
		}
	}

	function addOutputColor(imgSegment: ImageSegment, idx: number) {
		const hash = mod(idx, COLORS.length);
		const { color } = COLORS[hash];
		return { ...imgSegment, color };
	}

	async function addOutputCanvasData(imgSegment: ImageSegment): Promise<ImageSegment | undefined> {
		const { mask, color } = imgSegment;

		const maskImg = new Image();
		maskImg.src = `data:image/png;base64, ${mask}`;
		// await image.onload
		await new Promise((resolve, _) => {
			maskImg.onload = () => resolve(maskImg);
		});
		const imgData = getImageData(maskImg);
		if (imgData && color) {
			const { r, g, b } = colorToRgb[color];
			const maskColored = [r, g, b, maskOpacity];
			const background = Array(4).fill(0);

			for (let i = 0; i < imgData.data.length; i += 4) {
				const [r, g, b, a] = imgData.data[i] === 255 ? maskColored : background;
				imgData.data[i] = r;
				imgData.data[i + 1] = g;
				imgData.data[i + 2] = b;
				imgData.data[i + 3] = a;
			}

			const bitmap = await createImageBitmap(imgData);
			return { ...imgSegment, imgData, bitmap };
		}
	}

	function getImageData(maskImg: CanvasImageSource): ImageData | undefined {
		const tmpCanvas = document.createElement("canvas");
		tmpCanvas.width = imgW;
		tmpCanvas.height = imgH;
		const tmpCtx = tmpCanvas.getContext("2d");
		tmpCtx?.drawImage(maskImg, 0, 0, imgW, imgH);
		const segmentData = tmpCtx?.getImageData(0, 0, imgW, imgH);
		return segmentData;
	}

	// original: https://gist.github.com/MonsieurV/fb640c29084c171b4444184858a91bc7
	function polyfillCreateImageBitmap() {
		(window as any).createImageBitmap = async function (data: ImageData): Promise<ImageBitmap> {
			return new Promise((resolve, _) => {
				const canvas = document.createElement("canvas");
				const ctx = canvas.getContext("2d");
				canvas.width = data.width;
				canvas.height = data.height;
				ctx?.putImageData(data, 0, 0);
				const dataURL = canvas.toDataURL();
				const img = document.createElement("img");
				img.addEventListener("load", () => {
					resolve(img as any as ImageBitmap);
				});
				img.src = dataURL;
			});
		};
	}

	async function applyInputSample(sample: Record<string, any>) {
		imgSrc = sample.src;
		const blob = await getBlobFromUrl(imgSrc);
		getOutput(blob);
	}

	function previewInputSample(sample: Record<string, any>) {
		imgSrc = sample.src;
		output = [];
		outputJson = "";
	}

	onMount(() => {
		if (typeof createImageBitmap === "undefined") {
			polyfillCreateImageBitmap();
		}

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
					<Canvas {imgSrc} {highlightIndex} {mousemove} {mouseout} {output} />
				{/if}
			</WidgetDropzone>
			<!-- Better UX for mobile/table through CSS breakpoints -->
			{#if imgSrc}
				<Canvas classNames="mr-2 md:hidden" {imgSrc} {highlightIndex} {mousemove} {mouseout} {output} />
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
			<img alt="" bind:this={imgEl} class="hidden" src={imgSrc} />
		</form>
	</svelte:fragment>
	<svelte:fragment slot="bottom">
		<WidgetOutputChart classNames="pt-4" {output} {highlightIndex} {mouseover} {mouseout} />
	</svelte:fragment>
</WidgetWrapper>
