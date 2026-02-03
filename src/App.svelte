<script lang="ts">
  import JSZip from "jszip";

  const FORMATS = new Map([
    ["image/jpeg", "jpg"],
    ["image/png", "png"],
    ["image/webp", "webp"],
  ]);

  const images = $state<
    { url: string; blob: Blob; name: string; ext: string }[]
  >([]);
  let outputType = $state("image/jpeg");
  let quality = $state(100);

  let dialog: HTMLDialogElement;
  let viewIndex = $state(-1);

  function addNewImages(fileList: FileList) {
    const newImages = Array.from(fileList)
      .filter((file) => file.type.startsWith("image/"))
      .map((file) => {
        const filename = file.name;
        const i = filename.lastIndexOf(".");
        const name = i === -1 ? filename : filename.slice(0, i);
        const ext = i === -1 ? "" : filename.slice(i + 1);

        return {
          url: URL.createObjectURL(file),
          blob: file,
          name: name,
          ext: ext,
        };
      });

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
    const url = images[i].url;
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

  function download(blob: Blob, filename: string) {
    const url = URL.createObjectURL(blob);

    const a = document.createElement("a");
    a.href = url;
    a.download = filename;
    a.click();

    URL.revokeObjectURL(url);
  }

  async function convertImageFile(
    blob: Blob,
    name: string,
    i: number,
  ): Promise<{ converted: Blob; filename: string }> {
    const bitmap = await createImageBitmap(blob);
    const { width, height } = bitmap;

    const canvas = document.createElement("canvas");
    canvas.width = width;
    canvas.height = height;

    const ctx = canvas.getContext("2d")!;
    ctx.fillStyle = "white";
    ctx.fillRect(0, 0, width, height);
    ctx.drawImage(bitmap, 0, 0, width, height);
    bitmap.close();

    return new Promise((resolve) =>
      canvas.toBlob(
        (blob) => {
          const ext = FORMATS.get(blob!.type);
          const filename = `${name}_${i + 1}.${ext}`;

          resolve({ converted: blob!, filename: filename });
        },
        outputType,
        quality / 100, // for image/png, quality param is ignored
      ),
    );
  }

  async function clickDownloadButton() {
    const count = images.length;

    if (count <= 4) {
      await Promise.all(
        images.map(async ({ blob, name }, i) => {
          const { converted, filename } = await convertImageFile(blob, name, i);
          download(converted, filename);
        }),
      );

      return;
    }

    // images > 4
    const zip = JSZip();
    await Promise.all(
      images.map(async ({ blob, name }, i) => {
        const { converted, filename } = await convertImageFile(blob, name, i);
        zip.file(filename, converted);
      }),
    );
    const zipBlob = await zip.generateAsync({ type: "blob" });
    download(zipBlob, `converted_${count}_images.zip`);
  }

  async function viewImage(i: number) {
    viewIndex = i;
    // dialog.showModal();
    await new Promise<void>((resolve) => {
      // dialog.onclose = () => resolve();
    });
    viewIndex = -1;
  }
</script>

<div class="flex flex-col h-screen">
  <div class="flex flex-row bg-neutral-700 items-center">
    <a
      class="mx-4 my-2 p-2 cursor-pointer font-bold hover:bg-neutral-600 duration-150 rounded-lg"
      href=".">「img」</a
    >
    <a
      class="text-neutral-300 hover:underline"
      href="https://github.com/rayliu0712/image_convertor"
      >Canvas-Based Image Converter</a
    >
  </div>

  {#if images.length === 0}
    <div class="flex-1 flex items-center justify-center">
      <div
        class="w-3/4 h-3/4 rounded-xl bg-neutral-700/50 flex justify-center items-center"
      >
        <button
          class="bg-emerald-700 py-4 hover:bg-emerald-600 duration-150 rounded-xl cursor-pointer w-1/3"
          onclick={inputImages}>Add Images</button
        >
      </div>
    </div>
  {:else}
    <div
      class="flex-1 self-center w-1/2 md:w-1/3 flex flex-col items-stretch justify-evenly"
    >
      <div class="flex flex-col gap-2">
        <p>Format</p>
        <div class="relative">
          <select
            class="appearance-none bg-neutral-700 text-white rounded-lg px-4 py-2 pr-8 w-full cursor-pointer"
          >
            {#each FORMATS as [type, ext]}
              <option value={type}>{ext}</option>
            {/each}
          </select>
          <span
            class="absolute right-2 top-1/2 -translate-y-1/2 pointer-events-none text-white"
            >▾</span
          >
        </div>
      </div>

      <div class="flex flex-col gap-2">
        <p>Quality</p>
        <input type="range" min="0" max="100" step="1" />
      </div>

      <div class="flex flex-col gap-2">
        <button
          class="bg-emerald-700 py-4 cursor-pointer rounded-xl hover:bg-emerald-600 duration-150"
          >Download All</button
        >

        <button
          class="bg-neutral-700 py-4 cursor-pointer rounded-xl hover:bg-neutral-600 duration-150"
        >
          Copy
        </button>
      </div>
    </div>
  {/if}
</div>
