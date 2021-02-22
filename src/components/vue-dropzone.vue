<template>
  <div :id="id" ref="dropzoneElement" :class="{ 'vue-dropzone dropzone': includeStyling }">
    <div v-if="useCustomSlot" class="dz-message">
      <slot>Drop files here to upload</slot>
    </div>
  </div>
</template>

<script>
import Dropzone from 'dropzone';

Dropzone.autoDiscover = false;

export default {
  props: {
    id: {
      type: String,
      required: true,
      default: 'dropzone',
    },
    options: {
      type: Object,
      required: true,
    },
    includeStyling: {
      type: Boolean,
      default: true,
      required: false,
    },
    destroyDropzone: {
      type: Boolean,
      default: true,
      required: false,
    },
    duplicateCheck: {
      type: Boolean,
      default: false,
      required: false,
    },
    useCustomSlot: {
      type: Boolean,
      default: false,
      required: false,
    },
    confirm: {
      type: Function,
      default: null,
      required: false,
    },
  },
  computed: {
    dropzoneSettings() {
      const defaultValues = {
        thumbnailWidth: 200,
        thumbnailHeight: 200,
      };

      Object.keys(this.options).forEach((key) => {
        defaultValues[key] = this.options[key];
      }, this);

      return defaultValues;
    },
  },
  mounted() {
    if (this.$isServer && this.hasBeenMounted) {
      return;
    }

    this.hasBeenMounted = true;

    if (this.confirm) {
      Dropzone.confirm = this.confirm;
    }

    this.dropzone = new Dropzone(
      this.$refs.dropzoneElement,
      this.dropzoneSettings,
    );
    // const vm = this;

    this.dropzone.on('thumbnail', (file, dataUrl) => {
      this.$emit('vdropzone-thumbnail', file, dataUrl);
    });

    this.dropzone.on('addedfile', async (file) => {
      let isDuplicate = false;
      if (this.duplicateCheck) {
        if (this.files.length) {
          let _i;
          let _len;
          for (_i = 0, _len = this.files.length; _i < _len - 1; _i += 1) {
            if (
              this.files[_i].name === file.name
              && this.files[_i].size === file.size
              && this.files[_i].lastModified === file.lastModified
            ) {
              this.removeFile(file);
              isDuplicate = true;
              this.$emit('vdropzone-duplicate-file', file);
            }
          }
        }
      }
      this.$emit('vdropzone-file-added', file);
    });

    this.dropzone.on('addedfiles', (files) => {
      this.$emit('vdropzone-files-added', files);
    });

    this.dropzone.on('removedfile', (file) => {
      this.$emit('vdropzone-removed-file', file);
      if (file.manuallyAdded && this.dropzone.options.maxFiles !== null) {
        this.dropzone.options.maxFiles += 1;
      }
    });

    this.dropzone.on('success', (file, response) => {
      this.$emit('vdropzone-success', file, response);
    });

    this.dropzone.on('successmultiple', (file, response) => {
      this.$emit('vdropzone-success-multiple', file, response);
    });

    this.dropzone.on('error', (file, message, xhr) => {
      this.$emit('vdropzone-error', file, message, xhr);
    });

    this.dropzone.on('errormultiple', (files, message, xhr) => {
      this.$emit('vdropzone-error-multiple', files, message, xhr);
    });

    this.dropzone.on('sending', (file, xhr, formData) => {
      this.$emit('vdropzone-sending', file, xhr, formData);
    });

    this.dropzone.on('sendingmultiple', (file, xhr, formData) => {
      this.$emit('vdropzone-sending-multiple', file, xhr, formData);
    });

    this.dropzone.on('complete', (file) => {
      this.$emit('vdropzone-complete', file);
    });

    this.dropzone.on('completemultiple', (files) => {
      this.$emit('vdropzone-complete-multiple', files);
    });

    this.dropzone.on('canceled', (file) => {
      this.$emit('vdropzone-canceled', file);
    });

    this.dropzone.on('canceledmultiple', (files) => {
      this.$emit('vdropzone-canceled-multiple', files);
    });

    this.dropzone.on('maxfilesreached', (files) => {
      this.$emit('vdropzone-max-files-reached', files);
    });

    this.dropzone.on('maxfilesexceeded', (file) => {
      this.$emit('vdropzone-max-files-exceeded', file);
    });

    this.dropzone.on('processing', async (file) => {
      this.$emit('vdropzone-processing', file);
    });

    this.dropzone.on('processingmultiple', (files) => {
      this.$emit('vdropzone-processing-multiple', files);
    });

    this.dropzone.on('uploadprogress', (file, progress, bytesSent) => {
      this.$emit('vdropzone-upload-progress', file, progress, bytesSent);
    });

    this.dropzone.on('totaluploadprogress', (
      totaluploadprogress,
      totalBytes,
      totalBytesSent,
    ) => {
      this.$emit(
        'vdropzone-total-upload-progress',
        totaluploadprogress,
        totalBytes,
        totalBytesSent,
      );
    });

    this.dropzone.on('reset', () => {
      this.$emit('vdropzone-reset');
    });

    this.dropzone.on('queuecomplete', () => {
      this.$emit('vdropzone-queue-complete');
    });

    this.dropzone.on('drop', (event) => {
      this.$emit('vdropzone-drop', event);
    });

    this.dropzone.on('dragstart', (event) => {
      this.$emit('vdropzone-drag-start', event);
    });

    this.dropzone.on('dragend', (event) => {
      this.$emit('vdropzone-drag-end', event);
    });

    this.dropzone.on('dragenter', (event) => {
      this.$emit('vdropzone-drag-enter', event);
    });

    this.dropzone.on('dragover', (event) => {
      this.$emit('vdropzone-drag-over', event);
    });

    this.dropzone.on('dragleave', (event) => {
      this.$emit('vdropzone-drag-leave', event);
    });

    this.$emit('vdropzone-mounted');
  },
  beforeUnmount() {
    if (this.destroyDropzone) this.dropzone.destroy();
  },
  methods: {
    manuallyAddFile(file, fileUrl) {
      // eslint-disable-next-line
      file.manuallyAdded = true;
      this.dropzone.emit('addedfile', file);
      let containsImageFileType = false;
      const supportedThumbnailTypes = ['.svg', '.png', '.jpg', 'jpeg', '.gif', '.webp', 'image/'];
      if (supportedThumbnailTypes.filter((s) => fileUrl.toLowerCase().indexOf(s) > -1).length > 0) {
        containsImageFileType = true;
      }
      if (
        this.dropzone.options.createImageThumbnails
        && containsImageFileType
        && file.size <= this.dropzone.options.maxThumbnailFilesize * 1024 * 1024
      ) {
        // eslint-disable-next-line
        fileUrl && this.dropzone.emit('thumbnail', file, fileUrl);

        const thumbnails = file.previewElement.querySelectorAll('[data-dz-thumbnail]');
        for (let i = 0; i < thumbnails.length; i += 1) {
          thumbnails[i].style.width = `${this.dropzoneSettings.thumbnailWidth}px`;
          thumbnails[i].style.height = `${this.dropzoneSettings.thumbnailHeight}px`;
          thumbnails[i].classList.add('vdManualThumbnail');
        }
      }
      this.dropzone.emit('complete', file);
      if (this.dropzone.options.maxFiles) this.dropzone.options.maxFiles -= 1;
      // eslint-disable-next-line
      file.accepted = true;
      this.dropzone.files.push(file);
      this.$emit('vdropzone-file-added-manually', file);
    },
    setOption(option, value) {
      this.dropzone.options[option] = value;
    },
    removeAllFiles(bool) {
      this.dropzone.removeAllFiles(bool);
    },
    processQueue() {
      this.dropzone.processQueue();
      this.dropzone.on('success', () => {
        this.dropzone.options.autoProcessQueue = true;
      });
      this.dropzone.on('queuecomplete', () => {
        this.dropzone.options.autoProcessQueue = false;
      });
    },
    init() {
      return this.dropzone.init();
    },
    destroy() {
      return this.dropzone.destroy();
    },
    updateTotalUploadProgress() {
      return this.dropzone.updateTotalUploadProgress();
    },
    getFallbackForm() {
      return this.dropzone.getFallbackForm();
    },
    getExistingFallback() {
      return this.dropzone.getExistingFallback();
    },
    setupEventListeners() {
      return this.dropzone.setupEventListeners();
    },
    removeEventListeners() {
      return this.dropzone.removeEventListeners();
    },
    disable() {
      return this.dropzone.disable();
    },
    enable() {
      return this.dropzone.enable();
    },
    filesize(size) {
      return this.dropzone.filesize(size);
    },
    accept(file, done) {
      return this.dropzone.accept(file, done);
    },
    addFile(file) {
      return this.dropzone.addFile(file);
    },
    removeFile(file) {
      this.dropzone.removeFile(file);
    },
    getAcceptedFiles() {
      return this.dropzone.getAcceptedFiles();
    },
    getRejectedFiles() {
      return this.dropzone.getRejectedFiles();
    },
    getFilesWithStatus() {
      return this.dropzone.getFilesWithStatus();
    },
    getQueuedFiles() {
      return this.dropzone.getQueuedFiles();
    },
    getUploadingFiles() {
      return this.dropzone.getUploadingFiles();
    },
    getAddedFiles() {
      return this.dropzone.getAddedFiles();
    },
    getActiveFiles() {
      return this.dropzone.getActiveFiles();
    },
  },
};
</script>

