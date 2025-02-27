<template>
  <div v-bind="themeDataAttr" ref="editor" class="viewer__image-editor" :class="{ loading: !imageEditor }" />
</template>

<script lang="ts">
import { defineComponent, PropType } from 'vue';

import { emit } from '@nextcloud/event-bus';
import { showError, showSuccess } from '@nextcloud/dialogs';
import axios from '@nextcloud/axios';

import { FilerobotImageEditorConfig } from 'react-filerobot-image-editor';

import translations from './ImageEditorTranslations';

import { API } from '../../services/API';
import { IImageInfo, IPhoto } from '../../types';
import * as utils from '../../services/Utils';
import { fetchImage } from '../frame/XImgCache';

let TABS, TOOLS: any;
type FilerobotImageEditor = import('filerobot-image-editor').default;
let FilerobotImageEditor: typeof import('filerobot-image-editor').default;

async function loadFilerobot() {
  if (!FilerobotImageEditor) {
    FilerobotImageEditor = (await import('filerobot-image-editor')).default;
    TABS = (<any>FilerobotImageEditor).TABS;
    TOOLS = (<any>FilerobotImageEditor).TOOLS;
  }
  return FilerobotImageEditor;
}

export default defineComponent({
  props: {
    photo: {
      type: Object as PropType<IPhoto>,
      required: true,
    },
  },

  data: () => ({
    exif: null as Object | null,
    imageEditor: null as FilerobotImageEditor | null,
  }),

  computed: {
    config(): FilerobotImageEditorConfig & { theme: any } {
      return {
        source:
          this.photo.h && this.photo.w
            ? utils.getPreviewUrl(this.photo, false, 'screen')
            : API.IMAGE_DECODABLE(this.photo.fileid, this.photo.etag),

        defaultSavedImageName: this.defaultSavedImageName,
        defaultSavedImageType: this.defaultSavedImageType,
        // We use our own translations
        useBackendTranslations: false,

        // Watch resize
        observePluginContainerSize: true,

        // Default tab and tool
        defaultTabId: TABS.ADJUST,
        defaultToolId: TOOLS.CROP,

        // Displayed tabs, disabling watermark and draw
        tabsIds: Object.values(TABS)
          .filter((tab) => ![TABS.WATERMARK, TABS.ANNOTATE].includes(tab))
          .sort((a: string, b: string) => a.localeCompare(b)) as any[],

        // onBeforeSave: this.onBeforeSave,
        onClose: this.onClose,
        // onModify: this.onModify,
        onSave: this.onSave,

        Rotate: {
          angle: 90,
          componentType: 'buttons',
        },

        // Translations
        translations,

        theme: {
          palette: {
            'bg-secondary': 'var(--color-main-background)',
            'bg-primary': 'var(--color-background-dark)',
            // Accent
            'accent-primary': 'var(--color-primary)',
            // Use by the slider
            'border-active-bottom': 'var(--color-primary)',
            'icons-primary': 'var(--color-main-text)',
            // Active state
            'bg-primary-active': 'var(--color-background-dark)',
            'bg-primary-hover': 'var(--color-background-hover)',
            'accent-primary-active': 'var(--color-main-text)',
            // Used by the save button
            'accent-primary-hover': 'var(--color-primary)',

            warning: 'var(--color-error)',
          },
          typography: {
            fontFamily: 'var(--font-face)',
          },
        },

        savingPixelRatio: window.devicePixelRatio,
        previewPixelRatio: window.devicePixelRatio,
      };
    },

    defaultSavedImageName(): string {
      return this.photo.basename || '';
    },

    defaultSavedImageType(): 'jpeg' | 'png' | 'webp' {
      if (['image/jpeg', 'image/png', 'image/webp'].includes(this.photo.mimetype!)) {
        return this.photo.mimetype!.split('/')[1] as any;
      }
      return 'jpeg';
    },

    hasHighContrastEnabled(): boolean {
      const themes = globalThis.OCA?.Theming?.enabledThemes || [];
      return themes.find((theme) => theme.indexOf('highcontrast') !== -1);
    },

    themeDataAttr(): Record<string, boolean> {
      if (this.hasHighContrastEnabled) {
        return {
          'data-theme-dark-highcontrast': true,
        };
      }
      return {
        'data-theme-dark': true,
      };
    },
  },

  async mounted() {
    await loadFilerobot();

    globalThis._fileRobotOverrideImage = await this.getImage();

    const div = <HTMLElement>this.$refs.editor;
    this.imageEditor = new FilerobotImageEditor(div, this.config);
    this.imageEditor.render();

    // Handle keyboard
    window.addEventListener('keydown', this.handleKeydown, true);
  },

  beforeDestroy() {
    if (this.imageEditor) {
      this.imageEditor.terminate();
    }
    globalThis._fileRobotOverrideImage = undefined;
    window.removeEventListener('keydown', this.handleKeydown, true);
  },

  methods: {
    async getImage(): Promise<HTMLImageElement> {
      const img = new Image();
      img.name = this.defaultSavedImageName;

      await new Promise(async (resolve) => {
        img.onload = resolve;
        img.src = await fetchImage(<string>this.config.source);
      });

      if (this.photo.w && this.photo.h) {
        img.height = this.photo.h;
        img.width = this.photo.w;
      }

      return img;
    },

    onClose(closingReason, haveNotSavedChanges) {
      if (haveNotSavedChanges) {
        this.onExitWithoutSaving();
        return;
      }
      window.removeEventListener('keydown', this.handleKeydown, true);
      this.$emit('close');
    },

    /**
     * User saved the image
     *
     * @see https://github.com/scaleflex/filerobot-image-editor#onsave
     */
    async onSave(
      data: {
        name: string;
        extension: string;
        width?: number;
        height?: number;
        quality?: number;
        fullName?: string;
        imageBase64?: string;
      },
      state: any
    ): Promise<void> {
      // Copy state
      state = JSON.parse(JSON.stringify(state));

      // Convert crop to relative values
      if (state?.adjustments?.crop) {
        const iw = state.shownImageDimensions.width;
        const ih = state.shownImageDimensions.height;
        const { x, y, width, height } = state.adjustments.crop;
        state.adjustments.crop = {
          x: x / iw,
          y: y / ih,
          width: width / iw,
          height: height / ih,
        };
      }

      // Make sure we have an extension
      let name = data.name;
      const nameLower = name.toLowerCase();
      if (!nameLower.endsWith(data.extension) && !nameLower.endsWith('.jpg')) {
        name += '.' + data.extension;
      }

      try {
        const res = await axios.put<IImageInfo>(API.IMAGE_EDIT(this.photo.fileid), {
          name: name,
          width: data.width,
          height: data.height,
          quality: data.quality,
          extension: data.extension,
          state: state,
        });
        const fileid = res.data.fileid;

        // Success, emit an appropriate event
        showSuccess(this.t('memories', 'Image saved successfully'));

        if (fileid !== this.photo.fileid) {
          emit('files:file:created', { fileid });
        } else {
          utils.updatePhotoFromImageInfo(this.photo, res.data);
          emit('files:file:updated', { fileid });
        }
        this.onClose(undefined, false);
      } catch (err) {
        showError(this.t('memories', 'Error saving image'));
        console.error(err);
      }
    },

    /**
     * Show warning if unsaved changes
     */
    onExitWithoutSaving() {
      (<any>OC.dialogs).confirmDestructive(
        translations.changesLoseConfirmation + '\n\n' + translations.changesLoseConfirmationHint,
        this.t('memories', 'Unsaved changes'),
        {
          type: (<any>OC.dialogs).YES_NO_BUTTONS,
          confirm: this.t('memories', 'Drop changes'),
          confirmClasses: 'error',
          cancel: translations.cancel,
        },
        (decision) => {
          if (!decision) {
            return;
          }
          this.onClose('warning-ignored', false);
        }
      );
    },

    // Key Handlers, override default Viewer arrow and escape key
    handleKeydown(event) {
      event.stopImmediatePropagation();
      // escape key
      if (event.key === 'Escape') {
        // Since we cannot call the closeMethod and know if there
        // are unsaved changes, let's fake a close button trigger.
        event.preventDefault();
        (document.querySelector('.FIE_topbar-close-button') as HTMLElement).click();
      }

      // ctrl + S = save
      if (event.ctrlKey && event.key === 's') {
        event.preventDefault();
        (document.querySelector('.FIE_topbar-save-button') as HTMLElement).click();
      }

      // ctrl + Z = undo
      if (event.ctrlKey && event.key === 'z') {
        event.preventDefault();
        (document.querySelector('.FIE_topbar-undo-button') as HTMLElement).click();
      }
    },
  },
});
</script>

