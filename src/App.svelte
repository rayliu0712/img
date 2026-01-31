<script lang="ts">
  const FORMATS = [
    { type: "image/jpeg", ext: "jpg" },
    { type: "image/png", ext: "png" },
    { type: "image/webp", ext: "webp" },
  ];
  let quality = $state(100);
  let outputType = $state("image/jpeg");

  const images = $state<{ imgUrl: string; file: File }[]>([]);

  function addNewImages(fileList: FileList) {
    const newImages = Array.from(fileList)
      .filter((file) => file.type.startsWith("image/"))
      .map((file) => ({
        imgUrl: URL.createObjectURL(file),
        file: file,
      }));

    images.splice(0, 0, ...newImages);
  }

  function inputImages() {
    const input = document.createElement("input");
    input.type = "file";
    input.accept = "image/*";
    input.multiple = true;

    input.onchange = () => {
      const fileList = input.files;
      if (fileList) {
        addNewImages(fileList);
      }
    };
    input.click();
  }

  function onPaste(e: ClipboardEvent) {
    const fileList = e.clipboardData?.files;
    if (fileList) {
      addNewImages(fileList);
    }
  }

  function removeImage(i: number) {
    const url = images[i].imgUrl;
    URL.revokeObjectURL(url);
    images.splice(i, 1);
  }

  function onDrop(e: DragEvent) {
    e.preventDefault();

    const fileList = e.dataTransfer?.files;
    if (fileList) {
      addNewImages(fileList);
    }
  }

  function downloadOne(blob: Blob, filename: string) {
    const url = URL.createObjectURL(blob);

    const filenameWithoutExt = filename.slice(0, filename.lastIndexOf("."));
    let ext = blob.type.split("/")[1];
    if (ext == "jpeg") {
      ext = "jpg";
    }

    const a = document.createElement("a");
    a.href = url;
    a.download = `${filenameWithoutExt}.${ext}`;
    a.click();

    URL.revokeObjectURL(url);
  }

  async function convertOne(file: File) {
    const bitmap = await createImageBitmap(file);
    const { width, height } = bitmap;

    const canvas = document.createElement("canvas");
    canvas.width = width;
    canvas.height = height;

    const ctx = canvas.getContext("2d")!;
    ctx.fillStyle = "white";
    ctx.fillRect(0, 0, width, height);
    ctx.drawImage(bitmap, 0, 0, width, height);

    canvas.toBlob(
      (blob) => downloadOne(blob!, file.name),
      outputType,
      quality / 100, // for image/png, quality param is ignored
    );

    bitmap.close();
  }

  async function startConvert() {
    await Promise.all(images.map(({ file }) => convertOne(file)));
  }
</script>

<!-- svelte-ignore a11y_no_static_element_interactions -->
<div
  class="h-screen flex flex-row p-1"
  onpaste={onPaste}
  ondragover={(e) => e.preventDefault()}
  ondrop={onDrop}
>
  <!-- thumbnails -->
  <div
    class="flex-3 grid grid-cols-5 gap-1 content-start overflow-y-scroll pr-1"
  >
    <button
      class="w-full aspect-square bg-neutral-600 cursor-pointer text-5xl hover:bg-neutral-500 duration-150"
      onclick={inputImages}
    >
      +
    </button>

    {#each images as { imgUrl: url }, i}
      <div class="w-full aspect-square relative group">
        <img alt="thumbnail" src={url} class="size-full object-cover" />

        <div
          class="absolute inset-0 duration-300 bg-black opacity-0 group-hover:opacity-70 flex items-center justify-center"
        >
          <button
            class="absolute top-1 right-1 cursor-pointer"
            onclick={() => removeImage(i)}>✕</button
          >
          <button onclick={() => {}} class="cursor-pointer">View</button>
        </div>
      </div>
    {/each}
  </div>

  <!-- panel -->
  <div class="flex-1 p-5 flex flex-col gap-10">
    <button
      class="font-bold cursor-pointer border-2 border-solid py-2 hover:bg-neutral-600 duration-150"
      onclick={() => location.reload()}>Image Convertor</button
    >

    <div class="flex flex-col gap-2">
      <p>Output Format</p>
      <select
        class="bg-neutral-600 rounded-lg cursor-pointer px-4 py-2"
        bind:value={outputType}
      >
        {#each FORMATS as { type, ext }}
          <option value={type}>{ext}</option>
        {/each}
      </select>
    </div>

    {#if outputType !== "image/png"}
      <div class="flex flex-col gap-2">
        <p>Quality {quality}%</p>

        <input
          type="range"
          min="0"
          step="1"
          max="100"
          class="accent-cyan-600 cursor-pointer"
          bind:value={quality}
        />
      </div>
    {/if}

    <button
      onclick={startConvert}
      class="self-end rounded-full duration-150 bg-cyan-700 cursor-pointer hover:bg-cyan-600 px-8 py-2"
      >Download All</button
    >

    <div class="flex-1"></div>

    <a
      class="self-end cursor-pointer text-3xl hover:animate-spin"
      href="https://github.com/rayliu0712/image_convertor"
      target="_blank">🍹</a
    >
  </div>
</div>
