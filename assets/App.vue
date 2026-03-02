<template>
  <div class="app-shell">
    <section v-if="!isAuthed" class="login-screen">
      <div class="login-card">
        <div class="login-brand">
          <span class="logo-dot">●</span>
          <span class="logo-text">FlareDrive</span>
        </div>
        <p class="login-subtitle">Sign in to continue</p>
        <form class="login-form" @submit.prevent="login">
          <label for="username">Username</label>
          <input
            id="username"
            v-model.trim="username"
            name="username"
            type="text"
            autocomplete="username"
            required
          />
          <label for="password">Password</label>
          <input
            id="password"
            v-model="password"
            name="password"
            type="password"
            autocomplete="current-password"
            required
          />
          <button type="submit" :disabled="authLoading">
            {{ authLoading ? "Signing in..." : "Sign in" }}
          </button>
        </form>
        <p v-if="authError" class="error-text">{{ authError }}</p>
      </div>
    </section>

    <div
      v-else
      class="main"
      @dragenter.prevent
      @dragover.prevent
      @drop.prevent="onDrop"
      :style="{ backgroundImage: `url('${backgroundImageUrl}')` }"
    >
      <progress v-if="uploadProgress !== null" :value="uploadProgress" max="100"></progress>
      <UploadPopup
        v-model="showUploadPopup"
        @upload="onUploadClicked"
        @createFolder="createFolder"
      ></UploadPopup>

      <header class="gh-header">
        <div class="gh-breadcrumb">
          <span class="repo-owner">FlareDrive</span>
          <span class="slash">/</span>
          <span class="repo-name">README</span>
        </div>
        <div class="gh-actions">
          <input
            class="search-lite"
            type="search"
            v-model="search"
            aria-label="Search"
            placeholder="Search files"
          />
          <button class="ghost" @click="showUploadPopup = true">Upload</button>
          <button class="ghost" @click="showMenu = true">Sort</button>
          <Menu
            v-model="showMenu"
            :items="[{ text: '按照名称排序A-Z' }, { text: '按照大小递增排序' }, { text: '按照大小递减排序' }, { text: '粘贴文件到网盘' }]"
            @click="onMenuClick"
          />
          <button class="ghost danger" @click="logout">Sign out</button>
        </div>
      </header>

      <section class="readme-section">
        <div class="readme-card">
          <div class="readme-header">
            <span>README.md</span>
            <span class="muted">{{ readmeLoading ? 'Loading…' : 'GitHub-like preview' }}</span>
          </div>
          <div v-if="readmeError" class="error-box">{{ readmeError }}</div>
          <div v-else class="markdown-body" v-html="readmeHtml"></div>
        </div>
      </section>

      <div class="file-list-container">
        <ul class="file-list">
          <li v-if="cwd !== ''">
            <div
              tabindex="0"
              class="file-item"
              @click="cwd = cwd.replace(/[^\/]+\/$/, '')"
              @contextmenu.prevent
            >
              <div class="file-icon">
                <svg viewBox="0 0 576 512" xmlns="http://www.w3.org/2000/svg" width="36" height="36">
                  <path
                    d="M384 480l48 0c11.4 0 21.9-6 27.6-15.9l112-192c5.8-9.9 5.8-22.1 .1-32.1S555.5 224 544 224l-400 0c-11.4 0-21.9 6-27.6 15.9L48 357.1 48 96c0-8.8 7.2-16 16-16l117.5 0c4.2 0 8.3 1.7 11.3 4.7l26.5 26.5c21 21 49.5 32.8 79.2 32.8L416 144c8.8 0 16 7.2 16 16l0 32 48 0 0-32c0-35.3-28.7-64-64-64L298.5 96c-17 0-33.3-6.7-45.3-18.7L226.7 50.7c-12-12-28.3-18.7-45.3-18.7L64 32C28.7 32 0 60.7 0 96L0 416c0 35.3 28.7 64 64 64l23.7 0L384 480z"
                  />
                </svg>
              </div>
              <div class="file-info-container"><span class="file-name">返回上级目录</span></div>
            </div>
          </li>
          <li v-for="folder in filteredFolders" :key="folder">
            <div
              tabindex="0"
              class="file-item"
              @click="cwd = folder"
              @contextmenu.prevent="showContextMenu = true; focusedItem = folder;"
            >
              <div class="file-icon">
                <svg viewBox="0 0 576 512" xmlns="http://www.w3.org/2000/svg" width="36" height="36">
                  <path
                    d="M384 480l48 0c11.4 0 21.9-6 27.6-15.9l112-192c5.8-9.9 5.8-22.1 .1-32.1S555.5 224 544 224l-400 0c-11.4 0-21.9 6-27.6 15.9L48 357.1 48 96c0-8.8 7.2-16 16-16l117.5 0c4.2 0 8.3 1.7 11.3 4.7l26.5 26.5c21 21 49.5 32.8 79.2 32.8L416 144c8.8 0 16 7.2 16 16l0 32 48 0 0-32c0-35.3-28.7-64-64-64L298.5 96c-17 0-33.3-6.7-45.3-18.7L226.7 50.7c-12-12-28.3-18.7-45.3-18.7L64 32C28.7 32 0 60.7 0 96L0 416c0 35.3 28.7 64 64 64l23.7 0L384 480z"
                  />
                </svg>
              </div>
              <div class="file-info-container"><span class="file-name" v-text="folder.match(/.*?([^/]*)\/?$/)[1]"></span></div>
              <div style="margin-right: 10px;margin-left: auto;" @click.stop="showContextMenu = true; focusedItem = folder;">
                <svg t="1741761103305" class="icon" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg" p-id="6484" width="30" height="30">
                  <path
                    d="M341.333333 533.333333a128 128 0 0 1 128 128v149.333334a128 128 0 0 1-128 128H192a128 128 0 0 1-128-128v-149.333334a128 128 0 0 1 128-128h149.333333z m469.333334 0a128 128 0 0 1 128 128v149.333334a128 128 0 0 1-128 128h-149.333334a128 128 0 0 1-128-128v-149.333334a128 128 0 0 1 128-128h149.333334z m-469.333334 64H192a64 64 0 0 0-63.893333 60.245334L128 661.333333v149.333334a64 64 0 0 0 60.245333 63.893333L192 874.666667h149.333333a64 64 0 0 0 63.893334-60.245334L405.333333 810.666667v-149.333334a64 64 0 0 0-60.245333-63.893333L341.333333 597.333333z m469.333334 0h-149.333334a64 64 0 0 0-63.893333 60.245334L597.333333 661.333333v149.333334a64 64 0 0 0 60.245334 63.893333L661.333333 874.666667h149.333334a64 64 0 0 0 63.893333-60.245334L874.666667 810.666667v-149.333334a64 64 0 0 0-60.245334-63.893333L810.666667 597.333333zM341.333333 64a128 128 0 0 1 128 128v149.333333a128 128 0 0 1-128 128H192a128 128 0 0 1-128-128V192a128 128 0 0 1 128-128h149.333333z m469.333334 0a128 128 0 0 1 128 128v149.333333a128 128 0 0 1-128 128h-149.333334a128 128 0 0 1-128-128V192a128 128 0 0 1 128-128h149.333334zM341.333333 128H192a64 64 0 0 0-63.893333 60.245333L128 192v149.333333a64 64 0 0 0 60.245333 63.893334L192 405.333333h149.333333a64 64 0 0 0 63.893334-60.245333L405.333333 341.333333V192a64 64 0 0 0-60.245333-63.893333L341.333333 128z m469.333334 0h-149.333334a64 64 0 0 0-63.893333 60.245333L597.333333 192v149.333333a64 64 0 0 0 60.245334 63.893334L661.333333 405.333333h149.333334a64 64 0 0 0 63.893333-60.245333L874.666667 341.333333V192a64 64 0 0 0-60.245334-63.893333L810.666667 128z"
                    fill="#2c2c2c"
                    p-id="6485"
                  ></path>
                </svg>
              </div>
            </div>
          </li>
          <li v-for="file in filteredFiles" :key="file.key">
            <div
              @click="preview(`/raw/${file.key}`)"
              @contextmenu.prevent="showContextMenu = true; focusedItem = file;"
              class="file-item"
              style="position: relative;"
            >
              <MimeIcon
                :content-type="file.httpMetadata.contentType"
                :thumbnail="file.customMetadata.thumbnail ? `/raw/_$flaredrive$/thumbnails/${file.customMetadata.thumbnail}.png` : null"
              />
              <div class="file-info-container">
                <div class="file-name" v-text="file.key.split('/').pop()"></div>
                <div class="file-attr">
                  <span v-text="new Date(file.uploaded).toLocaleString()"></span>
                  <span v-text="formatSize(file.size)"></span>
                </div>
              </div>
              <div style="margin-right: 10px;margin-left: auto;" @click.stop="showContextMenu = true; focusedItem = file;">
                <svg t="1741761103305" class="icon" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg" p-id="6484" width="30" height="30">
                  <path
                    d="M341.333333 533.333333a128 128 0 0 1 128 128v149.333334a128 128 0 0 1-128 128H192a128 128 0 0 1-128-128v-149.333334a128 128 0 0 1 128-128h149.333333z m469.333334 0a128 128 0 0 1 128 128v149.333334a128 128 0 0 1-128 128h-149.333334a128 128 0 0 1-128-128v-149.333334a128 128 0 0 1 128-128h149.333334z m-469.333334 64H192a64 64 0 0 0-63.893333 60.245334L128 661.333333v149.333334a64 64 0 0 0 60.245333 63.893333L192 874.666667h149.333333a64 64 0 0 0 63.893334-60.245334L405.333333 810.666667v-149.333334a64 64 0 0 0-60.245333-63.893333L341.333333 597.333333z m469.333334 0h-149.333334a64 64 0 0 0-63.893333 60.245334L597.333333 661.333333v149.333334a64 64 0 0 0 60.245334 63.893333L661.333333 874.666667h149.333334a64 64 0 0 0 63.893333-60.245334L874.666667 810.666667v-149.333334a64 64 0 0 0-60.245334-63.893333L810.666667 597.333333zM341.333333 64a128 128 0 0 1 128 128v149.333333a128 128 0 0 1-128 128H192a128 128 0 0 1-128-128V192a128 128 0 0 1 128-128h149.333333z m469.333334 0a128 128 0 0 1 128 128v149.333333a128 128 0 0 1-128 128h-149.333334a128 128 0 0 1-128-128V192a128 128 0 0 1 128-128h149.333334zM341.333333 128H192a64 64 0 0 0-63.893333 60.245333L128 192v149.333333a64 64 0 0 0 60.245333 63.893334L192 405.333333h149.333333a64 64 0 0 0 63.893334-60.245333L405.333333 341.333333V192a64 64 0 0 0-60.245333-63.893333L341.333333 128z m469.333334 0h-149.333334a64 64 0 0 0-63.893333 60.245333L597.333333 192v149.333333a64 64 0 0 0 60.245334 63.893334L661.333333 405.333333h149.333334a64 64 0 0 0 63.893333-60.245333L874.666667 341.333333V192a64 64 0 0 0-60.245334-63.893333L810.666667 128z"
                    fill="#2c2c2c"
                    p-id="6485"
                  ></path>
                </svg>
              </div>
            </div>
          </li>
        </ul>
      </div>
      <div v-if="loading" style="margin: 20px 0; text-align: center">
        <span style="font-size: 20px;">加载中...</span>
      </div>
      <div v-else-if="!filteredFiles.length && !filteredFolders.length" style="margin: 20px 0; text-align: center">
        <span style="font-size: 20px;">没有文件</span>
      </div>
      <Dialog v-model="showContextMenu">
        <div
          style="height: 50px;display: flex; justify-content: center; align-items: center; padding:10px; background: #ddd; margin: 0 0 10px 0; border-radius: 8px;"
        >
          <div
            v-text="focusedItem.key || focusedItem"
            class="contextmenu-filename"
            @click.stop.prevent
            style="height:20px;width: 100%; max-width: 100%; white-space: nowrap; overflow: hidden; text-overflow: ellipsis;"
          ></div>
        </div>
        <ul v-if="typeof focusedItem === 'string'" class="contextmenu-list">
          <li>
            <button @click="copyLink(`/?p=${encodeURIComponent(focusedItem)}`)">
              <span>复制链接</span>
            </button>
          </li>
          <li>
            <button @click="moveFile(focusedItem + '_$folder$')">
              <span>移动</span>
            </button>
          </li>
          <li>
            <button style="color: red" @click="removeFile(focusedItem + '_$folder$')">
              <span>删除</span>
            </button>
          </li>
        </ul>
        <ul v-else class="contextmenu-list">
          <li>
            <button @click="renameFile(focusedItem.key)">
              <span>重命名</span>
            </button>
          </li>
          <li>
            <a :href="`/raw/${focusedItem.key}`" target="_blank" download>
              <span>下载</span>
            </a>
          </li>
          <li>
            <button @click="clipboard = focusedItem.key">
              <span>复制</span>
            </button>
          </li>
          <li>
            <button @click="moveFile(focusedItem.key)">
              <span>移动</span>
            </button>
          </li>
          <li>
            <button @click="copyLink(`/raw/${focusedItem.key}`)">
              <span>复制链接</span>
            </button>
          </li>
          <li>
            <button style="color: red" @click="removeFile(focusedItem.key)">
              <span>删除</span>
            </button>
          </li>
        </ul>
      </Dialog>
      <div style="flex:1"></div>
      <Footer />
    </div>
  </div>
