<script lang="ts">
import { tutorial } from "../ts/Tutorial";
import Dexie from "../ts/indexDB";

import Editor from "../components/Editor.vue";
import Preview from "../components/Preview.vue";
import Modal from "../components/Modal.vue";
import { compress } from "shrink-string";

import pako from "pako";
import JSZip from "jszip";


// @ts-ignore
// import JSONWorker from "url:monaco-editor/esm/vs/language/json/json.worker.js";
// @ts-ignore
// import CSSWorker from "url:monaco-editor/esm/vs/language/css/css.worker.js";
// @ts-ignore
// import HTMLWorker from "url:monaco-editor/esm/vs/language/html/html.worker.js";
// @ts-ignore
// import TSWorker from "url:monaco-editor/esm/vs/language/typescript/ts.worker.js";
// @ts-ignore
import EditorWorker from "url:monaco-editor/esm/vs/editor/editor.worker.js";
import { editor } from "monaco-editor";
import { randomString } from '../ts/utils';
import { ValueTypes } from "yjs/dist/src/internals";
import TextEditorInterface from '../ts/TextEditorInterface';

// @ts-ignore
window.MonacoEnvironment = {
  getWorkerUrl: function (moduleId, label) {
    /*
    if (label === "json") {
      return JSONWorker;
    }
    if (label === "css" || label === "scss" || label === "less") {
      return CSSWorker;
    }
    if (label === "html" || label === "handlebars" || label === "razor") {
      return HTMLWorker;
    }
    if (label === "typescript" || label === "javascript") {
      return TSWorker;
    }
    */
    return EditorWorker;
  }
  
};

