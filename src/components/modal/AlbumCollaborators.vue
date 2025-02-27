<template>
  <div class="manage-collaborators">
    <div class="manage-collaborators__subtitle">
      {{ t('photos', 'Add people or groups who can edit your album') }}
    </div>

    <form class="manage-collaborators__form" @submit.prevent>
      <NcPopover ref="popover" :auto-size="true" :distance="0">
        <label slot="trigger" class="manage-collaborators__form__input">
          <NcTextField
            :value.sync="searchText"
            autocomplete="off"
            type="search"
            name="search"
            :aria-label="t('photos', 'Search for collaborators')"
            aria-autocomplete="list"
            :aria-controls="`manage-collaborators__form__selection-${randomId} manage-collaborators__form__list-${randomId}`"
            :placeholder="t('photos', 'Search people or groups')"
            @input="searchCollaborators"
          >
            <Magnify :size="16" />
          </NcTextField>
          <NcLoadingIcon v-if="loadingCollaborators" />
        </label>

        <ul
          v-if="searchResults.length !== 0"
          :id="`manage-collaborators__form__list-${randomId}`"
          class="manage-collaborators__form__list"
        >
          <li v-for="collaboratorKey of searchResults" :key="collaboratorKey">
            <NcListItemIcon
              :id="availableCollaborators[collaboratorKey].id"
              class="manage-collaborators__form__list__result"
              :title="availableCollaborators[collaboratorKey].id"
              :search="searchText"
              :user="availableCollaborators[collaboratorKey].id"
              :display-name="availableCollaborators[collaboratorKey].label"
              :aria-label="
                t('photos', 'Add {collaboratorLabel} to the collaborators list', {
                  collaboratorLabel: availableCollaborators[collaboratorKey].label,
                })
              "
              @click="selectEntity(collaboratorKey)"
            />
          </li>
        </ul>
        <NcEmptyContent
          v-else
          key="emptycontent"
          class="manage-collaborators__form__list--empty"
          :title="t('photos', 'No collaborators available')"
        >
          <AccountGroup slot="icon" />
        </NcEmptyContent>
      </NcPopover>
    </form>

    <ul class="manage-collaborators__selection">
      <li
        v-for="collaboratorKey of listableSelectedCollaboratorsKeys"
        :key="collaboratorKey"
        class="manage-collaborators__selection__item"
      >
        <NcListItemIcon
          :id="availableCollaborators[collaboratorKey].id"
          :display-name="availableCollaborators[collaboratorKey].label"
          :title="availableCollaborators[collaboratorKey].id"
          :user="availableCollaborators[collaboratorKey].id"
        >
          <NcButton
            type="tertiary"
            :aria-label="
              t('photos', 'Remove {collaboratorLabel} from the collaborators list', {
                collaboratorLabel: availableCollaborators[collaboratorKey].label,
              })
            "
            @click="unselectEntity(collaboratorKey)"
          >
            <Close slot="icon" :size="20" />
          </NcButton>
        </NcListItemIcon>
      </li>
    </ul>

    <div class="actions">
      <div v-if="allowPublicLink" class="actions__public-link">
        <template v-if="isPublicLinkSelected">
          <NcButton
            class="manage-collaborators__public-link-button"
            :aria-label="t('photos', 'Copy the public link')"
            :disabled="publicLink.id === ''"
            @click="copyPublicLink"
          >
            <template v-if="publicLinkCopied">
              {{ t('photos', 'Public link copied!') }}
            </template>
            <template v-else>
              {{ t('photos', 'Copy public link') }}
            </template>
            <template #icon>
              <Check v-if="publicLinkCopied" />
              <ContentCopy v-else />
            </template>
          </NcButton>
          <NcButton
            type="tertiary"
            :aria-label="t('photos', 'Delete the public link')"
            :disabled="publicLink.id === ''"
            @click="deletePublicLink"
          >
            <NcLoadingIcon v-if="publicLink.id === ''" slot="icon" />
            <Close v-else slot="icon" />
          </NcButton>
        </template>
        <NcButton v-else class="manage-collaborators__public-link-button" @click="createPublicLinkForAlbum">
          <Earth slot="icon" />
          {{ t('photos', 'Share via public link') }}
        </NcButton>
      </div>

      <div class="actions__slot">
        <slot :collaborators="selectedCollaborators" />
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, PropType } from 'vue';

import Magnify from 'vue-material-design-icons/Magnify.vue';
import Close from 'vue-material-design-icons/Close.vue';
import Check from 'vue-material-design-icons/Check.vue';
import ContentCopy from 'vue-material-design-icons/ContentCopy.vue';
import AccountGroup from 'vue-material-design-icons/AccountGroup.vue';
import Earth from 'vue-material-design-icons/Earth.vue';

import axios from '@nextcloud/axios';
import * as dav from '../../services/DavRequests';
import { showError } from '@nextcloud/dialogs';
import { getCurrentUser } from '@nextcloud/auth';
import { generateOcsUrl, generateUrl } from '@nextcloud/router';