<style lang="scss">
.vue-dropzone {
  border: 2px solid #e5e5e5;
  font-family: "Arial", sans-serif;
  letter-spacing: 0.2px;
  color: #777;
  transition: 0.2s linear;
}

.vue-dropzone:hover {
  background-color: #f6f6f6;
}

.vue-dropzone > i {
  color: #ccc;
}

.vue-dropzone > .dz-preview .dz-image {
  border-radius: 0;
  width: 100%;
  height: 100%;
}

.vue-dropzone > .dz-preview .dz-image img:not([src]) {
  width: 200px;
  height: 200px;
}

.vue-dropzone > .dz-preview .dz-image:hover img {
  transform: none;
  -webkit-filter: none;
}

.vue-dropzone > .dz-preview .dz-details {
  bottom: 0;
  top: 0;
  color: white;
  background-color: rgba(33, 150, 243, 0.8);
  transition: opacity 0.2s linear;
  text-align: left;
}

.vue-dropzone > .dz-preview .dz-details .dz-filename {
  overflow: hidden;
}

.vue-dropzone > .dz-preview .dz-details .dz-filename span,
.vue-dropzone > .dz-preview .dz-details .dz-size span {
  background-color: transparent;
}

.vue-dropzone > .dz-preview .dz-details .dz-filename:not(:hover) span {
  border: none;
}