<style lang="scss" scoped>
// Take full screen size ()
.viewer__image-editor {
  position: absolute;
  z-index: 10100;
  top: 0;
  left: 0;
  width: 100%;
  height: 100vh;
  background-color: black;
}
</style>

<style lang="scss">
// Make sure the editor and its modals are above everything
.SfxModal-Wrapper {
  z-index: 10101 !important;
}

.SfxPopper-wrapper {
  z-index: 10102 !important;
}

// Default styling
.viewer__image-editor,
.SfxModal-Wrapper,
.SfxPopper-wrapper {
  * {
    // Fix font size for the entire image editor
    font-size: var(--default-font-size) !important;
  }

  label,
  button {
    color: var(--color-main-text);
    > span {
      font-size: var(--default-font-size) !important;
    }
  }

  // Fix button ratio and center content
  button {
    display: flex;
    align-items: center;
    justify-content: center;
    min-width: 44px;
    min-height: 44px;
    padding: 6px 12px;
  }
}

// Input styling
.SfxInput-root {
  height: auto !important;
  padding: 0 !important;
  .SfxInput-Base {
    margin: 0 !important;
  }
}

// Select styling
.SfxSelect-root {
  padding: 8px !important;
}

// Global buttons
.SfxButton-root {
  min-height: 44px !important;
  margin: 0 !important;
  border: transparent !important;
  &[color='error'] {
    color: white !important;
    background-color: var(--color-error) !important;
    &:hover,
    &:focus {
      border-color: white !important;
      background-color: var(--color-error-hover) !important;
    }
  }
  &[color='primary'] {
    color: var(--color-primary-text) !important;
    background-color: var(--color-primary-element) !important;
    &:hover,
    &:focus {
      background-color: var(--color-primary-element-hover) !important;
    }
  }
}

