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
              <!-- Bulma file input styling (you can style these however you want) -->
              <div class="field mb-3">
                <label class="label">Select a Folder</label>
                <div class="control">
                  <!-- 'webkitdirectory directory' for entire folder selection (Chromium browsers) -->
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
                  <!-- Multiple file selection -->
                  <input
                    type="file"
                    multiple
                    @change="onFileSelected"
                  />
                </div>
              </div>

              <!-- Show how many items (files) we've collected so far -->
              <p v-if="files.length > 0" class="mb-3">
                <strong>{{ files.length }}</strong> item(s) selected
              </p>

              <!-- Upload Button -->
              <div class="control">
                <button type="submit" class="button is-primary" :disabled="uploading">
                  <span v-if="!uploading">Upload</span>
                  <span v-else>Uploading...</span>
                </button>
              </div>
            </form>

            <!-- Progress bar (Bulma) -->
            <div v-if="uploading" class="mt-3">
              <progress class="progress is-info" :value="uploadProgress" max="100">
                {{ uploadProgress }}%
              </progress>
              <p>{{ uploadProgress }}%</p>
            </div>

            <p v-if="uploadMessage" class="has-text-success mt-2">{{ uploadMessage }}</p>
          </div>

          <!-- File list table -->
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
                    <!-- Download single file -->
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
                    <!-- Copy link -->
                    <button
                      @click="copyToClipboard(file.download_url)"
                      class="button is-small is-info"
                    >
                      <span class="icon">
                        <i class="fas fa-copy"></i>
                      </span>
                      <span>Copy Link</span>
                    </button>
                    <!-- Remove file -->
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

export default {
  name: "App",
  data() {
    return {
      isAuthenticated: false,
      files: [],          // holds both folder and file picks
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

    // When the user selects a FOLDER
    onFolderSelected(e) {
      const folderFiles = Array.from(e.target.files);
      // Append them to the existing array
      this.files = this.files.concat(folderFiles);
    },

    // When the user selects one or more FILES
    onFileSelected(e) {
      const selectedFiles = Array.from(e.target.files);
      // Append them to the existing array
      this.files = this.files.concat(selectedFiles);
    },

    async uploadFiles() {
      if (this.files.length === 0) {
        return;
      }
      try {
        const token = localStorage.getItem("github_token");
        if (!token || !this.owner || !this.repoName) {
          this.uploadMessage = "Missing token or repo info.";
          return;
        }

        // Prepare form data with *all* selected files
        const formData = new FormData();
        // "file.webkitRelativePath" is crucial for subfolders
        this.files.forEach((file) => {
          formData.append("files", file, file.webkitRelativePath || file.name);
        });

        // Reset progress + set "uploading" state
        this.uploadProgress = 0;
        this.uploading = true;
        this.uploadMessage = "";

        // POST to your multi-file endpoint
        const response = await axios.post(
          `https://statements-eastern-delivers-glen.trycloudflare.com/api/upload/${this.owner}/${this.repoName}`,
          formData,
          {
            headers: {
              Authorization: `Bearer ${token}`,
              "Content-Type": "multipart/form-data",
            },
            onUploadProgress: (progressEvent) => {
              if (progressEvent.total) {
                this.uploadProgress = Math.round(
                  (progressEvent.loaded * 100) / progressEvent.total
                );
              }
            },
          }
        );

        this.uploadMessage = response.data.message || "Upload complete.";
        this.fetchFiles();
      } catch (error) {
        console.error("Failed to upload files:", error);
        this.uploadMessage = "Error uploading files.";
      } finally {
        this.uploading = false;
        this.files = []; // clear the selected items
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
