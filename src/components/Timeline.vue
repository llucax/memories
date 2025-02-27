<template>
  <div class="container" ref="container" :class="{ 'icon-loading': loading }">
    <!-- Static top matter -->
    <TopMatter ref="topmatter" />

    <!-- No content found and nothing is loading -->
    <EmptyContent v-if="!loading && empty" />

    <!-- Main recycler view for rows -->
    <RecycleScroller
      ref="recycler"
      class="recycler hide-scrollbar"
      :class="{ empty }"
      :items="list"
      :emit-update="true"
      :buffer="800"
      :skipHover="true"
      key-field="id"
      size-field="size"
      type-field="type"
      :updateInterval="100"
      @update="scrollChange"
      @resize="handleResizeWithDelay"
    >
      <template #before>
        <!-- Show dynamic top matter, name of the view -->
        <div class="recycler-before" ref="recyclerBefore">
          <div class="text" v-show="!$refs.topmatter.type && list.length && viewName">
            {{ viewName }}
          </div>

          <OnThisDay
            v-if="routeIsBase && config_enableTopMemories"
            :key="config_timelinePath"
            :viewer="$refs.viewer"
            @load="scrollerManager.adjust()"
          >
          </OnThisDay>

          <FolderGrid v-if="folders.length" :items="folders" />
        </div>
      </template>

      <template v-slot="{ item, index }">
        <RowHead
          v-if="item.type === 0"
          :item="item"
          :monthView="isMonthView"
          @click="selectionManager.selectHead(item)"
        />

        <Photo
          class="photo"
          v-else
          v-for="photo of item.photos"
          :key="photo.key"
          :style="{
            height: `${photo.dispH}px`,
            width: `${photo.dispW}px`,
            transform: `translate(${photo.dispX}px, ${photo.dispY}px)`,
          }"
          :data="photo"
          :day="item.day"
          @select="selectionManager.selectPhoto"
          @pointerdown="selectionManager.clickPhoto(photo, $event, index)"
          @touchstart="selectionManager.touchstartPhoto(photo, $event, index)"
          @touchend="selectionManager.touchendPhoto(photo, $event, index)"
          @touchmove="selectionManager.touchmovePhoto(photo, $event, index)"
        />
      </template>
    </RecycleScroller>

    <!-- Managers -->
    <ScrollerManager
      ref="scrollerManager"
      :rows="list"
      :height="scrollerHeight"
      :recycler="$refs.recycler"
      :recyclerBefore="$refs.recyclerBefore"
      @interactend="loadScrollView()"
    />

    <SelectionManager
      ref="selectionManager"
      :heads="heads"
      :rows="list"
      :isreverse="isMonthView"
      :recycler="$refs.recycler?.$el"
      @refresh="softRefresh"
      @delete="deleteFromViewWithAnimation"
      @updateLoading="updateLoading"
    />

    <Viewer ref="viewer" @deleted="deleteFromViewWithAnimation" @fetchDay="fetchDay" @updateLoading="updateLoading" />
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue';

import axios from '@nextcloud/axios';
import { showError } from '@nextcloud/dialogs';
import { subscribe, unsubscribe } from '@nextcloud/event-bus';

import { getLayout } from '../services/Layout';
import { IDay, IFolder, IHeadRow, IPhoto, IRow, IRowType } from '../types';

import UserConfig from '../mixins/UserConfig';
import FolderGrid from './FolderGrid.vue';
import RowHead from './frame/RowHead.vue';
import Photo from './frame/Photo.vue';
import ScrollerManager from './ScrollerManager.vue';
import SelectionManager from './SelectionManager.vue';
import Viewer from './viewer/Viewer.vue';

import EmptyContent from './top-matter/EmptyContent.vue';
import OnThisDay from './top-matter/OnThisDay.vue';
import TopMatter from './top-matter/TopMatter.vue';

import * as dav from '../services/DavRequests';
import * as utils from '../services/Utils';
import * as strings from '../services/strings';

import { API, DaysFilterType } from '../services/API';

const SCROLL_LOAD_DELAY = 250; // Delay in loading data when scrolling
const DESKTOP_ROW_HEIGHT = 200; // Height of row on desktop
const MOBILE_ROW_HEIGHT = 120; // Approx row height on mobile
const ROW_NUM_LPAD = 16; // Number of rows to load before and after viewport

