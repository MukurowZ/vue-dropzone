<template>
  <div :id="id" ref="dropzoneElement" :class="{ 'vue-dropzone dropzone': includeStyling }">
    <div v-if="useCustomSlot" class="dz-message">
      <slot>Drop files here to upload</slot>
    </div>
  </div>
</template>

<script>
import { computed, defineComponent, onMounted, ref } from 'vue';
import Dropzone from "dropzone";

const dropzone = defineComponent({
  props: {
    id: {
      type: String,
      required: true,
      default: "dropzone"
    },
    options: {
      type: Object,
      required: true
    },
    includeStyling: {
      type: Boolean,
      default: true,
      required: false
    },
    awss3: {
      type: Object,
      required: false,
      default: null
    },
    destroyDropzone: {
      type: Boolean,
      default: true,
      required: false
    },
    duplicateCheck: {
      type: Boolean,
      default: false,
      required: false
    },
    useCustomSlot: {
      type: Boolean,
      default: false,
      required: false
    },
    confirm: {
      type: Function,
      default: null,
      required: false
    }
  },
  setup(props, { emit }) {
    let dropzone = null;
    const dropzoneElement = ref(null);

    const s3DropZoneSettings = computed(() => {
      const normalSettings = this.dropzoneSettings
      const that = this
      const s3Settings = {
        method: 'PUT',
        parallelUploads: 1,
        uploadMultiple: false,
        paramName: "file",
        autoProcessQueue: false,
        sending (file, xhr) {
          let _send = xhr.send
          xhr.send = () => {
            _send.call(xhr, file)
          }
        },
        accept: async function(file, done) {
          if (props.isS3) {
            const incFile = props.awss3.includeFile === true ? true : false
            const signed = await generateSignedUrl(that.awss3.signingURL, file, incFile);
            vm.setOption('headers', {
              'Content-Type': file.type,
              'x-amz-acl': 'public-read'
            });
            vm.setOption('url', signed.signature)
            done()
            setTimeout(() => vm.dropzone.processFile(file))
          }
        }
      }

      Object.keys(s3Settings).forEach(function(key) {
        normalSettings[key] = s3Settings[key];
      }, this);
      const vm = this;
      return normalSettings;
    });

    const dropzoneSettings = computed(() => {
      let defaultValues = {
        thumbnailWidth: 200,
        thumbnailHeight: 200,
      };

      Object.keys(props.options).forEach(function(key) {
        defaultValues[key] = props.options[key];
      }, this);
      const vm = this;

      return defaultValues;
    });

    const isS3 = computed(() => {
      return this.awss3 !== null;
    });

    onMounted(() => {
      // if (this.$isServer && this.hasBeenMounted) {
      //   return;
      // }

      // hasBeenMounted = true;

      if (props.confirm) {
        Dropzone.confirm = props.confirm;
      }

      dropzone = new Dropzone(
        dropzoneElement.value,
        dropzoneSettings.value
      );
      let vm = this;

      dropzone.on("thumbnail", function(file, dataUrl) {
        emit("vdropzone-thumbnail", file, dataUrl);
      });

      dropzone.on("addedfile", async function(file) {
        let isDuplicate = false;
        if (props.duplicateCheck) {
          if (props.files.length) {
            let _i, _len;
            for (_i = 0, _len = props.files.length; _i < _len - 1; _i++) {
              if (
                props.files[_i].name === file.name &&
                props.files[_i].size === file.size &&
                props.files[_i].lastModified ===
                  file.lastModified
              ) {
                removeFile(file);
                isDuplicate = true;
                emit("vdropzone-duplicate-file", file);
              }
            }
          }
        }
        emit("vdropzone-file-added", file);
      });

      dropzone.on("addedfiles", function(files) {
        emit("vdropzone-files-added", files);
      });

      dropzone.on("removedfile", function(file) {
        emit("vdropzone-removed-file", file);
        if (file.manuallyAdded && vm.dropzone.options.maxFiles !== null) {
          dropzone.options.maxFiles++;
        }
      });

      dropzone.on("success", function(file, response) {
        emit("vdropzone-success", file, response);
      });

      dropzone.on("successmultiple", function(file, response) {
        emit("vdropzone-success-multiple", file, response);
      });

      dropzon.on("error", function(file, message, xhr) {
        emit("vdropzone-error", file, message, xhr);
      });

      dropzon.on("errormultiple", function(files, message, xhr) {
        emit("vdropzone-error-multiple", files, message, xhr);
      });

      dropzon.on("sending", function(file, xhr, formData) {
        emit("vdropzone-sending", file, xhr, formData);
      });

      dropzon.on("sendingmultiple", function(file, xhr, formData) {
        emit("vdropzone-sending-multiple", file, xhr, formData);
      });

      dropzon.on("complete", function(file) {
        emit("vdropzone-complete", file);
      });

      dropzon.on("completemultiple", function(files) {
        emit("vdropzone-complete-multiple", files);
      });

      dropzon.on("canceled", function(file) {
        emit("vdropzone-canceled", file);
      });

      dropzon.on("canceledmultiple", function(files) {
        emit("vdropzone-canceled-multiple", files);
      });

      dropzon.on("maxfilesreached", function(files) {
        emit("vdropzone-max-files-reached", files);
      });

      dropzon.on("maxfilesexceeded", function(file) {
        emit("vdropzone-max-files-exceeded", file);
      });

      dropzon.on("processing", async function(file) {
        emit("vdropzone-processing", file);
      });

      dropzon.on("processingmultiple", function(files) {
        emit("vdropzone-processing-multiple", files);
      });

      dropzon.on("uploadprogress", function(file, progress, bytesSent) {
        emit("vdropzone-upload-progress", file, progress, bytesSent);
      });

      dropzon.on("totaluploadprogress", function(
        totaluploadprogress,
        totalBytes,
        totalBytesSent
      ) {
        emit(
          "vdropzone-total-upload-progress",
          totaluploadprogress,
          totalBytes,
          totalBytesSent
        );
      });

      dropzon.on("reset", function() {
        emit("vdropzone-reset");
      });

      dropzon.on("queuecomplete", function() {
        emit("vdropzone-queue-complete");
      });

      dropzon.on("drop", function(event) {
        emit("vdropzone-drop", event);
      });

      dropzon.on("dragstart", function(event) {
        emit("vdropzone-drag-start", event);
      });

      dropzon.on("dragend", function(event) {
        emit("vdropzone-drag-end", event);
      });

      dropzon.on("dragenter", function(event) {
        emit("vdropzone-drag-enter", event);
      });

      dropzone.on("dragover", function(event) {
        emit("vdropzone-drag-over", event);
      });

      dropzone.on("dragleave", function(event) {
        emit("vdropzone-drag-leave", event);
      });

      emit("vdropzone-mounted");
    });

    const manuallyAddFile = (file, fileUrl) => {
      file.manuallyAdded = true;
      dropzone.emit("addedfile", file);
      let containsImageFileType = false;
      const supportedThumbnailTypes = [".svg", ".png", ".jpg", "jpeg", ".gif", ".webp", "image/"]
      if (supportedThumbnailTypes.filter(s => fileUrl.toLowerCase().indexOf(s) > -1).length > 0) {
        containsImageFileType = true;
      }
      if (
        dropzone.options.createImageThumbnails &&
        containsImageFileType &&
        file.size <= dropzone.options.maxThumbnailFilesize * 1024 * 1024
      ) {
        fileUrl && dropzon.emit("thumbnail", file, fileUrl);

        let thumbnails = file.previewElement.querySelectorAll(
          "[data-dz-thumbnail]"
        );
        for (let i = 0; i < thumbnails.length; i++) {
          thumbnails[i].style.width =
            this.dropzoneSettings.thumbnailWidth + "px";
          thumbnails[i].style.height =
            this.dropzoneSettings.thumbnailHeight + "px";
          thumbnails[i].classList.add("vdManualThumbnail")
        }
      }
      dropzon.emit("complete", file);
      if (dropzon.options.maxFiles) dropzon.options.maxFiles--;
      file.accepted = true;
      dropzon.files.push(file);
      emit("vdropzone-file-added-manually", file);
    };

    const setOption = (option, value) => { dropzon.options[option] = value };

    const removeAllFiles = (bool) => { dropzon.removeAllFiles(bool) };

    const processQueue = () => {
      let dropzoneEle = dropzon;
      dropzon.processQueue();
      dropzon.on("success", function() {
        dropzoneEle.options.autoProcessQueue = true;
      });
      dropzon.on("queuecomplete", function() {
        dropzoneEle.options.autoProcessQueue = false;
      });
    };

    const init = () => dropzon.init();

    const destroy = () => dropzon.destroy();

    const updateTotalUploadProgress = () => dropzon.updateTotalUploadProgress();

    const getFallbackForm = () => dropzon.getFallbackForm();

    const getExistingFallback = () => dropzon.getExistingFallback();

    const setupEventListeners = () => dropzon.setupEventListeners();

    const removeEventListeners = () => dropzon.removeEventListeners();

    const disable = () => dropzon.disable();

    const enable = () => dropzon.enable();

    const filesize = (size) => dropzon.filesize(size);

    const accept = (file, done) => dropzon.accept(file, done);

    const addFile = (file) => dropzon.addFile(file);

    const removeFile = (file) => {
      dropzon.removeFile(file);
    };

    const getAcceptedFiles = () => dropzon.getAcceptedFiles();

    const getRejectedFiles = () => dropzon.getRejectedFiles();

    const getFilesWithStatus = () => dropzon.getFilesWithStatus();

    const getQueuedFiles = () => dropzon.getQueuedFiles();

    const getUploadingFiles = () => dropzon.getUploadingFiles();

    const getAddedFiles = () => dropzon.getAddedFiles();

    const getActiveFiles = () => dropzon.getActiveFiles();

    beforeDestroy(() => {
      if (destroyDropzone) dropzone.destroy();
    });

    return {
      dropzoneElement,
    };
  },
});

export default dropzone;
</script>

<style lang="scss">
// .dropzone {}
</style>