// Menu items
.SfxMenuItem-root {
  height: 44px;
  padding-left: 8px !important;
  // Center the menu entry icon and fix width
  > div {
    margin-right: 0;
    padding: 14px;
    // Minus the parent padding-left
    padding: calc(14px - 8px);
    cursor: pointer;
  }

  // Disable jpeg saving (jpg is already here)
  &[value='jpeg'] {
    display: none;
  }
}

// Modal
.SfxModal-Container {
  min-height: 300px;
  padding: 22px;

  // Fill height
  .SfxModal-root,
  .SfxModalTitle-root {
    flex: 1 1 100%;
    justify-content: center;
    color: var(--color-main-text);
  }
  .SfxModalTitle-Icon {
    margin-bottom: 22px !important;
    background: none !important;
    // Fit EmptyContent styling
    svg {
      width: 64px;
      height: 64px;
      opacity: 0.4;
      // Override all coloured icons

      --color-primary: var(--color-main-text);
      --color-error: var(--color-main-text);
    }
  }
  // Hide close icon (use cancel button)
  .SfxModalTitle-Close {
    display: none !important;
  }
  // Modal actions buttons display
  .SfxModalActions-root {
    justify-content: space-evenly !important;
  }
}

// Header buttons
.FIE_topbar-center-options > button,
.FIE_topbar-center-options > label {
  margin-left: 6px !important;
}