</template>

<script>
import { generateThumbnail, blobDigest, multipartUpload, SIZE_LIMIT } from "/assets/main.mjs";
import Dialog from "./Dialog.vue";
import Menu from "./Menu.vue";
import MimeIcon from "./MimeIcon.vue";
import UploadPopup from "./UploadPopup.vue";
import Footer from "./Footer.vue";

export default {
  data: () => ({
    authLoading: false,
    authError: "",
    authToken: null,
    isAuthed: false,
    username: "",
    password: "",
    readmeHtml: "",
    readmeLoading: false,
    readmeError: "",
    cwd: new URL(window.location).searchParams.get("p") || "",
    files: [],
    folders: [],
    clipboard: null,
    focusedItem: null,
    loading: false,
    order: null,
    search: "",
    showContextMenu: false,
    showMenu: false,
    showUploadPopup: false,
    uploadProgress: null,
    uploadQueue: [],
    backgroundImageUrl: "/assets/bg-light.webp",
    markdownRenderer: null,
  }),

  computed: {
    filteredFiles() {
      let files = this.files;
      if (this.search) {
        files = files.filter((file) => file.key.split("/").pop().includes(this.search));
      }
      return files;
    },

    filteredFolders() {
      let folders = this.folders;
      if (this.search) {
        folders = folders.filter((folder) => folder.includes(this.search));
      }
      return folders;
    },
  },

  methods: {
    setAuthHeader(token) {
      this.authToken = token;
      if (token) {
        axios.defaults.headers.common["Authorization"] = token;
      } else {
        delete axios.defaults.headers.common["Authorization"];
      }
    },

    async authorizedFetch(url, options = {}) {
      const headers = new Headers(options.headers || {});
      if (this.authToken) headers.set("Authorization", this.authToken);
      return fetch(url, { ...options, headers });
    },

    handleAuthFailure(message) {
      this.isAuthed = false;
      this.authError = message || "登录已失效，请重新登录";
      this.setAuthHeader(null);
      localStorage.removeItem("fdAuth");
    },

    async autoLogin() {
      const saved = localStorage.getItem("fdAuth");
      if (!saved) return;
      this.setAuthHeader(saved);
      try {
        const ok = await this.tryAuth();
        if (ok) {
          this.isAuthed = true;
          await Promise.all([this.fetchFiles(), this.fetchReadme()]);
        } else {
          this.handleAuthFailure("登录已失效，请重新登录");
        }
      } catch (error) {
        console.error(error);
        this.handleAuthFailure("自动登录失败，请重新输入凭据");
      }
    },

    async tryAuth() {
      const res = await this.authorizedFetch(`/api/children/${this.cwd}`);
      if (res.status === 401) return false;
      if (!res.ok) throw new Error("无法连接服务");
      return true;
    },

    async login() {
      this.authError = "";
      this.readmeError = "";
      this.authLoading = true;
      try {
        const token = `Basic ${btoa(`${this.username}:${this.password}`)}`;
        this.setAuthHeader(token);
        const ok = await this.tryAuth();
        if (!ok) throw new Error("认证失败，请检查用户名或密码");
        localStorage.setItem("fdAuth", token);
        this.isAuthed = true;
        await Promise.all([this.fetchFiles(), this.fetchReadme()]);
      } catch (error) {
        this.handleAuthFailure(error.message || "登录失败");
      } finally {
        this.authLoading = false;
      }
    },

    logout() {
      this.files = [];
      this.folders = [];
      this.readmeHtml = "";
      this.isAuthed = false;
      this.username = "";
      this.password = "";
      this.setAuthHeader(null);
      localStorage.removeItem("fdAuth");
    },

    async ensureMarkdownRenderer() {
      if (this.markdownRenderer) return;
      const { marked } = await import("https://cdn.jsdelivr.net/npm/marked@11.2.0/lib/marked.esm.js");
      marked.setOptions({ breaks: true, gfm: true });
      this.markdownRenderer = marked;
    },

    async renderMarkdown(markdown) {
      await this.ensureMarkdownRenderer();
      return this.markdownRenderer.parse(markdown);
    },

    async fetchReadme() {
      if (!this.isAuthed) return;
      this.readmeLoading = true;
      this.readmeError = "";
      try {
        const res = await this.authorizedFetch("/raw/README.md");
        if (res.status === 401) {
          this.handleAuthFailure("认证失效，请重新登录");
          return;
        }
        if (!res.ok) throw new Error("读取 README.md 失败");
        const markdown = await res.text();
        this.readmeHtml = await this.renderMarkdown(markdown || "# README\n暂无内容");
      } catch (error) {
        this.readmeError = error.message || "无法加载 README.md";
      } finally {
        this.readmeLoading = false;
      }
    },

    copyLink(link) {
      const url = new URL(link, window.location.origin);
      navigator.clipboard.writeText(url.toString());
    },

    async copyPaste(source, target) {
      const uploadUrl = `/api/write/items/${target}`;
      await axios.put(uploadUrl, "", { headers: { "x-amz-copy-source": encodeURIComponent(source) } });
    },

    async createFolder() {
      if (!this.isAuthed) return;
      try {
        const folderName = window.prompt("请输入文件夹名称");
        if (!folderName) return;
        this.showUploadPopup = false;
        const uploadUrl = `/api/write/items/${this.cwd}${folderName}/_$folder$`;
        await axios.put(uploadUrl, "");
        this.fetchFiles();
      } catch (error) {
        fetch("/api/write/")
          .then((value) => {
            if (value.redirected) window.location.href = value.url;
          })
          .catch(() => {});
        console.log(`Create folder failed`);
      }
    },

    async fetchFiles() {
      if (!this.isAuthed) return;
      this.files = [];
      this.folders = [];
      this.loading = true;
      try {
        const res = await this.authorizedFetch(`/api/children/${this.cwd}`);
        if (res.status === 401) {
          this.handleAuthFailure("认证失效，请重新登录");
          this.loading = false;
          return;
        }
        if (!res.ok) {
          this.loading = false;
          return;
        }
        const files = await res.json();
        this.files = files.value;
        if (this.order) {
          this.files.sort((a, b) => {
            if (this.order === "size") {
              return b.size - a.size;
            }
          });
        }
        this.folders = files.folders;
      } finally {
        this.loading = false;
      }
    },

    formatSize(size) {
      const units = ["B", "KB", "MB", "GB", "TB"];
      let i = 0;
      while (size >= 1024) {
        size /= 1024;
        i++;
      }
      return `${size.toFixed(1)} ${units[i]}`;
    },

    onDrop(ev) {
      let files;
      if (ev.dataTransfer.items) {
        files = [...ev.dataTransfer.items].filter((item) => item.kind === "file").map((item) => item.getAsFile());
      } else files = ev.dataTransfer.files;
      this.uploadFiles(files);
    },

    onMenuClick(text) {
      switch (text) {
        case "按照名称排序A-Z":
          this.order = null;
          break;
        case "按照大小递增排序":
          this.order = "大小↑";
          break;
        case "按照大小递减排序":
          this.order = "大小↓";
          break;
        case "粘贴文件到网盘":
          return this.pasteFile();
      }
      this.files.sort((a, b) => {
        if (this.order === "大小↑") {
          return a.size - b.size;
        } else if (this.order === "大小↓") {
          return b.size - a.size;
        } else {
          return a.key.localeCompare(b.key);
        }
      });
    },

    onUploadClicked(fileElement) {
      if (!fileElement.value) return;
      this.uploadFiles(fileElement.files);
      this.showUploadPopup = false;
      fileElement.value = null;
    },

    preview(filePath) {
      window.open(filePath);
    },

    async pasteFile() {
      if (!this.clipboard) return;
      let newName = window.prompt("Rename to:");
      if (newName === null) return;
      if (newName === "") newName = this.clipboard.split("/").pop();
      await this.copyPaste(this.clipboard, `${this.cwd}${newName}`);
      this.fetchFiles();
    },

    async processUploadQueue() {
      if (!this.uploadQueue.length) {
        this.fetchFiles();
        this.uploadProgress = null;
        return;
      }

      const { basedir, file } = this.uploadQueue.pop(0);
      let thumbnailDigest = null;

      if (file.type.startsWith("image/") || file.type === "video/mp4") {
        try {
          const thumbnailBlob = await generateThumbnail(file);
          const digestHex = await blobDigest(thumbnailBlob);

          const thumbnailUploadUrl = `/api/write/items/_$flaredrive$/thumbnails/${digestHex}.png`;
          try {
            await axios.put(thumbnailUploadUrl, thumbnailBlob);
            thumbnailDigest = digestHex;
          } catch (error) {
            fetch("/api/write/")
              .then((value) => {
                if (value.redirected) window.location.href = value.url;
              })
              .catch(() => {});
            console.log(`Upload ${digestHex}.png failed`);
          }
        } catch (error) {
          console.log(`Generate thumbnail failed`);
        }
      }

      try {
        const uploadUrl = `/api/write/items/${basedir}${file.name}`;
        const headers = {};
        const onUploadProgress = (progressEvent) => {
          var percentCompleted = (progressEvent.loaded * 100) / progressEvent.total;
          this.uploadProgress = percentCompleted;
        };
        if (thumbnailDigest) headers["fd-thumbnail"] = thumbnailDigest;
        if (file.size >= SIZE_LIMIT) {
          await multipartUpload(`${basedir}${file.name}`, file, { headers, onUploadProgress });
        } else {
          await axios.put(uploadUrl, file, { headers, onUploadProgress });
        }
      } catch (error) {
        fetch("/api/write/")
          .then((value) => {
            if (value.redirected) window.location.href = value.url;
          })
          .catch(() => {});
        console.log(`Upload ${file.name} failed`, error);
      }
      setTimeout(this.processUploadQueue);
    },

    async removeFile(key) {
      if (!window.confirm(`确定要删除 ${key} 吗？`)) return;
      await axios.delete(`/api/write/items/${key}`);
      this.fetchFiles();
    },

    async renameFile(key) {
      const newName = window.prompt("重命名为:");
      if (!newName) return;
      await this.copyPaste(key, `${this.cwd}${newName}`);
      await axios.delete(`/api/write/items/${key}`);
      this.fetchFiles();
    },

    async moveFile(key) {
      const currentPath = this.cwd;
      const allFolders = [...this.folders];

      if (currentPath !== "") {
        const parentPath = currentPath.replace(/[^\/]+\/$/, "");
        if (!allFolders.includes(parentPath) && parentPath !== "") {
          allFolders.unshift(parentPath);
        }
      }

      if (!allFolders.includes("")) {
        allFolders.unshift("");
      }

      const folderOptions = allFolders.map((folder) => {
        const displayName = folder === "" ? "根目录" : folder === currentPath ? "当前目录" : folder.replace(/.*\/(?!$)|\//g, "") + "/";
        return { display: displayName, value: folder };
      });

      const options = folderOptions.map((opt, index) => `${index + 1}. ${opt.display}`).join("\n");

      const promptText = `请选择目标目录(输入数字):\n${options}\n`;
      const selection = window.prompt(promptText);

      if (!selection) return;

      const selectedIndex = parseInt(selection) - 1;
      if (isNaN(selectedIndex) || selectedIndex < 0 || selectedIndex >= folderOptions.length) {
        alert("无效的选择");
        return;
      }

      const targetPath = folderOptions[selectedIndex].value;

      const fileName = key.split("/").pop();
      const finalFileName = fileName.endsWith("_$folder$") ? fileName.slice(0, -9) : fileName;

      const normalizedPath = targetPath === "" ? "" : targetPath.endsWith("/") ? targetPath : targetPath + "/";

      try {
        if (key.endsWith("_$folder$")) {
          const sourceBasePath = key.slice(0, -9);
          const targetBasePath = normalizedPath + finalFileName + "/";

          const allItems = await this.getAllItems(sourceBasePath);

          const totalItems = allItems.length;
          let processedItems = 0;

          for (const item of allItems) {
            const relativePath = item.key.substring(sourceBasePath.length);
            const newPath = targetBasePath + relativePath;

            try {
              await this.copyPaste(item.key, newPath);
              await axios.delete(`/api/write/items/${item.key}`);

              processedItems++;
              this.uploadProgress = (processedItems / totalItems) * 100;
            } catch (error) {
              console.error(`移动 ${item.key} 失败:`, error);
            }
          }

          const targetFolderPath = targetBasePath.slice(0, -1) + "_$folder$";
          await this.copyPaste(key, targetFolderPath);
          await axios.delete(`/api/write/items/${key}`);

          this.uploadProgress = null;
        } else {
          const targetFilePath = normalizedPath + finalFileName;
          await this.copyPaste(key, targetFilePath);
          await axios.delete(`/api/write/items/${key}`);
        }

        this.fetchFiles();
      } catch (error) {
        console.error("移动失败:", error);
        alert("移动失败,请检查目标路径是否正确");
      }
    },

    async getAllItems(prefix) {
      const items = [];
      let marker = null;

      do {
        const url = new URL(`/api/children/${prefix}`, window.location.origin);
        if (marker) {
          url.searchParams.set("marker", marker);
        }

        const response = await this.authorizedFetch(url);
        const data = await response.json();

        items.push(...data.value);

        for (const folder of data.folders) {
          items.push({ key: folder + "_$folder$", size: 0, uploaded: new Date().toISOString() });
          const subItems = await this.getAllItems(folder);
          items.push(...subItems);
        }

        marker = data.marker;
      } while (marker);

      return items;
    },

    uploadFiles(files) {
      if (this.cwd && !this.cwd.endsWith("/")) this.cwd += "/";

      const uploadTasks = Array.from(files).map((file) => ({ basedir: this.cwd, file }));
      this.uploadQueue.push(...uploadTasks);
      setTimeout(() => this.processUploadQueue());
    },
  },

  watch: {
    cwd: {
      handler() {
        if (!this.isAuthed) return;
        this.fetchFiles();
        const url = new URL(window.location);
        if ((url.searchParams.get("p") || "") !== this.cwd) {
          this.cwd ? url.searchParams.set("p", this.cwd) : url.searchParams.delete("p");
          window.history.pushState(null, "", url.toString());
        }
        document.title = this.cwd.replace(/.*\/(?!$)|\//g, "") === "/"
          ? "FlareDrive-R2 - 优雅的 Cloudflare R2 网盘文件库"
          : `${this.cwd.replace(/.*\/(?!$)|\//g, "") || "/"} - 优雅的 Cloudflare R2 网盘文件库`;
      },
      immediate: true,
    },
  },

  created() {
    window.addEventListener("popstate", () => {
      const searchParams = new URL(window.location).searchParams;
      if (searchParams.get("p") !== this.cwd) this.cwd = searchParams.get("p") || "";
    });
    this.autoLogin();
  },

  components: {
    Dialog,
    Menu,
    MimeIcon,
    UploadPopup,
    Footer,
  },
};
</script>

<style>
.app-shell {
  min-height: 100vh;
  background: #f6f8fa;
  color: #24292f;
  font-family: "Segoe UI", -apple-system, BlinkMacSystemFont, "Segoe UI Adjusted", "Segoe UI", "Liberation Sans", sans-serif;
}

.login-screen {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  background: linear-gradient(135deg, #0d1117 0%, #111827 50%, #0d1117 100%);
  padding: 32px;
}

.login-card {
  width: 360px;
  background: #161b22;
  border: 1px solid #30363d;
  border-radius: 12px;
  padding: 28px;
  box-shadow: 0 16px 48px rgba(0, 0, 0, 0.35);
  color: #c9d1d9;
}

.login-brand {
  display: flex;
  align-items: center;
  gap: 10px;
  font-size: 22px;
  font-weight: 700;
  letter-spacing: 0.5px;
  color: #f0f6fc;
}

.logo-dot {
  color: #58a6ff;
  font-size: 18px;
}

.logo-text {
  text-transform: uppercase;
}

.login-subtitle {
  margin: 8px 0 24px;
  color: #8b949e;
}

.login-form {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.login-form label {
  font-size: 14px;
  color: #c9d1d9;
}

.login-form input {
  background: #0d1117;
  color: #f0f6fc;
  border: 1px solid #30363d;
  border-radius: 6px;
  padding: 10px 12px;
  outline: none;
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
}

.login-form input:focus {
  border-color: #58a6ff;
  box-shadow: 0 0 0 2px rgba(88, 166, 255, 0.3);
}

.login-form button {
  margin-top: 8px;
  background: #238636;
  color: #fff;
  border: none;
  border-radius: 6px;
  padding: 10px 12px;
  cursor: pointer;
  font-weight: 600;
  transition: background 0.2s ease, transform 0.1s ease;
}

.login-form button:disabled {
  opacity: 0.7;
  cursor: not-allowed;
}

.login-form button:hover:not(:disabled) {
  background: #2ea043;
  transform: translateY(-1px);
}

.error-text {
  margin-top: 12px;
  color: #ff7b72;
}

.main {
  display: flex;
  min-height: 100vh;
  background-size: cover;
  background-position: center;
  overflow-y: auto;
  flex-direction: column;
  padding-bottom: 24px;
}

.gh-header {
  position: sticky;
  top: 0;
  z-index: 3;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 14px 24px;
  background: rgba(255, 255, 255, 0.9);
  border-bottom: 1px solid #d0d7de;
  backdrop-filter: blur(12px);
}

.gh-breadcrumb {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 18px;
  font-weight: 600;
}

.repo-owner {
  color: #57606a;
}

.repo-name {
  color: #24292f;
}

.slash {
  color: #8b949e;
}

.gh-actions {
  display: flex;
  align-items: center;
  gap: 10px;
}

.search-lite {
  border: 1px solid #d0d7de;
  border-radius: 6px;
  padding: 8px 10px;
  background: #fff;
  min-width: 200px;
}

.ghost {
  border: 1px solid #d0d7de;
  background: #fff;
  border-radius: 6px;
  padding: 8px 12px;
  cursor: pointer;
  transition: background 0.2s ease;
}

.ghost:hover {
  background: #f3f4f6;
}

.ghost.danger {
  border-color: #ff7b72;
  color: #d1242f;
}

.readme-section {
  max-width: 960px;
  width: 92%;
  margin: 20px auto 0;
}

.readme-card {
  background: #fff;
  border: 1px solid #d0d7de;
  border-radius: 12px;
  box-shadow: 0 4px 12px rgba(27, 31, 35, 0.08);
}

.readme-header {
  display: flex;
  justify-content: space-between;
  padding: 14px 16px;
  border-bottom: 1px solid #d0d7de;
  font-weight: 600;
}

.muted {
  color: #57606a;
  font-weight: 400;
}

.markdown-body {
  padding: 18px;
  color: #24292f;
  line-height: 1.6;
}

.markdown-body h1,
.markdown-body h2,
.markdown-body h3,
.markdown-body h4,
.markdown-body h5,
.markdown-body h6 {
  margin-top: 24px;
  border-bottom: 1px solid #d0d7de;
  padding-bottom: 6px;
}

.markdown-body pre {
  background: #f6f8fa;
  border: 1px solid #d0d7de;
  border-radius: 6px;
  padding: 12px;
  overflow: auto;
}

.markdown-body code {
  background: #f6f8fa;
  padding: 2px 4px;
  border-radius: 4px;
}

.error-box {
  padding: 12px 16px;
  color: #d1242f;
  border-bottom: 1px solid #d0d7de;
}

.file-list-container {
  margin: 20px auto;
  padding: 10px;
  width: 60%;
  max-width: 95%;
  background: rgba(255, 255, 255, 0.5);
  backdrop-filter: blur(10px);
  border-radius: 12px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  transition: width 0.3s ease;
}

@media (max-width: 1280px) {
  .file-list-container {
    width: 768px;
    padding: 10px;
  }
}

@media (max-width: 400px) {
  .gh-actions {
    flex-wrap: wrap;
  }
  .search-lite {
    min-width: 100%;
  }
}

.file-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.file-item {
  display: flex;
  align-items: center;
  padding: 10px;
  border-radius: 8px;
  transition: background 0.15s ease;
  cursor: pointer;
}

.file-item:hover {
  background: #f6f8fa;
}

.file-icon {
  margin-right: 12px;
}

.file-info-container {
  display: flex;
  flex-direction: column;
}

.file-name {
  font-weight: 600;
}

.file-attr {
  color: #57606a;
  font-size: 13px;
  display: flex;
  gap: 12px;
}

.menu-button>button {
  transition: background-color 0.2s ease;
}

.menu-button>button:hover {
  background-color: rgb(212, 212, 212);
}
</style>
