<template>
  <div class="outer">
    <div class="lat-lon">
      <div class="coords">
        <span>{{ loc }}</span> {{ dirty ? '*' : '' }}
      </div>

      <div class="action">
        <NcActions :inline="2">
          <NcActionButton v-if="dirty" :aria-label="t('memories', 'Reset')" @click="reset()">
            {{ t('memories', 'Reset') }}
            <template #icon> <UndoIcon :size="20" /> </template>
          </NcActionButton>

          <NcActionButton v-if="lat && lon" :aria-label="t('memories', 'Remove location')" @click="clear()">
            {{ t('memories', 'Remove location') }}
            <template #icon> <CloseIcon :size="20" /> </template>
          </NcActionButton>
        </NcActions>
      </div>
    </div>

    <NcTextField
      :value.sync="searchBar"
      :placeholder="t('memories', 'Search location / landmark')"
      trailing-button-icon="arrowRight"
      :show-trailing-button="searchBar.length > 0 && !loading"
      @trailing-button-click="search"
      @keypress.enter="search"
    >
      <MagnifyIcon :size="16" />
    </NcTextField>

    <div class="osm-attribution">
      Powered by
      <a href="https://nominatim.openstreetmap.org" target="_blank">Nominatim</a>
      &copy;
      <a href="https://www.openstreetmap.org/copyright" target="_blank">OpenStreetMap</a>
      contributors
    </div>

    <NcLoadingIcon class="loading-spinner" v-if="loading" />

    <ul v-if="options.length > 0">
      <li
        v-for="option in options"
        :key="option.osm_id"
        @click="select(option)"
        @keypress.enter="select(option)"
        tabindex="0"
      >
        {{ option.display_name }}
      </li>
    </ul>
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import { IPhoto } from '../../types';

import axios from '@nextcloud/axios';
import { showError } from '@nextcloud/dialogs';

import NcActions from '@nextcloud/vue/dist/Components/NcActions';
import NcActionButton from '@nextcloud/vue/dist/Components/NcActionButton';
const NcTextField = () => import('@nextcloud/vue/dist/Components/NcTextField');
const NcListItem = () => import('@nextcloud/vue/dist/Components/NcListItem');
import NcLoadingIcon from '@nextcloud/vue/dist/Components/NcLoadingIcon';

import MagnifyIcon from 'vue-material-design-icons/Magnify.vue';
import CloseIcon from 'vue-material-design-icons/Close.vue';
import UndoIcon from 'vue-material-design-icons/UndoVariant.vue';

type NLocation = {
  osm_id: number;
  type: string;
  icon: string;
  display_name: string;
  lat: string;
  lon: string;
};

export default defineComponent({
  components: {
    NcActions,
    NcActionButton,
    NcTextField,
    NcListItem,
    NcLoadingIcon,
    MagnifyIcon,
    CloseIcon,
    UndoIcon,
  },

  props: {
    photos: {
      type: Array<IPhoto>,
      required: true,
    },
  },

  data: () => ({
    dirty: false,
    lat: null as number | null,
    lon: null as number | null,
    searchBar: '',
    loading: false,

    options: [] as NLocation[],
  }),

  computed: {
    loc() {
      if (this.lat && this.lon) {
        return `${this.lat.toFixed(6)}, ${this.lon.toFixed(6)}`;
      }
      return this.t('memories', 'No coordinates');
    },
  },

  mounted() {
    this.reset();
  },

  methods: {
    reset() {
      this.dirty = false;
      const photos = this.photos as IPhoto[];

      let lat = 0,
        lon = 0,
        count = 0;
      for (const photo of photos) {
        const exif = photo.imageInfo?.exif;
        if (!exif) {
          continue;
        }

        if (exif.GPSLatitude && exif.GPSLongitude) {
          lat += Number(exif.GPSLatitude);
          lon += Number(exif.GPSLongitude);
          count++;
        }
      }

      if (count > 0) {
        this.lat = lat / count;
        this.lon = lon / count;
      }
    },

    search() {
      if (this.loading || this.searchBar.length === 0) {
        return;
      }

      this.loading = true;
      const q = window.encodeURIComponent(this.searchBar);
      axios
        .get(`https://nominatim.openstreetmap.org/search.php?q=${q}&format=jsonv2`)
        .then((response) => {
          this.loading = false;
          this.options = response.data.filter((x: NLocation) => {
            return x.lat && x.lon && x.display_name;
          });
        })
        .catch((error) => {
          this.loading = false;
          console.error(error);
          showError(this.t('memories', 'Failed to search for location with Nominatim.'));
        });
    },

    clear() {
      this.dirty = true;
      this.lat = 0;
      this.lon = 0;
    },

    select(option: NLocation) {
      this.dirty = true;
      this.lat = Number(option.lat);
      this.lon = Number(option.lon);
      this.options = [];
      this.searchBar = '';
    },

    result() {
      if (!this.dirty) return null;

      const lat = (this.lat || 0).toFixed(6);
      const lon = (this.lon || 0).toFixed(6);

      return {
        GPSLatitude: lat,
        GPSLongitude: lon,
        GPSLatitudeRef: lat,
        GPSLongitudeRef: lon,
        GPSCoordinates: `${lat}, ${lon}`,
      };
    },
  },
});
</script>

<style scoped lang="scss">
.outer {
  .lat-lon {
    display: flex;
    padding: 4px;
    margin-bottom: -10px;

    > .coords {
      display: inline-block;
      flex-grow: 1;
      min-height: 36px;

      > span {
        user-select: all;
      }
    }

    > .action {
      margin-top: -10px;
      margin-left: 2px;
      > * {
        cursor: pointer;
      }
    }
  }

  .osm-attribution {
    margin: -3px 4px;
    font-size: 0.65em;
    a {
      color: var(--color-primary);
    }
  }

  .loading-spinner {
    margin: 10px;
  }

  ul {
    margin: 10px 0;
    max-height: 200px;
    overflow-y: auto;

    li {
      font-size: 0.9em;
      padding: 5px 10px;
      margin: 2px 0;
      cursor: pointer;

      &:hover {
        background-color: var(--color-background-hover);
      }
    }
  }
}
</style>