// Tabs
.FIE_tabs {
  padding: 6px !important;
  overflow: hidden;
  overflow-y: auto;
}

.FIE_tab {
  width: 80px !important;
  height: 80px !important;
  padding: 8px;
  border-radius: var(--border-radius-large) !important;
  svg {
    width: 16px;
    height: 16px;
  }
  &-label {
    margin-top: 8px !important;
    overflow: hidden;
    text-overflow: ellipsis;
    max-width: 100%;
    white-space: nowrap;
    display: block !important;
  }

  &:hover,
  &:focus {
    background-color: var(--color-background-hover) !important;
  }

  &[aria-selected='true'] {
    color: var(--color-main-text);
    background-color: var(--color-background-dark);
    box-shadow: 0 0 0 2px var(--color-primary-element);
  }
}

// Tools bar
.FIE_tools-bar {
  &-wrapper {
    max-height: max-content !important;
  }

  // Matching buttons tools
  & > div[class$='-tool-button'],
  & > div[class$='-tool'] {
    display: flex;
    align-items: center;
    justify-content: center;
    min-width: 44px;
    height: 44px;
    padding: 6px 16px;
    border-radius: var(--border-radius-pill);
  }
}

// Crop preset select button
.FIE_crop-presets-opener-button {
  // override default button width
  min-width: 0 !important;
  padding: 5px !important;
  padding-left: 10px !important;
  border: none !important;
  background-color: transparent !important;
}

// Force icon-only style
.FIE_topbar-history-buttons button,
.FIE_topbar-close-button,
.FIE_resize-ratio-locker {
  border: none !important;
  background-color: transparent !important;

  &:hover,
  &:focus {
    background-color: var(--color-background-hover) !important;
  }

  svg {
    width: 16px;
    height: 16px;
  }
}

// Left top bar buttons
.FIE_topbar-history-buttons button {
  &.FIE_topbar-reset-button {
    &::before {
      content: attr(title);
      font-weight: normal;
    }
    svg {
      display: none;
    }
  }
}

// Save button fixes
.FIE_topbar-save-button {
  color: var(--color-primary-text) !important;
  border: none !important;
  background-color: var(--color-primary-element) !important;
  &:hover,
  &:focus {
    background-color: var(--color-primary-element-hover) !important;
  }
}

// Save Modal fixes
.FIE_resize-tool-options {
  .FIE_resize-width-option,
  .FIE_resize-height-option {
    flex: 1 1;
    min-width: 0;
  }
}

// Resize lock
.FIE_resize-ratio-locker {
  margin-right: 8px !important;
  // Icon is very thin
  svg {
    width: 20px;
    height: 20px;
    path {
      stroke-width: 1;
      stroke: var(--color-main-text);
      fill: var(--color-main-text);
    }
  }
}

// Close editor button fixes
.FIE_topbar-close-button {
  svg path {
    // The path viewbox is weird and
    // not correct, this fixes it
    transform: scale(1.6);
  }
}

// Canvas container
.FIE_canvas-container {
  background-color: var(--color-main-background) !important;
}

// Loader
.FIE_spinner::after,
.FIE_spinner-label {
  display: none !important;
}

.FIE_spinner-wrapper {
  background-color: transparent !important;
}

.FIE_spinner::before {
  position: absolute;
  z-index: 2;
  top: 50%;
  left: 50%;
  width: 28px;
  height: 28px;
  margin: -16px 0 0 -16px;
  content: '';
  -webkit-transform-origin: center;
  -ms-transform-origin: center;
  transform-origin: center;
  -webkit-animation: rotate 0.8s infinite linear;
  animation: rotate 0.8s infinite linear;
  border: 2px solid var(--color-loading-light);
  border-top-color: var(--color-loading-dark);
  border-radius: 100%;

  filter: var(--background-invert-if-dark);
}

.FIE_carousel-prev-button,
.FIE_carousel-next-button {
  background: none !important;
}
</style>