import NcButton from '@nextcloud/vue/dist/Components/NcButton';
import NcLoadingIcon from '@nextcloud/vue/dist/Components/NcLoadingIcon';
import NcPopover from '@nextcloud/vue/dist/Components/NcPopover';
import NcEmptyContent from '@nextcloud/vue/dist/Components/NcEmptyContent';
const NcTextField = () => import('@nextcloud/vue/dist/Components/NcTextField');
const NcListItemIcon = () => import('@nextcloud/vue/dist/Components/NcListItemIcon');

import { Type } from '@nextcloud/sharing';

type Collaborator = {
  id: string;
  label: string;
  type: Type;
};

export default defineComponent({
  name: 'AddToAlbumModal',
  components: {
    Magnify,
    Close,
    AccountGroup,
    ContentCopy,
    Check,
    Earth,
    NcLoadingIcon,
    NcButton,
    NcListItemIcon,
    NcTextField,
    NcPopover,
    NcEmptyContent,
  },

  props: {
    albumName: {
      type: String,
      required: true,
    },
    collaborators: {
      type: Array as PropType<Collaborator[]>,
      required: true,
    },
    allowPublicLink: {
      type: Boolean,
      required: false,
    },
  },

  data: () => ({
    searchText: '',
    availableCollaborators: {} as { [key: string]: Collaborator },
    selectedCollaboratorsKeys: [] as string[],
    currentSearchResults: [] as Collaborator[],
    loadingAlbum: false,
    errorFetchingAlbum: null as number | null,
    loadingCollaborators: false,
    errorFetchingCollaborators: null,
    randomId: Math.random().toString().substring(2, 10),
    publicLinkCopied: false,
    config: {
      minSearchStringLength: parseInt(window.OC.config['sharing.minSearchStringLength'], 10) || 0,
    },
  }),

  computed: {
    searchResults(): string[] {
      return this.currentSearchResults
        .filter(({ id }) => id !== getCurrentUser()?.uid)
        .map(({ type, id }) => `${type}:${id}`)
        .filter((collaboratorKey) => !this.selectedCollaboratorsKeys.includes(collaboratorKey));
    },

    listableSelectedCollaboratorsKeys(): string[] {
      return this.selectedCollaboratorsKeys.filter(
        (collaboratorKey) => this.availableCollaborators[collaboratorKey].type !== Type.SHARE_TYPE_LINK
      );
    },

    selectedCollaborators(): Collaborator[] {
      return this.selectedCollaboratorsKeys.map((collaboratorKey) => this.availableCollaborators[collaboratorKey]);
    },

    isPublicLinkSelected(): boolean {
      return this.selectedCollaboratorsKeys.includes(`${Type.SHARE_TYPE_LINK}`);
    },

    publicLink(): Collaborator {
      return this.availableCollaborators[Type.SHARE_TYPE_LINK];
    },
  },
  watch: {
    collaborators(collaborators) {
      this.populateCollaborators(collaborators);
    },
  },

  mounted() {
    this.searchCollaborators();
    this.populateCollaborators(this.collaborators);
  },

  methods: {
    /**
     * Fetch possible collaborators.
     */
    async searchCollaborators() {
      if (this.searchText.length >= 1) {
        (<any>this.$refs.popover).$refs.popover.show();
      }

      try {
        if (this.searchText.length < this.config.minSearchStringLength) {
          return;
        }

        this.loadingCollaborators = true;
        const response = await axios.get(generateOcsUrl('core/autocomplete/get'), {
          params: {
            search: this.searchText,
            itemType: 'share-recipients',
            shareTypes: [Type.SHARE_TYPE_USER, Type.SHARE_TYPE_GROUP],
          },
        });

        this.currentSearchResults = response.data.ocs.data.map((collaborator) => {
          switch (collaborator.source) {
            case 'users':
              return {
                id: collaborator.id,
                label: collaborator.label,
                type: Type.SHARE_TYPE_USER,
              };
            case 'groups':
              return {
                id: collaborator.id,
                label: collaborator.label,
                type: Type.SHARE_TYPE_GROUP,
              };
            default:
              throw new Error(`Invalid collaborator source ${collaborator.source}`);
          }
        });

        this.availableCollaborators = {
          ...this.availableCollaborators,
          ...this.currentSearchResults.reduce(this.indexCollaborators, {}),
        };
      } catch (error) {
        this.errorFetchingCollaborators = error;
        showError(this.t('photos', 'Failed to fetch collaborators list.'));
      } finally {
        this.loadingCollaborators = false;
      }
    },

    /**
     * Populate selectedCollaboratorsKeys and availableCollaborators.
     */
    populateCollaborators(collaborators: Collaborator[]) {
      const initialCollaborators = collaborators.reduce(this.indexCollaborators, {});
      this.selectedCollaboratorsKeys = Object.keys(initialCollaborators);
      this.availableCollaborators = {
        3: {
          id: '',
          label: this.t('photos', 'Public link'),
          type: Type.SHARE_TYPE_LINK,
        },
        ...this.availableCollaborators,
        ...initialCollaborators,
      };
    },

    /**
     * @param {Object<string, Collaborator>} collaborators - Index of collaborators
     * @param {Collaborator} collaborator - A collaborator
     */
    indexCollaborators(collaborators: { [s: string]: Collaborator }, collaborator: Collaborator) {
      return {
        ...collaborators,
        [`${collaborator.type}${collaborator.type === Type.SHARE_TYPE_LINK ? '' : ':'}${
          collaborator.type === Type.SHARE_TYPE_LINK ? '' : collaborator.id
        }`]: collaborator,
      };
    },

    async createPublicLinkForAlbum() {
      this.selectEntity(`${Type.SHARE_TYPE_LINK}`);
      await this.updateAlbumCollaborators();
      try {
        this.loadingAlbum = true;
        this.errorFetchingAlbum = null;

        const uid = getCurrentUser()?.uid.toString();
        if (!uid) return;
        const album = await dav.getAlbum(uid, this.albumName);
        this.populateCollaborators(album.collaborators);
      } catch (error) {
        if (error.response?.status === 404) {
          this.errorFetchingAlbum = 404;
        } else {
          this.errorFetchingAlbum = error;
        }

        showError(this.t('photos', 'Failed to fetch album.'));
      } finally {
        this.loadingAlbum = false;
      }
    },

    async deletePublicLink() {
      this.unselectEntity(`${Type.SHARE_TYPE_LINK}`);
      this.availableCollaborators[3] = {
        id: '',
        label: this.t('photos', 'Public link'),
        type: Type.SHARE_TYPE_LINK,
      };
      this.publicLinkCopied = false;
      await this.updateAlbumCollaborators();
    },

    async updateAlbumCollaborators() {
      try {
        const uid = getCurrentUser()?.uid?.toString();
        if (!uid) return;
        const album = await dav.getAlbum(uid, this.albumName);
        await dav.updateAlbum(album, {
          albumName: this.albumName,
          properties: {
            collaborators: this.selectedCollaborators,
          },
        });
      } catch (error) {
        showError(this.t('photos', 'Failed to update album.'));
      } finally {
        this.loadingAlbum = false;
      }
    },

    async copyPublicLink() {
      await navigator.clipboard.writeText(
        `${window.location.protocol}//${window.location.host}${generateUrl(`apps/memories/a/${this.publicLink.id}`)}`
      );
      this.publicLinkCopied = true;
      setTimeout(() => {
        this.publicLinkCopied = false;
      }, 10000);
    },

    selectEntity(collaboratorKey) {
      if (this.selectedCollaboratorsKeys.includes(collaboratorKey)) {
        return;
      }

      (<any>this.$refs.popover).$refs.popover.hide();
      this.selectedCollaboratorsKeys.push(collaboratorKey);
    },

    unselectEntity(collaboratorKey) {
      const index = this.selectedCollaboratorsKeys.indexOf(collaboratorKey);

      if (index === -1) {
        return;
      }

      this.selectedCollaboratorsKeys.splice(index, 1);
    },
  },
});
</script>
<style lang="scss" scoped>
.manage-collaborators {
  display: flex;
  flex-direction: column;
  height: 500px;

  &__title {
    font-weight: bold;
  }

  &__subtitle {
    color: var(--color-text-lighter);
  }

  &__public-link-button {
    margin: 4px 0;
  }

  &__form {
    margin-top: 4px 0;
    display: flex;
    flex-direction: column;

    &__input {
      position: relative;
      display: block;

      input {
        width: 100%;
        padding-left: 34px;
      }

      .loading-icon {
        position: absolute;
        top: calc(36px / 2 - 20px / 2);
        right: 8px;
      }
    }

    &__list {
      padding: 8px;
      height: 350px;
      overflow: scroll;

      &__result {
        padding: 8px;
        border-radius: 100px;
        box-sizing: border-box;

        &,
        & * {
          cursor: pointer !important;
        }

        &:hover {
          background: var(--color-background-dark);
        }
      }

      &--empty {
        margin: 100px 0;
      }
    }
  }

  &__selection {
    display: flex;
    flex-direction: column;
    margin-top: 8px;
    flex-grow: 1;

    &__item {
      border-radius: var(--border-radius-pill);
      padding: 0 8px;

      &:hover {
        background: var(--color-background-dark);
      }
    }
  }

  .actions {
    display: flex;
    margin-top: 8px;

    &__public-link {
      display: flex;
      align-items: center;

      button {
        margin-left: 8px;
      }
    }

    &__slot {
      flex-grow: 1;
      display: flex;
      justify-content: flex-end;
      align-items: center;
    }
  }
}
</style>