export default {
  name: "LiaScript",

  props: ["storageId", "content", "fileUrl", "connection"],

  data() {
    let database: Dexie | undefined;

    if (this.$props.storageId) {
      database = new Dexie();
      database.maybeInit(this.$props.storageId);

      const self = this;
      database.watch(this.$props.storageId, (meta) => {
        self.meta = meta;
      });
    }

    let connectionType = this.$props.connection || "offline";

    switch (connectionType) {
      case "webrtc":
        connectionType = "WebRTC";
        break;
      case "websocket":
        connectionType = "WebSocket";
        break;
      default:
        connectionType = "Offline";
    }

    return {
      preview: undefined,
      previewNotReady: true,
      compilationCounter: 0,
      mode: 0,
      editorIsReady: false,
      database,
      meta: {},
      onlineUsers: 0,
      lights: false,
      conn: {
        type: connectionType,
        users: 0,
      },
    };
  },

  computed: {
    lightMode() {
      return this.lights ? "bi bi-sun" : "bi bi-moon";
    },
  },

  methods: {
    urlPath(path: string[]) {
      return (
        window.location.origin +
        window.location.pathname +
        "?/" +
        path.join("/")
      );
    },

    online(users: number) {
      this.conn.users = users;
    },

    changeMode(mode: number) {
      this.mode = mode;
    },

    shareLink() {
      this.$refs.modal.show(
        "Collaboration link",
        `
        If you share the link below, the editor will be in collaborative mode.
        Working should also be possible offline, but all connected users will work on the same course.
        If you did receive this via a collaboration link and want to make a complete new course by yourself, then you will have to click onto the "Fork" button, which will create a complete new version.

        <hr>

        <a target="_blank" href="${window.location.toString()}">
          ${window.location.toString()}
        </a>`
      );
    },

    shareFile() {
      const fileUrl = prompt(
        "please insert the URL of a Markdown file you want to share",
        ""
      );

      if (!fileUrl) return;

      this.$refs.modal.show(
        "External resource",
        `
        Use this URL to predefine the content for your share link.
        In this case the editor will at first try to load the Markdown file and insert its content into the editor.
        This link will only work if your Markdown file is accessible via the internet.
                    
        <hr>
        
        <a target="_blank" style="word-break: break-all" href="${this.urlPath([
          "show",
          "file",
          fileUrl,
        ])}">
          ${this.urlPath(["show", "file", fileUrl])}
        </a>`
      );
    },

    bytesInfo(url: string) {
      return `<code>URL-length: ${url.length} bytes</code><br>`;
    },

    async shareData() {
      let base64 = "";

      let uriEncode = "";
      let gzip = "";

      try {
        base64 =
          "https://liascript.github.io/course/?data:text/plain;base64," +
          btoa(this.$refs.editor.getValue());

        base64 =
          this.bytesInfo(base64) +
          `<a target="_blank" style="word-break: break-all" href="${base64}">
          ${base64}
        </a>`;
      } catch (e) {}

      try {
        uriEncode =
          "https://liascript.github.io/course/?data:text/plain," +
          encodeURIComponent(this.$refs.editor.getValue());

        uriEncode =
          this.bytesInfo(uriEncode) +
          `<a target="_blank" style="word-break: break-all" href="${uriEncode}">
          ${uriEncode}
        </a>`;
      } catch (e) {}

      try {
        gzip = pako.gzip(this.$refs.editor.getValue());
        gzip =
          "https://liascript.github.io/course/?data:text/plain;charset=utf-8;Content-Encoding=gzip;base64," +
          btoa(String.fromCharCode.apply(null, gzip));

        gzip =
          this.bytesInfo(gzip) +
          `<a target="_blank" style="word-break: break-all" href="${gzip}">
          ${gzip}
        </a>`;
      } catch (e) {}

      this.$refs.modal.show(
        "Data-Protocol",
        `
        The entire content of the course is base64 encoded or URI-encoded put into a data-URI.
        Since base64 might fail for certain languages, the URI-encoded URL is generated as well.
        However, this works only for short courses, the longer the course the longer the URi.
        Sharing your editor via a messenger for example, you will have to check first if no parts are truncated!
        Additionally different browser support different lengths of URLs... (Choose the shorter version)

        <hr>

        ${gzip}

        <hr>
        
        ${base64}
        
        <hr>

        ${uriEncode}
        `
      );
    },

    async shareCode() {
      const zipCode = this.urlPath([
        "show",
        "code",
        await compress(this.$refs.editor.getValue()),
      ]);

      this.$refs.modal.show(
        "Snapshot url",
        `
        Snapshots are URLs that contain the entire course defintion.
        However, this works only for short courses, the longer the course the longer the URL.
        Sharing your editor via a messenger for example, you will have to check first if no parts are truncated!
        Additionally different browser support different lengths of URLs...
                    
        <hr>
        ${this.bytesInfo(zipCode)}
        <a target="_blank" style="word-break: break-all" href="${zipCode}">
          ${zipCode}
        </a>`
      );
    },

    download() {
      const element = document.createElement("a");
      element.setAttribute(
        "href",
        "data:text/plain;charset=utf-8," +
          encodeURIComponent(this.$refs.editor.getValue())
      );
      element.setAttribute("download", "README.md");
      element.style.display = "none";
      document.body.appendChild(element);
      element.click();
      document.body.removeChild(element);
    },

    downloadZip() {
      const zip = JSZip();

      zip.file("README.md", this.$refs.editor.getValue());

      const blobs = this.$refs.editor.getAllBlobs();

      if (blobs) {
        for (const blob of blobs) {
          zip.file(blob.name, blob.data);
        }
      }

      const fileName =
        "Project-" +
        (this.$props?.storageId?.slice(0, 8) || "xxxxxxxx") +
        ".zip";
      zip.generateAsync({ type: "blob" }).then(function (content) {
        let url = URL.createObjectURL(content);

        const element = document.createElement("a");
        element.href = url;
        element.setAttribute("download", fileName);
        element.style.display = "none";
        document.body.appendChild(element);
        element.click();
        document.body.removeChild(element);
        element.addEventListener("click", function () {
          setTimeout(() => URL.revokeObjectURL(url), 30 * 1000);
        });
      });
    },

    fork() {
      this.$refs.editor.fork();
    },

    switchLights() {
      this.lights = this.$refs.editor.switchLights();
    },

    compile() {
      console.log("liascript: compile");

      if (this.preview && this.editorIsReady) {
        let code = this.$refs.editor.getValue();

        if (code.trim().length == 0) {
          code = tutorial;
        }

        this.preview.focusOnMain = false;
        this.preview.scrollUpOnMain = false;

        this.preview.jit(code);
      }
    },

    fetchError(src: string) {
      return this.$refs.editor.getBlob(src);
    },

    editorReady() {
      console.log("liascript: editor ready");
      this.editorIsReady = true;
      this.lights = this.$refs.editor.lights;
      this.compile();
    },

    previewReady(preview: any) {
      console.log("liascript: preview ready");
      this.preview = preview;
      this.compile();
     
      
    },

    gotoEditor(line: number) {
      if (this.$refs.editor) {
        this.$refs.editor.gotoLine(line);
      }
    },

    gotoPreview(line: number) {
      if (this.preview) this.preview.gotoLine(line);
    },

    previewUpdate(params: any) {
      console.log("liascript: update");
      

      // @ts-ignore
      window.LiascriptEditor ={
          value: this.$refs.editor.getValue(),
      }

      const iframe = document.getElementById(
        "liascript-preview"
      ) as HTMLIFrameElement;
      
      setTimeout(()=>{
      
        const navbar = iframe.contentDocument?.getElementById('lia-toolbar-nav') as HTMLElement;
        if(navbar)
          navbar.style.display="none"
        const searchbar = iframe.contentDocument?.querySelector('.lia-toc__search') as HTMLElement;
        if(searchbar)
          searchbar.style.display="none"
        const slides = iframe.contentDocument?.querySelectorAll('.lia-slide__content');
        slides?.forEach((s)=>s.style.margin=0)
        const container = iframe.contentDocument?.querySelectorAll('.lia-slide__container');
        container?.forEach((s)=>s.style.marginTop="20px")
        const toctoggle = iframe.contentDocument?.querySelector('#lia-toc button') as HTMLElement;

        const newRule1 = '.lia-toc--open button#lia-btn-toc { transform: translate(0rem, -100%)!important;}';
        const newRule2 = '.lia-toc button#lia-btn-toc { transform: translate(5rem, -80%)}';
        const sheet = iframe.contentDocument?.styleSheets[0];
        sheet?.insertRule(newRule1, sheet.cssRules.length);
        sheet?.insertRule(newRule2, sheet.cssRules.length);


        

      }, 50);
        

      
      this.compilationCounter++;
      if (this.compilationCounter > 1) {
        this.previewNotReady = false;

        if (this.$props.storageId) {
          const titleMatch = this.$refs.editor.getValue().match(/^# (.+)/m);

          if (titleMatch) params.title = titleMatch[1];

          this.database.put(this.$props.storageId, params);
        }
      }
    },
  },

  components: { Editor, Preview, Modal },

  mounted() {
    
    // Machen Sie die Methode global über `window` zugänglich
    // @ts-ignore
    window.updateEditorContent = this.updateEditorContent;
  },

};
</script>

<template>

  <nav class="navbar navbar-expand-lg bg-light">
    <div class="container-fluid">
      <!--
      <a
        class="navbar-brand"
        href="./"
        data-link="true"
      >
        <img
          src="../../assets/logo.png"
          alt="LiaScript"
          height="28"
        >
        LiaEdit
      </a>
      -->
      <button
        type="button"
        class="btn btn-outline-secondary me-2 px-3"
        @click="switchLights()"
        title="Switch between light and dark mode"
      >
        <i :class="lightMode"></i>
      </button>

      <div
        class="btn-toolbar btn-group-sm"
        role="toolbar"
        aria-label="Toolbar with button groups"
      >
        <div
          class="btn-group btn-outline-secondary me-2"
          role="group"
          aria-label="Basic radio toggle button group"
        >
          <input
            type="radio"
            class="btn-check"
            name="btnradio"
            id="btnradio1"
            autocomplete="off"
            @click="changeMode(-1)"
          >
          <label
            class="btn btn-outline-secondary"
            for="btnradio1"
            title="Editor only"
          >
            <i class="bi bi-pencil"></i>
          </label>

          <input
            type="radio"
            class="btn-check"
            name="btnradio"
            id="btnradio2"
            autocomplete="off"
            checked
            @click="changeMode(0)"
          >
          <label
            class="btn btn-outline-secondary"
            for="btnradio2"
            title="Split view"
          >
            <i class="bi bi-layout-split"></i>
          </label>

          <input
            type="radio"
            class="btn-check"
            name="btnradio"
            id="btnradio3"
            autocomplete="off"
            @click="changeMode(1)"
          >
          <label
            class="btn btn-outline-secondary"
            for="btnradio3"
            title="Preview only"
          >
            <i class="bi bi-eye"></i>
          </label>
        </div>
      </div>

      <button
        type="button"
        class="btn btn-outline-secondary me-2 px-3"
        @click="compile()"
        title="Compile (Ctrl+S)"
      >
        <i class="bi bi-arrow-counterclockwise"></i>
      </button>

      <!-- Drop-Down Navigation -->

      <button
        class="navbar-toggler"
        type="button"
        data-bs-toggle="collapse"
        data-bs-target="#navbarSupportedContent"
        aria-controls="navbarSupportedContent"
        aria-expanded="false"
        aria-label="Toggle navigation"
      >
        <span class="navbar-toggler-icon"></span>
      </button>

      <div
        class="collapse navbar-collapse"
        id="navbarSupportedContent"
      ></div>
    </div>
  </nav>

  <div
    class="container p-0"
    style="max-width:100%"
  >
    <div
      id="liascript-ide"
      class="row align-items-start p-0 m-0 flex-nowrap"
      style="height: calc(100vh - 56px);"
    >
      <div
        class="col-5 w-45 p-0 h-100"
        :hidden="mode > 0"
        :class="{'w-45': mode==0, 'w-100': mode!=0}"
        style="border-right: solid lightgray 2px;"
      >
        <Editor
          class="col w-100 p-0 h-100"
          @compile="compile"
          @ready="editorReady"
          @online="online"
          @goto="gotoPreview"
          :storage-id="$props.storageId"
          :content="$props.content"
          ref="editor"
          :connection="$props.connection"
        >
        </Editor>
      </div>

      <div
        class="col-7 w-55 h-100 p-0"
        :hidden="mode < 0"
        :class="{'w-55': mode==0, 'w-100': mode!=0}"
      >
        <div
          v-show="previewNotReady"
          style="position: absolute; background-color: white; width:50%; height: 100%"
        >
          <div
            class="spinner-grow"
            style="width: 5rem; height: 5rem; margin-top: 50%; margin-left: 45%; margin-right: 45%;"
            role="status"
          >
            <span class="visually-hidden">Loading...</span>
          </div>
        </div>

        <Preview
          @ready="previewReady"
          @update="previewUpdate"
          @goto="gotoEditor"
          :fetchError="fetchError"
          :class="{ invisible: previewNotReady }"
        />
      </div>
    </div>
  </div>

  <Modal ref="modal" />
</template>

<style scoped>
#liascript {
  height: 100vh;
}

.invisible {
  visibility: hidden;
}
</style>