export default defineComponent({
  name: 'Timeline',

  components: {
    FolderGrid,
    RowHead,
    Photo,
    EmptyContent,
    OnThisDay,
    TopMatter,
    SelectionManager,
    ScrollerManager,
    Viewer,
  },

  mixins: [UserConfig],

  data: () => ({
    /** Loading days response */
    loading: 0,
    /** Main list of rows */
    list: [] as IRow[],
    /** List of top folders */
    folders: [] as IFolder[],
    /** Computed number of columns */
    numCols: 0,
    /** Header rows for dayId key */
    heads: {} as { [dayid: number]: IHeadRow },

    /** Size of outer container [w, h] */
    containerSize: [0, 0] as [number, number],
    /** Computed row height */
    rowHeight: 100,
    /** Computed row width */
    rowWidth: 100,

    /** Current start index */
    currentStart: 0,
    /** Current end index */
    currentEnd: 0,
    /** Resizing timer */
    resizeTimer: null as number | null,
    /** Height of the scroller */
    scrollerHeight: 100,

    /** Set of dayIds for which images loaded */
    loadedDays: new Set<number>(),
    /** Set of dayIds for which image size is calculated */
    sizedDays: new Set<number>(),
    /** Days to load in the next call */
    fetchDayQueue: [] as number[],
    /** Timer to load day call */
    fetchDayTimer: null as number | null,

    /** State for request cancellations */
    state: Math.random(),

    /** Selection manager component */
    selectionManager: null as InstanceType<typeof SelectionManager> & any,
    /** Scroller manager component */
    scrollerManager: null as InstanceType<typeof ScrollerManager> & any,
  }),

  mounted() {
    this.selectionManager = <any>this.$refs.selectionManager;
    this.scrollerManager = <any>this.$refs.scrollerManager;
    this.routeChange(this.$route);
  },

  watch: {
    async $route(to: any, from?: any) {
      await this.routeChange(to, from);
    },
  },

  created() {
    subscribe(this.config_eventName, this.softRefresh);
    subscribe('files:file:created', this.softRefresh);
    subscribe('memories:window:resize', this.handleResizeWithDelay);
  },

  beforeDestroy() {
    unsubscribe(this.config_eventName, this.softRefresh);
    unsubscribe('files:file:created', this.softRefresh);
    unsubscribe('memories:window:resize', this.handleResizeWithDelay);
    this.resetState();
  },

  computed: {
    routeIsBase(): boolean {
      return this.$route.name === 'timeline';
    },
    routeIsPeople(): boolean {
      return ['recognize', 'facerecognition'].includes(<string>this.$route.name);
    },
    routeIsArchive(): boolean {
      return this.$route.name === 'archive';
    },
    routeIsFolders(): boolean {
      return this.$route.name === 'folders';
    },
    isMonthView(): boolean {
      if (this.$route.query.sort === 'timeline') return false;

      return (
        this.$route.query.sort === 'album' ||
        (this.config_sortAlbumMonth && (this.$route.name === 'albums' || this.$route.name === 'album-share')) ||
        (this.config_sortFolderMonth && this.$route.name === 'folders')
      );
    },

    /** Get view name for dynamic top matter */
    viewName(): string {
      return strings.viewName(this.$route.name);
    },

    /** Nothing to show here */
    empty(): boolean {
      return !this.list.length && !this.folders.length;
    },
  },

  methods: {
    async routeChange(to: any, from?: any) {
      // Always do a hard refresh if the path changes
      if (from?.path !== to.path) {
        await this.refresh();
      }

      // Do a soft refresh if the query changes
      else if (JSON.stringify(from.query) !== JSON.stringify(to.query)) {
        await this.softRefresh();
      }

      // The viewer might change the route immediately again
      await this.$nextTick();

      // Check if hash has changed
      const viewerIsOpen = (this.$refs.viewer as any)?.isOpen;
      if (from?.hash !== to.hash && to.hash?.startsWith('#v') && !viewerIsOpen) {
        // Open viewer
        const parts = to.hash.split('/');
        if (parts.length !== 3) return;

        // Get params
        const dayid = parseInt(parts[1]);
        const key = parts[2];
        if (isNaN(dayid) || !key) return;

        // Get day
        const day = this.heads[dayid]?.day;
        if (day && !day.detail) {
          const state = this.state;
          await this.fetchDay(dayid, true);
          if (state !== this.state) return;
        }

        // Find photo
        const photo = day?.detail?.find((p) => p.key === key);
        if (!photo) return;

        // Scroll to photo if initializing
        if (!from) {
          const index = this.list.findIndex((r) => r.day.dayid === dayid && r.photos?.includes(photo));
          if (index !== -1) {
            (this.$refs.recycler as any).scrollToItem(index);
          }
        }

        (this.$refs.viewer as any).open(photo, this.list);
      } else if (from?.hash?.startsWith('#v') && !to.hash?.startsWith('#v') && viewerIsOpen) {
        // Close viewer
        (this.$refs.viewer as any).close();
      }
    },
    updateLoading(delta: number) {
      this.loading += delta;
    },

    isMobile() {
      return this.containerSize[0] <= 768;
    },

    isMobileLayout() {
      return this.containerSize[0] <= 600;
    },

    allowBreakout() {
      return globalThis.windowInnerWidth <= 600 && !this.config_squareThumbs;
    },

    /** Create new state */
    async createState() {
      // Wait for one tick before doing anything
      await this.$nextTick();

      // Fit to window
      this.recomputeSizes();

      // Timeline recycler init
      (this.$refs.recycler as any).$el.addEventListener('scroll', this.scrollPositionChange, { passive: true });

      // Get data
      await this.fetchDays();
    },

    /** Reset all state */
    async resetState() {
      this.selectionManager.clearSelection();
      this.loading = 0;
      this.list = [];
      this.folders = [];
      this.heads = {};
      this.currentStart = 0;
      this.currentEnd = 0;
      this.scrollerManager.reset();
      this.state = Math.random();
      this.loadedDays.clear();
      this.sizedDays.clear();
      this.fetchDayQueue = [];
      window.clearTimeout(this.fetchDayTimer ?? 0);
      window.clearTimeout(this.resizeTimer ?? 0);
    },

    /** Recreate everything */
    async refresh() {
      await this.resetState();
      await this.createState();
    },

    /** Re-process days */
    async softRefresh() {
      this.selectionManager.clearSelection();
      this.fetchDayQueue = []; // reset queue
      await this.fetchDays(true);
    },

    /** Do resize after some time */
    handleResizeWithDelay() {
      utils.setRenewingTimeout(this, 'resizeTimer', this.recomputeSizes, 100);
    },

    /** Recompute static sizes of containers */
    recomputeSizes() {
      // Size of outer container
      const e = this.$refs.container as Element;
      const height = e.clientHeight;
      const width = e.clientWidth;
      this.containerSize = [width, height];

      // Scroller spans the container height
      this.scrollerHeight = height;

      // Static top matter to exclude from recycler height
      const topmatter = this.$refs.topmatter as any;
      const tmHeight = topmatter.$el?.clientHeight || 0;

      // Recycler height
      const recycler = this.$refs.recycler as any;
      const targetHeight = height - tmHeight - 4;
      const targetWidth = this.isMobile() ? width : width - 40;
      const heightChanged = recycler.$el.clientHeight !== targetHeight;
      const widthChanged = this.rowWidth !== targetWidth;

      if (heightChanged) {
        recycler.$el.style.height = targetHeight + 'px';
      }

      if (widthChanged) {
        this.rowWidth = targetWidth;
      }

      if (!heightChanged && !widthChanged) {
        // If the target size is the same, nothing else could have
        // possibly changed either, so just skip
        return;
      }

      if (this.isMobileLayout()) {
        // Mobile
        this.numCols = Math.max(3, Math.floor(this.rowWidth / MOBILE_ROW_HEIGHT));
        this.rowHeight = Math.floor(this.rowWidth / this.numCols);
      } else {
        // Desktop
        if (this.config_squareThumbs) {
          this.numCols = Math.max(3, Math.floor(this.rowWidth / DESKTOP_ROW_HEIGHT));
          this.rowHeight = Math.floor(this.rowWidth / this.numCols);
        } else {
          // As a heuristic, assume all images are 4:3 landscape
          this.rowHeight = DESKTOP_ROW_HEIGHT;
          this.numCols = Math.ceil(this.rowWidth / ((this.rowHeight * 4) / 3));
        }
      }

      // Reflow if there are elements (this isn't an init call)
      // An init call reaches here when the top matter size changes
      if (this.list.length > 0) {
        // At this point we're sure the size has changed, so we need
        // to invalidate everything related to sizes
        this.sizedDays.clear();
        this.scrollerManager.adjust();

        // Explicitly request a scroll event
        this.loadScrollView();
      }
    },

    /**
     * Triggered when position of scroll change.
     * This does NOT indicate the items have changed, only that
     * the pixel position of the recycler has changed.
     */
    scrollPositionChange(event?: any) {
      this.scrollerManager.recyclerScrolled(event);
    },

    /** Trigger when recycler view changes */
    scrollChange(startIndex: number, endIndex: number) {
      if (startIndex === this.currentStart && endIndex === this.currentEnd) {
        return;
      }

      // Reset placeholder state for rows including padding
      const rmin = Math.max(0, startIndex - ROW_NUM_LPAD);
      const rmax = Math.min(this.list.length, endIndex + ROW_NUM_LPAD);
      for (let i = rmin; i < rmax; i++) {
        const row = this.list[i];
        if (!row) {
          continue;
        }

        // Initialize photos and add placeholders
        if (row.pct && !row.photos?.length) {
          row.photos = new Array(row.pct);
          for (let j = 0; j < row.pct; j++) {
            // Any row that has placeholders has ONLY placeholders
            // so we can calculate the display width
            row.photos[j] = {
              flag: this.c.FLAG_PLACEHOLDER,
              fileid: Math.random(),
              dayid: row.dayId,
              dispW: utils.roundHalf(this.rowWidth / this.numCols),
              dispX: utils.roundHalf((j * this.rowWidth) / this.numCols),
              dispH: this.rowHeight,
              dispY: 0,
            };
          }
        }

        // No need for the fake count regardless of what happened above
        delete row.pct;
      }

      // Check if this was requested by a refresh
      const force = this.currentEnd === -1;

      // We only need to debounce loads if the user is dragging the scrollbar
      const scrolling = this.scrollerManager.interacting;

      // Make sure we don't do this too often
      this.currentStart = startIndex;
      this.currentEnd = endIndex;

      // Check if we can do this immediately
      const delay = force || !scrolling ? 0 : SCROLL_LOAD_DELAY;

      // Debounce; only execute the newest call after delay
      utils.setRenewingTimeout(this, '_scrollChangeTimer', this.loadScrollView, delay);
    },

    /** Load image data for given view (index based) */
    loadScrollView(startIndex?: number, endIndex?: number) {
      // Default values if not defined
      startIndex ??= this.currentStart;
      endIndex ??= this.currentEnd;

      // Check if any side needs a padding.
      // Whenever less than half rows are loaded, we need to pad with full
      // rows on that side. This ensures we have minimal reflows.
      const rmin = Math.max(0, startIndex - ROW_NUM_LPAD / 2);
      const rmax = Math.min(this.list.length - 1, endIndex + ROW_NUM_LPAD / 2);
      const notsized = (r: IRow) => r && !this.sizedDays.has(r.dayId);

      // Check at the start
      if (this.list.slice(rmin, startIndex).some(notsized)) {
        startIndex -= ROW_NUM_LPAD;
      }

      // Check at the end
      if (this.list.slice(endIndex + 1, rmax + 1).some(notsized)) {
        endIndex += ROW_NUM_LPAD;
      }

      // Make sure start and end valid
      startIndex = Math.max(0, startIndex);
      endIndex = Math.min(this.list.length - 1, endIndex);

      // Fetch all visible days
      for (let i = startIndex; i <= endIndex; i++) {
        const item = this.list[i];
        if (!item) continue;
        if (this.loadedDays.has(item.dayId)) {
          if (!this.sizedDays.has(item.dayId)) {
            // Just quietly reflow without refetching
            this.processDay(item.dayId, item.day.detail!);
          }
          continue;
        }

        this.fetchDay(item.dayId);
      }
    },

    /** Get query string for API calls */
    getQuery() {
      const query: { [key: string]: string } = {};

      // Favorites
      if (this.$route.name === 'favorites') {
        API.DAYS_FILTER(query, DaysFilterType.FAVORITES);
      }

      // Videos
      if (this.$route.name === 'videos') {
        API.DAYS_FILTER(query, DaysFilterType.VIDEOS);
      }

      // Folder
      if (this.$route.name === 'folders') {
        const path = utils.getFolderRoutePath(this.config_foldersPath);
        API.DAYS_FILTER(query, DaysFilterType.FOLDER, path);
        if (this.$route.query.recursive) {
          API.DAYS_FILTER(query, DaysFilterType.RECURSIVE);
        }
      }

      // Archive
      if (this.$route.name === 'archive') {
        API.DAYS_FILTER(query, DaysFilterType.ARCHIVE);
      }

      // Albums
      const user = <string>this.$route.params.user;
      const name = <string>this.$route.params.name;
      if (this.$route.name === 'albums') {
        if (!user || !name) {
          throw new Error('Invalid album route');
        }
        API.DAYS_FILTER(query, DaysFilterType.ALBUM, `${user}/${name}`);
      }

      // People
      if (this.routeIsPeople) {
        if (!user || !name) {
          throw new Error('Invalid album route');
        }

        const filter = <DaysFilterType>this.$route.name;
        API.DAYS_FILTER(query, filter, `${user}/${name}`);

        // Face rect
        if (this.config_showFaceRect) {
          API.DAYS_FILTER(query, DaysFilterType.FACE_RECT);
        }
      }

      // Places
      if (this.$route.name === 'places') {
        if (!name || !name.includes('-')) {
          throw new Error('Invalid place route');
        }

        const id = <string>name.split('-', 1)[0];
        API.DAYS_FILTER(query, DaysFilterType.PLACE, id);
      }

      // Tags
      if (this.$route.name === 'tags') {
        if (!name) {
          throw new Error('Invalid tag route');
        }
        API.DAYS_FILTER(query, DaysFilterType.TAG, name);
      }

      // Map Bounds
      if (this.$route.name === 'map') {
        const bounds = <string>this.$route.query.b;
        if (!bounds) {
          throw new Error('Missing map bounds');
        }

        API.DAYS_FILTER(query, DaysFilterType.MAP_BOUNDS, bounds);
      }

      // Month view
      if (this.isMonthView) {
        API.DAYS_FILTER(query, DaysFilterType.MONTH_VIEW);
        API.DAYS_FILTER(query, DaysFilterType.REVERSE);
      }

      return query;
    },

    /** Fetch folders */
    async fetchFolders() {
      if (!this.routeIsFolders || this.$route.query.recursive) {
        this.folders = [];
        return;
      }

      // Get subfolders URL
      const folder = utils.getFolderRoutePath(this.config_foldersPath);
      const url = API.Q(API.FOLDERS_SUB(), { folder });

      // Make API call to get subfolders
      try {
        this.loading++;
        const state = this.state;
        const res = await axios.get<IFolder[]>(url);
        if (state !== this.state) return;
        this.folders = res.data;
      } finally {
        this.loading--;
      }

      // Filter out hidden folders
      if (!this.config_showHidden) {
        this.folders = this.folders.filter((f) => !f.name.startsWith('.') && f.previews?.length);
      }
    },

    /** Fetch timeline main call */
    async fetchDays(noCache = false) {
      // Awaiting this is important because the folders must render
      // before the timeline to prevent glitches
      await this.fetchFolders();

      // Get URL an cache identifier
      let url: string;
      try {
        url = API.Q(API.DAYS(), this.getQuery());
      } catch (err) {
        // Likely invalid route; just quit doing anything
        return;
      }

      // URL for cached data
      const cacheUrl = <string>this.$route.name + url;

      // Try cache first
      let cache: IDay[] | null = null;

      // Make sure to refresh scroll later
      this.currentEnd = -1;

      try {
        this.loading++;
        const startState = this.state;

        let data: IDay[] = [];
        if (this.$route.name === 'thisday') {
          data = await dav.getOnThisDayData();
        } else if (dav.isSingleItem()) {
          data = await dav.getSingleItemData();
          this.$router.replace(utils.getViewerRoute(data[0]!.detail![0]));
        } else {
          // Try the cache
          if (!noCache) {
            try {
              if ((cache = await utils.getCachedData(cacheUrl))) {
                await this.processDays(cache);
                this.loading--;
              }
            } catch {
              console.warn(`Failed to process days cache: ${cacheUrl}`);
              cache = null;
            }
          }

          // Get from network
          const res = await axios.get<IDay[]>(url);
          if (res.status !== 200) throw res; // don't cache this
          data = res.data;
        }

        // Put back into cache
        utils.cacheData(cacheUrl, data);

        // Make sure we're still on the same page
        if (this.state !== startState) return;
        await this.processDays(data);
      } catch (err) {
        console.error(err);
        showError(err?.response?.data?.message || err.message);
      } finally {
        // If cache is set here, loading was already decremented
        if (!cache) this.loading--;
      }
    },

    /** Process the data for days call including folders */
    async processDays(data: IDay[]) {
      const list: typeof this.list = [];
      const heads: typeof this.heads = {};

      // Store the preloads in a separate map.
      // This is required since otherwise the inner detail objects
      // do not become reactive (which happens only after assignment).
      const preloads: {
        [dayId: number]: {
          day: IDay;
          detail: IPhoto[];
        };
      } = {};

      let prevDay: IDay | null = null;
      for (const day of data) {
        // Initialization
        day.rows = [];

        // Nothing here
        if (day.count === 0) {
          continue;
        }

        // Store the preloads
        if (day.detail) {
          preloads[day.dayid] = {
            day: day,
            detail: day.detail,
          };
          delete day.detail;
        }

        // Create header for this day
        const head: IHeadRow = {
          id: `${day.dayid}-head`,
          num: -1,
          size: 40,
          type: IRowType.HEAD,
          selected: false,
          dayId: day.dayid,
          day: day,
        };

        // Special headers
        if (this.$route.name === 'thisday' && (!prevDay || Math.abs(prevDay.dayid - day.dayid) > 30)) {
          // thisday view with new year title
          head.size = 67;
          head.super = utils.getFromNowStr(utils.dayIdToDate(day.dayid));
        }

        // Add header to list
        heads[day.dayid] = head;
        list.push(head);

        // Dummy rows for placeholders
        let nrows = Math.ceil(day.count / this.numCols);

        // Check if already loaded - we can learn
        const prevRows = this.heads[day.dayid]?.day?.rows;
        nrows = prevRows?.length || nrows;

        // Add rows
        for (let i = 0; i < nrows; i++) {
          const row = this.addRow(day);
          list.push(row);

          // Add placeholder count
          const leftNum = day.count - i * this.numCols;
          row.pct = Math.max(0, Math.min(this.numCols, leftNum));
          row.photos = [];

          // Learn from existing row
          if (prevRows && i < prevRows.length && !prevRows[i].pct) {
            row.size = prevRows[i].size;
            row.photos = prevRows[i].photos;
            delete row.pct;
          }
        }

        // Continue processing
        prevDay = day;
      }

      // Store globally
      this.list = list;
      this.heads = heads;
      this.loadedDays.clear();
      this.sizedDays.clear();

      // Iterate the preload map
      // Now the inner detail objects are reactive
      for (const dayId in preloads) {
        this.processDay(Number(dayId), preloads[dayId].detail);
      }

      // Notify parent components about stats
      this.$emit('daysLoaded', {
        count: data.reduce((acc, day) => acc + day.count, 0),
      });

      // Fix view height variable
      await this.scrollerManager.reflow();
      this.scrollPositionChange();
    },

    /** API url for Day call */
    getDayUrl(dayId: number | string) {
      return API.Q(API.DAY(dayId), this.getQuery());
    },

    /** Fetch image data for one dayId */
    async fetchDay(dayId: number, now = false) {
      const head = this.heads[dayId];
      if (!head) return;

      // Do this in advance to prevent duplicate requests
      this.loadedDays.add(dayId);
      this.sizedDays.add(dayId);

      // Look for cache
      const cacheUrl = this.getDayUrl(dayId);
      try {
        const cache = await utils.getCachedData<IPhoto[]>(cacheUrl);
        if (cache) this.processDay(dayId, cache);
      } catch {
        console.warn(`Failed to process day cache: ${cacheUrl}`);
      }

      // Aggregate fetch requests
      this.fetchDayQueue.push(dayId);

      // Only single queries allowed for month vie
      if (now || this.isMonthView) {
        return this.fetchDayExpire();
      }

      // Defer for aggregation
      if (!this.fetchDayTimer) {
        this.fetchDayTimer = window.setTimeout(() => {
          this.fetchDayTimer = null;
          this.fetchDayExpire();
        }, 150);
      }
    },

    async fetchDayExpire() {
      if (this.fetchDayQueue.length === 0) return;

      // Construct URL
      const dayStr = this.fetchDayQueue.join(',');
      const url = this.getDayUrl(dayStr);
      this.fetchDayQueue = [];

      try {
        const startState = this.state;
        const res = await axios.get<IPhoto[]>(url);
        if (res.status !== 200) throw res;
        const data = res.data;

        // Check if the state has changed
        if (this.state !== startState || this.getDayUrl(dayStr) !== url) {
          return;
        }

        // Bin the data into separate days
        // It is already sorted in dayid DESC
        const dayMap = new Map<number, IPhoto[]>();
        for (const photo of data) {
          if (!dayMap.has(photo.dayid)) dayMap.set(photo.dayid, []);
          dayMap.get(photo.dayid)!.push(photo);
        }

        // Store cache asynchronously
        // Do this regardless of whether the state has
        // changed since the data is already fetched
        //
        // These loops cannot be combined because processDay
        // creates circular references which cannot be stringified
        for (const [dayId, photos] of dayMap) {
          utils.cacheData(this.getDayUrl(dayId), photos);
        }

        // Process each day as needed
        for (const [dayId, photos] of dayMap) {
          // Check if the response has any delta
          const head = this.heads[dayId];
          if (head?.day?.detail?.length) {
            if (
              head.day.detail.length === photos.length &&
              head.day.detail.every((p, i) => p.fileid === photos[i].fileid && p.etag === photos[i].etag)
            ) {
              continue;
            }
          }

          // Pass ahead
          this.processDay(dayId, photos);
        }
      } catch (e) {
        showError(this.t('memories', 'Failed to load some photos'));
        console.error(e);
      }
    },

    /**
     * Process items from day response.
     *
     * @param dayId id of day
     * @param data photos
     */
    processDay(dayId: number, data: IPhoto[]) {
      if (!data) return;

      const head = this.heads[dayId];
      if (!head) return;

      const day = head.day;
      this.loadedDays.add(dayId);
      this.sizedDays.add(dayId);

      // Convert server flags to bitflags
      data.forEach(utils.convertFlags);

      // Set and make reactive
      day.count = data.length;
      day.detail = data;
      day.rows ??= [];

      // Reset rows including placeholders
      for (const row of day.rows) {
        row.photos = [];
      }

      // Force all to square
      const squareMode = this.isMobileLayout() || this.config_squareThumbs;

      // Create justified layout with correct params
      const justify = getLayout(
        day.detail.map((p) => {
          return {
            width: p.w || this.rowHeight,
            height: p.h || this.rowHeight,
            forceSquare: false,
          };
        }),
        {
          rowWidth: this.rowWidth,
          rowHeight: this.rowHeight,
          squareMode: squareMode,
          numCols: this.numCols,
          allowBreakout: this.allowBreakout(),
          seed: dayId,
        }
      );

      // Check if some rows were added
      let addedRows: IRow[] = [];

      // Recycler scroll top
      let scrollTop = (<any>this.$refs.recycler).$el.scrollTop;
      let needAdjust = false;

      // Get index and Y position of header in O(n)
      let headIdx = 0;
      let headY = 0;
      for (const row of this.list) {
        if (row === head) break;
        headIdx++;
        headY += row.size;
      }
      let rowIdx = headIdx + 1;
      let rowY = headY + head.size;

      // Duplicate detection, e.g. for face rects
      const seen = new Map<number, number>();

      // Previous justified row
      let prevJustifyTop = justify[0]?.top || 0;

      // Add all rows
      let dataIdx = 0;
      while (dataIdx < data.length) {
        // Check if we ran out of rows
        if (rowIdx >= this.list.length || this.list[rowIdx].type === IRowType.HEAD) {
          const newRow = this.addRow(day);
          addedRows.push(newRow);
          this.list.splice(rowIdx, 0, newRow);

          // Scroll down if new row is above the current visible position
          if (rowY < scrollTop) {
            scrollTop += newRow.size;
          }
          needAdjust = true;
        }

        // Get row
        const row = this.list[rowIdx];

        // Go to the next row
        const jbox = justify[dataIdx];
        if (jbox.top !== prevJustifyTop) {
          prevJustifyTop = jbox.top;
          rowIdx++;
          rowY += row.size;
          continue;
        }

        // Set row height
        const jH = utils.roundHalf(jbox.rowHeight || jbox.height);
        const delta = jH - row.size;
        // If the difference is too small, it's not worth risking an adjustment
        // especially on square layouts on mobile. Also don't do this if animating.
        if (Math.abs(delta) > 0) {
          if (rowY < scrollTop) {
            scrollTop += delta;
          }
          needAdjust = true;
          row.size = jH;
        }

        // Add the photo to the row
        const photo = data[dataIdx];
        photo.d = day; // backref to day

        // Get aspect ratio
        const setPos = () => {
          photo.dispW = utils.roundHalf(jbox.width);
          photo.dispX = utils.roundHalf(jbox.left);
          photo.dispH = utils.roundHalf(jbox.height);
          photo.dispY = 0;
          photo.dispRowNum = row.num;
        };
        if (photo.dispW !== undefined) {
          // photo already displayed: animate
          window.setTimeout(setPos, 50);

          if (
            photo.dispRowNum !== undefined &&
            photo.dispRowNum !== row.num &&
            photo.dispRowNum >= 0 &&
            photo.dispRowNum < day.rows.length
          ) {
            // Row change animation
            const start = Math.min(photo.dispRowNum, row.num);
            const end = Math.max(photo.dispRowNum, row.num);
            const sizeDelta = day.rows.slice(start, end).reduce((acc, r) => {
              acc += r.size;
              return acc;
            }, 0);
            photo.dispY = sizeDelta * (photo.dispRowNum < row.num ? -1 : 1);
            photo.dispH = day.rows[photo.dispRowNum].size;
          }
        } else {
          setPos();
        }

        // Move to next index of photo
        dataIdx++;

        // Duplicate detection.
        // These may be valid, e.g. in face rects. All we need to have
        // is a unique Vue key for the v-for loop.
        const key = photo.faceid || photo.fileid;
        const val = seen.get(key);
        if (val) {
          photo.key = `${key}-${val}`;
          seen.set(key, val + 1);
        } else {
          photo.key = `${key}`;
          seen.set(key, 1);
        }

        // Add photo to row
        row.photos!.push(photo);
        delete row.pct;
      }

      // Restore selection day
      this.selectionManager.restoreDay(day);

      // Rows that were removed
      const removedRows: IRow[] = [];
      let headRemoved = false;

      // No rows, splice everything including the header
      if (data.length === 0) {
        removedRows.push(...this.list.splice(headIdx, 1));
        rowIdx = headIdx - 1;
        headRemoved = true;
        delete this.heads[dayId];
      }

      // Get rid of any extra rows
      let spliceCount = 0;
      for (let i = rowIdx + 1; i < this.list.length && this.list[i].type !== IRowType.HEAD; i++) {
        spliceCount++;
      }
      if (spliceCount > 0) {
        removedRows.push(...this.list.splice(rowIdx + 1, spliceCount));
      }

      // Update size delta for removed rows and remove from day
      for (const row of removedRows) {
        // Scroll up if if above visible range
        if (rowY < scrollTop) {
          scrollTop -= row.size;
        }
        needAdjust = true;

        // Remove from day
        const idx = day.rows.indexOf(row);
        if (idx >= 0) day.rows.splice(idx, 1);
      }

      // This will be true even if the head is being spliced
      // because one row is always removed in that case
      if (needAdjust) {
        if (headRemoved) {
          // If the head was removed, we need a reflow,
          // or adjust isn't going to work right
          this.scrollerManager.reflow();
        } else {
          // Otherwise just adjust the ticks
          this.scrollerManager.adjust();
        }

        // Scroll to new position
        (<any>this.$refs.recycler).$el.scrollTop = scrollTop;
      }
    },

    /** Add and get a new blank photos row */
    addRow(day: IDay): IRow {
      // Make sure rows exists
      day.rows ??= [];

      // Create new row
      const row = {
        id: `${day.dayid}-${day.rows.length}`,
        num: day.rows.length,
        photos: [],
        type: IRowType.PHOTOS,
        size: this.rowHeight,
        dayId: day.dayid,
        day: day,
      };

      // Add to day
      day.rows.push(row);

      return row;
    },

    /**
     * Delete elements from main view with some animation
     *
     * This is also going to update day.detail for you and make
     * a call to processDay so just pass it the list of ids to
     * delete and the days that were updated.
     *
     * @param delPhotos photos to delete
     */
    async deleteFromViewWithAnimation(delPhotos: IPhoto[]) {
      // Only keep photos with day
      delPhotos = delPhotos.filter((p) => p?.d);
      if (delPhotos.length === 0) return;

      // Get all days that need to be updatd
      const updatedDays = new Set<IDay>(delPhotos.map((p) => p.d!));
      const delPhotosSet = new Set(delPhotos);

      // Animate the deletion
      for (const photo of delPhotos) {
        photo.flag |= this.c.FLAG_LEAVING;
      }

      // wait for 200ms
      await new Promise((resolve) => setTimeout(resolve, 200));

      // clear selection at this point
      this.selectionManager.clearSelection(delPhotos);

      // Reflow all touched days
      for (const day of updatedDays) {
        const newDetail = day.detail?.filter((p) => !delPhotosSet.has(p));
        this.processDay(day.dayid, newDetail!);
      }
    },
  },
});
</script>