.vue-dropzone > .dz-preview .dz-details .dz-filename:hover span {
  background-color: transparent;
  border: none;
}

.vue-dropzone > .dz-preview .dz-progress .dz-upload {
  background: #cccccc;
}

.vue-dropzone > .dz-preview .dz-remove {
  position: absolute;
  z-index: 30;
  color: white;
  margin-left: 15px;
  padding: 10px;
  top: inherit;
  bottom: 15px;
  border: 2px white solid;
  text-decoration: none;
  text-transform: uppercase;
  font-size: 0.8rem;
  font-weight: 800;
  letter-spacing: 1.1px;
  opacity: 0;
}

.vue-dropzone > .dz-preview:hover .dz-remove {
  opacity: 1;
}

.vue-dropzone > .dz-preview .dz-success-mark,
.vue-dropzone > .dz-preview .dz-error-mark {
  margin-left: auto;
  margin-top: auto;
  width: 100%;
  top: 35%;
  left: 0;
}

.vue-dropzone > .dz-preview .dz-success-mark svg,
.vue-dropzone > .dz-preview .dz-error-mark svg {
  margin-left: auto;
  margin-right: auto;
}

.vue-dropzone > .dz-preview .dz-error-message {
  margin-left: auto;
  margin-right: auto;
  left: 0;
  width: 100%;
  text-align: center;
}

.vue-dropzone > .dz-preview .dz-error-message:after {
  display: none;
}
.vue-dropzone > .dz-preview .vdManualThumbnail {
  object-fit: cover;
}
</style>
