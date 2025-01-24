<template>
  <section class="section">
    <div class="container" id="app">
      <!-- Title/Hero area -->
      <div class="hero is-primary mb-5">
        <div class="hero-body">
          <p class="title is-3">GitHub Hosted Files</p>
          <p class="subtitle">
            Store &amp; Download files (and folders!) in your own GitHub repository
          </p>
        </div>
      </div>

      <!-- If not authenticated, show the connect card -->
      <div v-if="!isAuthenticated" class="card my-5">
        <div class="card-content">
          <p class="title is-5">Connect to GitHub</p>
          <p class="content">Please sign in with GitHub to get started.</p>
          <button @click="authenticate" class="button is-link">
            <span class="icon">
              <i class="fab fa-github"></i>
            </span>
            <span>Connect GitHub</span>
          </button>
        </div>
      </div>

      <!-- If authenticated, show the main UI -->
      <div v-else class="card my-5">
        <div class="card-content">
          <h2 class="title is-5">Welcome to your hosted files repo!</h2>
          <p>
            <strong>Owner:</strong> {{ owner }}<br />
            <strong>Repository:</strong> {{ repoName }}
          </p>
          <hr />

          <!-- Upload multiple files/folders -->
          <div class="box">
            <h3 class="title is-6 mb-3">Upload Files or Folders</h3>

            <form @submit.prevent="uploadFiles" class="field">
              <div class="field mb-3">
                <label class="label">Select a Folder</label>
                <div class="control">
                  <!-- 'webkitdirectory directory' for entire folder selection -->
                  <input
                    type="file"
                    webkitdirectory
                    directory
                    @change="onFolderSelected"
                  />
                </div>
              </div>

              <div class="field mb-3">
                <label class="label">Or Select Files</label>
                <div class="control">
                  <input type="file" multiple @change="onFileSelected" />
                </div>
              </div>

              <!-- Show how many chunked items we have in the array -->
              <p v-if="files.length > 0" class="mb-3">
                <strong>{{ files.length }}</strong> item(s) ready
              </p>

              <div class="control">
                <button type="submit" class="button is-primary" :disabled="uploading">
                  <span v-if="!uploading">Upload</span>
                  <span v-else>Uploading...</span>
                </button>
              </div>
            </form>

            <!-- Progress bar -->
            <div v-if="uploading" class="mt-3">
              <progress class="progress is-info" :value="uploadProgress" max="100">
                {{ uploadProgress }}%
              </progress>
              <p>{{ uploadProgress }}%</p>
            </div>

            <p v-if="uploadMessage" class="has-text-success mt-2">{{ uploadMessage }}</p>
          </div>

          <!-- File list table from the repo -->
          <div v-if="filesInRepo && filesInRepo.length > 0" class="mt-5">
            <h3 class="title is-6">Files in Repository</h3>
            <table class="table is-fullwidth is-bordered is-hoverable">
              <thead>
                <tr>
                  <th>File Name</th>
                  <th>Size (bytes)</th>
                  <th>Actions</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="file in filesInRepo" :key="file.sha">
                  <td>{{ file.name }}</td>
                  <td>{{ file.size }}</td>
                  <td class="buttons">
                    <a
                      :href="file.download_url"
                      target="_blank"
                      rel="noopener noreferrer"
                      class="button is-small is-link"
                    >
                      <span class="icon">
                        <i class="fas fa-download"></i>
                      </span>
                      <span>Download</span>
                    </a>
                    <button
                      @click="copyToClipboard(file.download_url)"
                      class="button is-small is-info"
                    >
                      <span class="icon">
                        <i class="fas fa-copy"></i>
                      </span>
                      <span>Copy Link</span>
                    </button>
                    <button
                      @click="removeFile(file.name)"
                      class="button is-small is-danger"
                    >
                      <span class="icon">
                        <i class="fas fa-trash"></i>
                      </span>
                      <span>Remove</span>
                    </button>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>

          <div v-else-if="filesInRepo && filesInRepo.length === 0" class="mt-5">
            <p>No files found in this repo yet.</p>
          </div>

          <!-- Refresh and Download entire repo -->
          <div class="buttons mt-3">
            <button @click="fetchFiles" class="button is-info">
              <span class="icon">
                <i class="fas fa-sync-alt"></i>
              </span>
              <span>Refresh File List</span>
            </button>
            <button @click="downloadRepo" class="button is-primary">
              <span class="icon">
                <i class="fas fa-file-archive"></i>
              </span>
              <span>Download Full Repo</span>
            </button>
          </div>
        </div>
      </div>
    </div>
  </section>