<style lang="scss" scoped>
/** Main view */
.container {
  height: 100%;
  width: 100%;
  overflow: hidden;
  user-select: none;
  position: relative;

  * {
    -webkit-tap-highlight-color: transparent;
    -webkit-touch-callout: none;
  }
}

.recycler {
  will-change: scroll-position;
  contain: strict;
  height: 300px;
  width: 100%;
  transition: opacity 0.2s ease-in-out;

  :deep .vue-recycle-scroller__slot {
    contain: content;
  }

  :deep .vue-recycle-scroller__item-wrapper {
    contain: strict;
  }

  :deep .vue-recycle-scroller__item-view {
    contain: layout style;
  }

  &.empty {
    opacity: 0;
    transition: none;
    width: 0;
  }
}

.recycler .photo {
  contain: strict;
  display: block;
  position: absolute;
  top: 0;
  left: 0;
  cursor: pointer;
  height: 100%;
  transition: width 0.2s ease-in-out, height 0.2s ease-in-out, transform 0.2s ease-in-out; // reflow
}

/** Dynamic top matter */
.recycler-before {
  width: 100%;
  > .text {
    font-size: 1.2em;
    padding-top: 13px;
    padding-left: 8px;
    @media (max-width: 768px) {
      padding-left: 48px;
    }
  }
}
</style>
