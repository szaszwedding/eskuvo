<template>
	<div class="container">
		<div class="row">
			<div class="mt-4">
				<div class="upload-section d-flex justify-content-center">
					<input
						type="file"
						ref="fileInput"
						accept="image/*, video/*"
						style="display: none"
						@change="handleFileInputChange"
						multiple
					/>
					<button class="btn btn-success me-3" @click="openFileInput">
						Válaszd ki a képeket!
					</button>
					<button
						:disabled="isUploadDisabled"
						class="btn btn-success"
						@click="uploadFiles"
					>
						Feltöltés
					</button>
				</div>
				<div
					class="d-flex justify-content-center mt-2"
					v-if="selectedFilesCount > 0"
				>
					{{ selectedFilesCount }} feltöltésre kiválasztott fotó(k).
				</div>
			</div>
		</div>
		<div class="row mt-4">
			<div class="col">
				<div class="header-image-container">
					<img
						src="https://weddingdz.blob.core.windows.net/images/albom-borito_1685352002325.jpeg"
						alt="Header Image"
						class="img-fluid"
					/>
				</div>
			</div>
		</div>

		<div class="row mt-4">
			<div class="col">
				<div class="grid-container">
					<a
						v-for="(image, index) in images"
						:key="index"
						:href="image"
						class="grid-item"
					>
						<img :src="image" alt="Image" />
					</a>
				</div>
			</div>
		</div>
	</div>
	<div v-if="loading" class="overlay">
		<div class="spinner-border text-primary" role="status">
			<span class="visually-hidden">Loading...</span>
		</div>
	</div>
</template>

<script>
import { BlobServiceClient } from "@azure/storage-blob";
import SimpleLightbox from "simplelightbox";

export default {
	data() {
		return {
			images: [],
			lightbox: null,
			selectedFiles: [],
			connectionSAS: process.env.VUE_APP_CONNECTION_SAS,
			containerName: "images",
			loading: false,
		};
	},
	computed: {
		isUploadDisabled() {
			return this.selectedFiles.length === 0;
		},
		selectedFilesCount() {
			return this.selectedFiles.length;
		},
	},
	async mounted() {
		this.loading = true;
		await this.fetchImages();
		this.lightbox = new SimpleLightbox(".grid-container a");
		this.loading = false;
	},
	methods: {
		async fetchImages() {
			try {
				const blobServiceClient = new BlobServiceClient(this.connectionSAS);
				const containerClient = blobServiceClient.getContainerClient(
					this.containerName
				);

				const blobItems = containerClient.listBlobsFlat({
					prefix: "",
					includeMetadata: true,
				});

				const sortedBlobs = [];
				for await (const blob of blobItems) {
					sortedBlobs.push(blob);
				}

				sortedBlobs.sort(
					(a, b) =>
						b.properties.createdOn.valueOf() - a.properties.createdOn.valueOf()
				);

				this.images = sortedBlobs.map(
					(blob) => containerClient.getBlobClient(blob.name).url
				);
			} catch (error) {
				console.error("Error retrieving images:", error);
			}
		},

		openFileInput() {
			this.$refs.fileInput.click();
		},

		handleFileInputChange(event) {
			this.selectedFiles = Array.from(event.target.files);
		},

		async uploadFiles() {
			this.loading = true;
			try {
				const files = this.selectedFiles;
				const blobServiceClient = new BlobServiceClient(this.connectionSAS);
				const containerClient = blobServiceClient.getContainerClient(
					this.containerName
				);

				for (let i = 0; i < files.length; i++) {
					const file = files[i];
					const uniqueName = this.generateUniqueBlobName(file.name);
					const blockBlobClient =
						containerClient.getBlockBlobClient(uniqueName);
					await blockBlobClient.uploadBrowserData(file);
				}

				this.selectedFiles = [];
				console.log("Files uploaded successfully!");
				await this.fetchImages();
			} catch (error) {
				console.error("Error uploading files:", error);
			}
			this.loading = false;
		},

		generateUniqueBlobName(originalName) {
			const uniqueIdentifier = Date.now();
			const fileName = originalName.substr(0, originalName.lastIndexOf("."));
			const fileExtension = originalName.substr(originalName.lastIndexOf("."));

			return `${fileName}_${uniqueIdentifier}${fileExtension}`;
		},
	},
};
</script>
<style>
.grid-container {
	display: grid;
	grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
	grid-gap: 10px;
}

.grid-item {
	display: block;
	width: 100%;
	height: auto;
	max-width: 100%;
	max-height: 100%;
}

img {
	width: 100%;
}

.overlay {
	position: fixed;
	top: 0;
	left: 0;
	width: 100%;
	height: 100%;
	background-color: rgba(0, 0, 0, 0.5);
	display: flex;
	align-items: center;
	justify-content: center;
	z-index: 9999;
}
</style>
