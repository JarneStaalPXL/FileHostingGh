<template>
  <section class="section">
    <div class="container" id="app">
      <!-- Title/Hero area -->
      <div class="hero is-primary mb-5">
        <div class="hero-body">
          <p class="title is-3">GitHub Hosted Files</p>
          <p class="subtitle">Store & Download files in your own GitHub repository</p>
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

          <!-- Upload a file -->
          <div class="box">
            <h3 class="title is-6 mb-3">Upload a File</h3>
            <form @submit.prevent="uploadFile" class="field has-addons">
              <!-- Bulma file input styling -->
              <div class="file has-name mr-2">
                <label class="file-label">
                  <input
                    class="file-input"
                    type="file"
                    name="file"
                    @change="onFileChange"
                  />
                  <span class="file-cta">
                    <span class="file-icon">
                      <i class="fas fa-upload"></i>
                    </span>
                    <span class="file-label"> Choose a file… </span>
                  </span>
                  <span class="file-name" v-if="file">
                    {{ file.name }}
                  </span>
                </label>
              </div>
              <div>
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
          <div v-if="files && files.length > 0" class="mt-5">
            <h3 class="title is-6">Files in Repository</h3>
            <table class="table is-fullwidth is-bordered is-hoverable">
              <thead>
                <tr>
                  <th>File Name</th>
                  <th>Size (bytes)</th>
                  <th>Download</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="file in files" :key="file.sha">
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
                  </td>
                </tr>
              </tbody>
            </table>
          </div>

          <div v-else-if="files && files.length === 0" class="mt-5">
            <p>No files found in this repo yet.</p>
          </div>

          <button @click="fetchFiles" class="button is-info mt-3">
            <span class="icon">
              <i class="fas fa-sync-alt"></i>
            </span>
            <span>Refresh File List</span>
          </button>
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
      file: null,
      uploadMessage: "",
      owner: "",
      repoName: "",
      files: [],
      // For progress bar
      uploading: false,
      uploadProgress: 0,
    };
  },
  methods: {
    async copyToClipboard(url) {
      try {
        await navigator.clipboard.writeText(url);
        this.uploadMessage = "Link copied to clipboard!";
        setTimeout(() => {
          this.uploadMessage = "";
        }, 2000); // Clear the message after 2 seconds
      } catch (error) {
        console.error("Failed to copy link:", error);
        this.uploadMessage = "Failed to copy link.";
      }
    },
    authenticate() {
      // Start the GitHub OAuth flow
      window.location.href = "https://polo-techrepublic-spreading-wright.trycloudflare.com/auth/github";
    },
    onFileChange(e) {
      this.file = e.target.files[0];
    },
    async uploadFile() {
      if (!this.file) return;

      try {
        const token = localStorage.getItem("github_token");
        if (!token || !this.owner || !this.repoName) {
          this.uploadMessage = "Missing token or repo info.";
          return;
        }

        // Prepare form data
        const formData = new FormData();
        formData.append("file", this.file);

        // Reset progress + set "uploading" state
        this.uploadProgress = 0;
        this.uploading = true;
        this.uploadMessage = "";

        const response = await axios.post(
          `https://polo-techrepublic-spreading-wright.trycloudflare.com/api/upload/${this.owner}/${this.repoName}`,
          formData,
          {
            headers: {
              Authorization: `Bearer ${token}`,
              "Content-Type": "multipart/form-data",
            },
            onUploadProgress: (progressEvent) => {
              // progressEvent.loaded / progressEvent.total gives fraction
              if (progressEvent.total) {
                this.uploadProgress = Math.round(
                  (progressEvent.loaded * 100) / progressEvent.total
                );
              }
            },
          }
        );

        this.uploadMessage = response.data.message;

        // Refresh the file list
        this.fetchFiles();
      } catch (error) {
        console.error("Failed to upload file:", error);
        this.uploadMessage = "Error uploading file.";
      } finally {
        this.uploading = false;
      }
    },
    async fetchFiles() {
      try {
        const token = localStorage.getItem("github_token");
        if (!token || !this.owner || !this.repoName) return;

        const response = await axios.get(
          `https://polo-techrepublic-spreading-wright.trycloudflare.com/api/list-files/${this.owner}/${this.repoName}`,
          {
            headers: {
              Authorization: `Bearer ${token}`,
            },
          }
        );
        this.files = response.data;
      } catch (error) {
        console.error("Failed to fetch files:", error);
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
    // If already logged in
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
/* We assume you imported Bulma in main.js or similar.
   Optionally, override some styles here. */

#app {
  max-width: 900px;
  margin: 0 auto;
}
.my-5 {
  margin-top: 2rem;
  margin-bottom: 2rem;
}
.file-name {
  max-width: 150px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
</style>