</template>

<script>
import axios from "axios";

// Let’s define a constant 100 MB chunk threshold
const MAX_CHUNK_SIZE = 100 * 1024 * 1024; // 100 MB

export default {
  name: "App",
  data() {
    return {
      isAuthenticated: false,

      // We'll store final upload "entries" in this.files,
      // Each entry is { blob: File|Blob, displayName: string }
      files: [],

      uploadMessage: "",
      owner: "",
      repoName: "",
      filesInRepo: [],
      uploading: false,
      uploadProgress: 0,
    };
  },
  methods: {
    // Start GitHub OAuth
    authenticate() {
      window.location.href =
        "https://statements-eastern-delivers-glen.trycloudflare.com/auth/github";
    },

    // Called when user selects a folder (webkitdirectory)
    onFolderSelected(e) {
      const folderFiles = Array.from(e.target.files);
      if (folderFiles.length === 0) {
        console.error("No files selected from folder.");
        return;
      }

      // Extract folder name and log the file paths
      const folderName = folderFiles[0]?.webkitRelativePath.split("/")[0];
      console.log("Selected folder:", folderName);

      folderFiles.forEach((file) => {
        console.log("Processing file:", file.webkitRelativePath);
        this.addFileWithChunking(file);
      });
    },

    // Called when user selects normal single/multiple files
    onFileSelected(e) {
      const selectedFiles = Array.from(e.target.files);
      selectedFiles.forEach((file) => {
        this.addFileWithChunking(file);
      });
    },

    // The "magic" method: checks if file > 100MB => splits
    addFileWithChunking(file) {
      // If the file is smaller than 100 MB, it's just one chunk
      if (file.size <= MAX_CHUNK_SIZE) {
        // We keep the "displayName" as either webkitRelativePath or fallback to file.name
        const name = file.webkitRelativePath || file.name;
        this.files.push({
          blob: file,
          displayName: name,
        });
      } else {
        // If bigger, slice it into 100 MB pieces
        const totalChunks = Math.ceil(file.size / MAX_CHUNK_SIZE);
        for (let i = 0; i < totalChunks; i++) {
          const start = i * MAX_CHUNK_SIZE;
          const end = Math.min(file.size, start + MAX_CHUNK_SIZE);
          const chunkBlob = file.slice(start, end);

          // We'll name the chunk so the backend can reassemble
          // e.g. "myFolder/myBigFile.mp4__CHUNK_0-of-5"
          const baseName = file.webkitRelativePath || file.name;
          const chunkName = `${baseName}__CHUNK_${i}-of-${totalChunks}`;

          this.files.push({
            blob: chunkBlob,
            displayName: chunkName,
          });
        }
      }
    },

    async uploadFiles() {
      const CHUNK_SIZE = 100; // Number of files per batch (adjust as needed)
      const token = localStorage.getItem("github_token");
      if (!token || !this.owner || !this.repoName) {
        this.uploadMessage = "Missing token or repo info.";
        return;
      }

      const chunks = [];
      for (let i = 0; i < this.files.length; i += CHUNK_SIZE) {
        chunks.push(this.files.slice(i, i + CHUNK_SIZE));
      }

      this.uploading = true;
      this.uploadProgress = 0;

      try {
        for (let index = 0; index < chunks.length; index++) {
          const chunk = chunks[index];
          const formData = new FormData();
          chunk.forEach((entry) => {
            formData.append("files", entry.blob, entry.displayName);
          });

          // Send the chunk
          await axios.post(
            `https://statements-eastern-delivers-glen.trycloudflare.com/api/upload/${this.owner}/${this.repoName}`,
            formData,
            {
              headers: {
                Authorization: `Bearer ${token}`,
                "Content-Type": "multipart/form-data",
              },
              onUploadProgress: (progressEvent) => {
                if (progressEvent.total) {
                  const totalFilesUploaded = index * CHUNK_SIZE + progressEvent.loaded;
                  this.uploadProgress = Math.round(
                    (totalFilesUploaded * 100) / this.files.length
                  );
                }
              },
            }
          );

          console.log(`Batch ${index + 1}/${chunks.length} uploaded.`);
        }

        this.uploadMessage = "All files uploaded successfully.";
      } catch (error) {
        console.error("Failed to upload batch:", error);
        this.uploadMessage = "Error uploading files.";
      } finally {
        this.uploading = false;
        this.files = []; // Clear files after upload
      }
    },
    async removeFile(fileName) {
      const confirmDelete = confirm(`Are you sure you want to delete "${fileName}"?`);
      if (!confirmDelete) return;

      try {
        const token = localStorage.getItem("github_token");
        if (!token || !this.owner || !this.repoName) {
          alert("Missing token or repo information.");
          return;
        }

        const response = await axios.delete(
          `https://statements-eastern-delivers-glen.trycloudflare.com/api/delete-file/${this.owner}/${this.repoName}/${fileName}`,
          {
            headers: {
              Authorization: `Bearer ${token}`,
            },
          }
        );

        alert(response.data.message);
        this.fetchFiles();
      } catch (error) {
        console.error("Failed to delete file:", error?.response?.data || error);
        alert("Error deleting file.");
      }
    },

    async copyToClipboard(url) {
      try {
        await navigator.clipboard.writeText(url);
        this.uploadMessage = "Link copied to clipboard!";
        setTimeout(() => {
          this.uploadMessage = "";
        }, 2000);
      } catch (error) {
        console.error("Failed to copy link:", error);
        this.uploadMessage = "Failed to copy link.";
      }
    },

    async fetchFiles() {
      try {
        const token = localStorage.getItem("github_token");
        if (!token || !this.owner || !this.repoName) return;

        const response = await axios.get(
          `https://statements-eastern-delivers-glen.trycloudflare.com/api/list-files/${this.owner}/${this.repoName}`,
          {
            headers: {
              Authorization: `Bearer ${token}`,
            },
          }
        );
        this.filesInRepo = response.data;
      } catch (error) {
        console.error("Failed to fetch files:", error);
      }
    },

    async downloadRepo() {
      try {
        const token = localStorage.getItem("github_token");
        if (!token || !this.owner || !this.repoName) return;

        const response = await axios.get(
          `https://statements-eastern-delivers-glen.trycloudflare.com/api/download-repo/${this.owner}/${this.repoName}`,
          {
            headers: { Authorization: `Bearer ${token}` },
            responseType: "arraybuffer",
          }
        );

        // Create a Blob and download it
        const blob = new Blob([response.data], { type: "application/zip" });
        const downloadUrl = URL.createObjectURL(blob);

        const link = document.createElement("a");
        link.href = downloadUrl;
        link.download = `${this.repoName}.zip`;
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
        URL.revokeObjectURL(downloadUrl);
      } catch (error) {
        console.error("Failed to download repo:", error);
      }
    },
  },
  mounted() {
    // Check URL params for token, owner, repo
    const params = new URLSearchParams(window.location.search);
    const token = params.get("token");
    if (!token) {
      this.isAuthenticated = false;
      return;
    }
    const owner = params.get("owner");
    const repo = params.get("repo");

    if (token && owner && repo) {
      localStorage.setItem("github_token", token);
      localStorage.setItem("github_owner", owner);
      localStorage.setItem("github_repo", repo);
      this.isAuthenticated = true;
      this.owner = owner;
      this.repoName = repo;
      window.history.replaceState({}, document.title, "/");
      this.fetchFiles();
      return;
    }

    // If already logged in previously
    const storedToken = localStorage.getItem("github_token");
    const storedOwner = localStorage.getItem("github_owner");
    const storedRepo = localStorage.getItem("github_repo");

    if (storedToken && storedOwner && storedRepo) {
      this.isAuthenticated = true;
      this.owner = storedOwner;
      this.repoName = storedRepo;
      this.fetchFiles();
    }
  },
};
</script>

<style scoped>
#app {
  max-width: 900px;
  margin: 0 auto;
}
.my-5 {
  margin-top: 2rem;
  margin-bottom: 2rem;
}
.file-name {
  max-width: 200px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
</style>
