<template>

  <div>
    <div v-if="currentChannelIsCustom">
      <CustomContentRenderer :topic="topic" />
    </div>
    <!-- appearanceOverrides overrides the default page styling -->
    <!-- by replacing it with an empty object -->
    <ImmersivePage
      v-else-if="!loading"
      :loading="loading"
      :route="back"
      :appBarTitle="barTitle"
      :appearanceOverrides="{}"
      :primary="!allowDownloads"
      class="page"
    >
      <template #actions>
        <DeviceConnectionStatus :deviceId="deviceId" color="white" />
      </template>
      <!-- Header with thumbail and tagline -->
      <TopicsHeader
        v-if="!windowIsSmall"
        ref="header"
        role="complementary"
        data-test="header-breadcrumbs"
        :title="(topic && topic.title) || ''"
        :description="topic && topic.description"
        :thumbnail="topic && topic.thumbnail"
        :breadcrumbs="breadcrumbs"
      />

      <!-- mobile tabs (different alignment and interactions) -->
      <TopicsMobileHeader v-else :topic="topic" />

      <main
        class="main-content-grid"
        :style="gridOffset"
      >
        <KBreadcrumbs
          v-if="breadcrumbs.length && windowIsSmall"
          data-test="mobile-breadcrumbs"
          :items="breadcrumbs"
          :ariaLabel="learnString('channelAndFoldersLabel')"
        />

        <div class="card-grid">
          <!-- Filter buttons - shown when not sidebar not visible -->
          <div v-if="!windowIsLarge" data-test="tab-buttons">
            <KButton
              v-if="topics.length"
              icon="topic"
              data-test="folders-button"
              class="overlay-toggle-button"
              :text="coreString('folders')"
              :primary="false"
              @click="toggleFolderSearchSidePanel('folder')"
            />
            <KButton
              icon="filter"
              class="overlay-toggle-button"
              data-test="filter-button"
              :text="coreString('filter')"
              :primary="false"
              @click="toggleFolderSearchSidePanel('search')"
            />
          </div>

          <!-- default/preview display of nested folder structure, not search -->
          <div v-if="!displayingSearchResults" data-test="topics">
            <!-- Rows of cards and links / show more for each Topic -->
            <template v-for="t in topicsForDisplay">
              <TopicSubsection
                :key="t.id"
                :topic="t"
                :subTopicLoading="t.id === subTopicLoading"
                :allowDownloads="allowDownloads"
                @showMore="handleShowMore"
                @loadMoreInSubtopic="handleLoadMoreInSubtopic"
                @toggleInfoPanel="toggleInfoPanel"
              />
            </template>

            <!-- display for each nested topic/folder  -->
            <!-- display all resources at the top level of the folder -->
            <LibraryAndChannelBrowserMainContent
              v-if="resources.length"
              :gridType="2"
              :allowDownloads="allowDownloads"
              data-test="search-results"
              :contents="resourcesDisplayed"
              :numCols="numCols"
              currentCardViewStyle="card"
              @toggleInfoPanel="toggleInfoPanel"
            />
            <KButton
              v-if="moreResources"
              class="more-after-grid"
              appearance="basic-link"
              @click="handleShowMoreResources"
            >
              {{ coreString('showMoreAction') }}
            </KButton>
            <div v-else-if="topicMore" class="end-button-block">
              <KButton
                v-if="!topicMoreLoading"
                :text="coreString('viewMoreAction')"
                appearance="raised-button"
                :disabled="topicMoreLoading"
                @click="handleLoadMoreInTopic"
              />
              <KCircularLoader v-else />
            </div>
          </div>

          <!-- search results -->
          <!-- TODO: Should card preference be permitted in Topics page as well? At least for
              search results? -->
          <SearchResultsGrid
            v-else
            data-test="search-results"
            :allowDownloads="allowDownloads"
            :currentCardViewStyle="currentSearchCardViewStyle"
            :hideCardViewToggle="true"
            :results="results"
            :removeFilterTag="removeFilterTag"
            :clearSearch="clearSearch"
            :moreLoading="moreLoading"
            :searchMore="searchMore"
            :searchTerms="searchTerms"
            :searchLoading="searchLoading"
            :more="more"
            @setCardStyle="style => currentSearchCardViewStyle = style"
            @setSidePanelMetadataContent="content => metadataSidePanelContent = content"
          />
        </div>
      </main>

      <!-- Side Panels for filtering and searching  -->
      <SearchPanelModal
        v-if="!windowIsLarge && sidePanelIsOpen"
        v-model="searchTerms"
        :mobileSearchActive="mobileSearchActive"
        :topicMore="topicMore"
        :topics="topics"
        :topicsLoading="topicMoreLoading"
        @searchTerms="newTerms => searchTerms = newTerms"
        @currentCategory="handleShowSearchModal"
        @closeCategoryModal="closeCategoryModal"
        @loadMoreTopics="handleLoadMoreInTopic"
        @close="sidePanelIsOpen = false"
      />

      <!-- Embedded Side panel is on larger views, and exists next to content -->
      <ToggleHeaderTabs
        v-if="!!windowIsLarge"
        :topic="topic"
        :topics="topics"
        :style="tabPosition"
      />
      <SearchFiltersPanel
        v-if="!!windowIsLarge"
        ref="sidePanel"
        v-model="searchTerms"
        :topicsListDisplayed="!desktopSearchActive"
        class="side-panel"
        topicPage="True"
        :topics="topics"
        :topicsLoading="topicMoreLoading"
        :more="topicMore"
        :width="`${sidePanelWidth}px`"
        :showChannels="false"
        position="embedded"
        :style="sidePanelStyleOverrides"
        @currentCategory="handleShowSearchModal"
        @loadMoreTopics="handleLoadMoreInTopic"
      />
      <!-- The full screen side panel is used on smaller screens, and toggles as an overlay -->
      <!-- FullScreen is a container component, and then the SearchFiltersPanel sits within -->
      <SidePanelModal
        v-if="!windowIsLarge && sidePanelIsOpen"
        class="full-screen-side-panel"
        alignment="left"
        :closeButtonIconType="closeButtonIcon"
        :sidePanelOverrideWidth="`${sidePanelOverlayWidth}px`"
        @closePanel="closeEventHandler()"
        @shouldFocusFirstEl="findFirstEl()"
      >
        <SearchFiltersPanel
          v-if="!currentCategory"
          ref="embeddedPanel"
          v-model="searchTerms"
          :topicsListDisplayed="!mobileSearchActive"
          topicPage="True"
          :topics="topics"
          :topicsLoading="topicMoreLoading"
          :more="topicMore"
          :showChannels="false"
          position="overlay"
          @currentCategory="handleShowSearchModal"
          @loadMoreTopics="handleLoadMoreInTopic"
        />
        <CategorySearchModal
          v-if="currentCategory && windowIsSmall"
          :selectedCategory="currentCategory"
          @cancel="currentCategory = null"
          @input="handleCategory"
        />
      </SidePanelModal>
      <CategorySearchModal
        v-if="currentCategory"
        :selectedCategory="currentCategory"
        @cancel="currentCategory = null"
        @input="handleCategory"
      />


      <!-- Side panel for showing the information of selected content with a link to view it -->
      <SidePanelModal
        v-if="metadataSidePanelContent"
        alignment="right"
        :closeButtonIconType="closeButtonIcon"
        @closePanel="metadataSidePanelContent = null"
        @shouldFocusFirstEl="findFirstEl()"
      >
        <template #header>
          <!-- Flex styles tested in ie11 and look good. Ensures good spacing between
            multiple chips - not a common thing but just in case -->
          <div
            v-for="activity in metadataSidePanelContent.learning_activities"
            :key="activity"
            class="side-panel-chips"
            :class="$computedClass({ '::after': {
              content: '',
              flex: 'auto'
            } })"
          >
            <LearningActivityChip
              class="chip"
              style="margin-left: 8px; margin-bottom: 8px;"
              :kind="activity"
            />
          </div>
        </template>

        <BrowseResourceMetadata
          ref="resourcePanel"
          :content="metadataSidePanelContent"
          :showLocationsInChannel="true"
        />
      </SidePanelModal>

    </ImmersivePage>
  </div>

</template>


<script>

  import { mapActions, mapState } from 'vuex';
  import isEqual from 'lodash/isEqual';
  import set from 'lodash/set';
  import KBreadcrumbs from 'kolibri-design-system/lib/KBreadcrumbs';
  import { computed, getCurrentInstance } from 'kolibri.lib.vueCompositionApi';
  import responsiveWindowMixin from 'kolibri.coreVue.mixins.responsiveWindowMixin';
  import { ContentNodeKinds } from 'kolibri.coreVue.vuex.constants';
  import commonCoreStrings from 'kolibri.coreVue.mixins.commonCoreStrings';
  import { throttle } from 'frame-throttle';
  import ImmersivePage from 'kolibri.coreVue.components.ImmersivePage';
  import SidePanelModal from '../SidePanelModal';
  import { PageNames } from '../../constants';
  import useSearch from '../../composables/useSearch';
  import useContentLink from '../../composables/useContentLink';
  import LibraryAndChannelBrowserMainContent from '../LibraryAndChannelBrowserMainContent';
  import SearchFiltersPanel from '../SearchFiltersPanel';
  import BrowseResourceMetadata from '../BrowseResourceMetadata';
  import LearningActivityChip from '../LearningActivityChip';
  import CustomContentRenderer from '../ChannelRenderer/CustomContentRenderer';
  import CategorySearchModal from '../CategorySearchModal';
  import SearchResultsGrid from '../SearchResultsGrid';
  import DeviceConnectionStatus from '../DeviceConnectionStatus.vue';
  import TopicsHeader from './TopicsHeader';
  import ToggleHeaderTabs from './ToggleHeaderTabs';
  import TopicsMobileHeader from './TopicsMobileHeader';
  import TopicSubsection from './TopicSubsection';
  import SearchPanelModal from './SearchPanelModal';
  import commonLearnStrings from './../commonLearnStrings';
  import plugin_data from 'plugin_data';

  export default {
    name: 'TopicsPage',
    metaInfo() {
      let title;
      if (this.isRoot) {
        title = this.$tr('documentTitleForChannel', {
          channelTitle: this.channelTitle,
        });
      } else {
        title = this.$tr('documentTitleForTopic', {
          channelTitle: this.channelTitle,
          topicTitle: this.topic ? this.topic.title : '',
        });
      }
      return { title };
    },
    components: {
      KBreadcrumbs,
      TopicsHeader,
      ToggleHeaderTabs,
      LibraryAndChannelBrowserMainContent,
      CustomContentRenderer,
      CategorySearchModal,
      SearchFiltersPanel,
      SidePanelModal,
      LearningActivityChip,
      BrowseResourceMetadata,
      SearchResultsGrid,
      TopicsMobileHeader,
      TopicSubsection,
      SearchPanelModal,
      ImmersivePage,
      DeviceConnectionStatus,
    },
    mixins: [responsiveWindowMixin, commonCoreStrings, commonLearnStrings],
    setup() {
      const store = getCurrentInstance().proxy.$store;
      const topic = computed(() => store.state.topicsTree && store.state.topicsTree.topic);
      const {
        searchTerms,
        displayingSearchResults,
        searchLoading,
        moreLoading,
        results,
        more,
        search,
        searchMore,
        removeFilterTag,
        clearSearch,
        setCategory,
      } = useSearch(topic);
      const { back, genContentLinkKeepCurrentBackLink } = useContentLink();
      return {
        searchTerms,
        displayingSearchResults,
        searchLoading,
        moreLoading,
        results,
        more,
        search,
        searchMore,
        removeFilterTag,
        clearSearch,
        setCategory,
        back,
        genContentLinkKeepCurrentBackLink,
      };
    },
    props: {
      loading: {
        type: Boolean,
        default: null,
      },
      deviceId: {
        type: String,
        default: null,
      },
    },
    data: function() {
      return {
        sidePanelStyleOverrides: {},
        tabPosition: {},
        currentCategory: null,
        showSearchModal: false,
        showMoreResources: false,
        sidePanelIsOpen: false,
        metadataSidePanelContent: null,
        expandedTopics: {},
        subTopicLoading: null,
        topicMoreLoading: false,
        mobileSearchActive: false,
        currentSearchCardViewStyle: 'card',
      };
    },
    computed: {
      ...mapState('topicsTree', ['channel', 'contents', 'isRoot', 'topic']),
      allowDownloads() {
        return Boolean(this.deviceId);
      },
      barTitle() {
        return this.deviceId
          ? this.learnString('exploreLibraries')
          : (this.topic && this.topic.title) || '';
      },
      childrenToDisplay() {
        return Math.max(this.numCols, 3);
      },
      breadcrumbs() {
        if (!this.topic || !this.topic.ancestors) {
          return [];
        }
        return [
          ...this.topic.ancestors.map(({ title, id }, index) => {
            const link = this.genContentLinkKeepCurrentBackLink(id, false);
            // To allow navigating to a specific topic under the breadcrumb
            // without following the normal skip logic, add a special query
            // parameter to signal that we do not want to skip.
            set(link, ['query', 'skip'], 'false');
            return {
              // Use the channel name just in case the root node does not have a title.
              text: index === 0 ? this.channelTitle : title,
              link,
            };
          }),
          { text: this.topic.ancestors.length ? this.topic.title : this.channelTitle },
        ];
      },
      searchTabLink() {
        // navigates the main page to the search view
        if (this.topic) {
          const query = { ...this.$route.query };
          delete query.dropdown;
          return {
            name: PageNames.TOPICS_TOPIC_SEARCH,
            id: this.topic.id,
            query: query,
          };
        }
        return {};
      },
      desktopSearchActive() {
        return this.$route.name === PageNames.TOPICS_TOPIC_SEARCH;
      },
      channelTitle() {
        return this.channel.name;
      },
      closeButtonIcon() {
        if (this.windowIsSmall && this.currentCategory) {
          return 'back';
        } else {
          return 'close';
        }
      },
      resources() {
        return this.contents.filter(content => content.kind !== ContentNodeKinds.TOPIC);
      },
      resourcesDisplayed() {
        // if no folders are shown at this level, show more resources to fill the space
        // or if the user has explicitly requested to show more resources
        if (!this.topics.length || this.showMoreResources) {
          return this.resources;
        }
        return this.resources.slice(0, this.childrenToDisplay);
      },
      moreResources() {
        return this.resourcesDisplayed.length < this.resources.length;
      },
      topics() {
        return this.contents
          .filter(content => content.kind === ContentNodeKinds.TOPIC)
          .filter(t => t.children && t.children.results.length)
          .map(t => {
            let topicChildren = t.children.results;
            const prefixTitles = [];
            while (topicChildren.length === 1 && !topicChildren[0].is_leaf) {
              // If the topic has only one child, and that child is also a topic
              // we should collapse the topics to display the child topic instead.
              prefixTitles.push(t.title);
              t = topicChildren[0];
              topicChildren = t.children ? t.children.results : [];
            }
            t.prefixTitles = prefixTitles;
            return t;
          });
      },
      topicsForDisplay() {
        return this.topics
          .filter(t => (this.subTopicId ? t.id === this.subTopicId : true))
          .map(t => {
            let childrenToDisplay;
            const topicChildren = t.children ? t.children.results : [];
            if (this.subTopicId || this.topics.length === 1) {
              // If we are in a subtopic display, we should only be displaying this topic
              // so don't bother checking if the ids match.
              // Alternatively, if there is only one topic, we should display all of its children.
              childrenToDisplay = topicChildren.length;
            } else if (this.expandedTopics[t.id]) {
              // If topic is expanded show three times as many children.
              childrenToDisplay = this.childrenToDisplay * 3;
            } else {
              childrenToDisplay = this.childrenToDisplay;
            }
            const children = topicChildren.slice(0, childrenToDisplay);
            // showMore is whether we should show more inline
            const showMore =
              !this.subTopicId &&
              topicChildren.length > this.childrenToDisplay &&
              !this.expandedTopics[t.id];

            // viewMore is the 'more' object that will be used to load more items from this topic.
            const viewMore = t.children ? t.children.more : null;

            // viewAll is a flag + link object to link to a subpage which shows all initially
            // loaded topics content
            const viewAll =
              !this.subTopicId && (topicChildren.length > childrenToDisplay || viewMore)
                ? {
                    ...this.$route,
                    params: {
                      ...this.$route.params,
                      subtopic: t.id,
                    },
                  }
                : null;

            return {
              ...t,
              viewAll,
              children,
              showMore,
              viewMore,
            };
          });
      },
      subTopicId() {
        return this.$route.params.subtopic;
      },
      currentChannelIsCustom() {
        if (
          plugin_data.enableCustomChannelNav &&
          this.topic &&
          this.topic.options.modality === 'CUSTOM_NAVIGATION'
        ) {
          return true;
        }
        return false;
      },
      sidePanelWidth() {
        if (!this.windowIsLarge) {
          return 0;
        } else if (this.windowBreakpoint < 4) {
          return 234;
        } else {
          return 346;
        }
      },
      gridOffset() {
        return this.isRtl
          ? { marginRight: `${this.sidePanelWidth + 24}px` }
          : { marginLeft: `${this.sidePanelWidth + 24}px` };
      },
      numCols() {
        if (this.windowBreakpoint > 1 && this.windowBreakpoint < 2) {
          return 2;
        } else if (this.windowBreakpoint >= 2 && this.windowBreakpoint <= 4) {
          return 3;
        } else if (this.windowBreakpoint > 4) {
          return 4;
        } else return null;
      },
      throttledStickyCalculation() {
        return throttle(this.stickyCalculation);
      },
      // calls handleScroll no more than every 17ms
      throttledTabPositionCalculation() {
        return throttle(this.tabPositionCalculation);
      },
      topicMore() {
        return this.topic && this.topic.children && this.topic.children.more;
      },
    },
    watch: {
      subTopicId(newValue, oldValue) {
        if (newValue && newValue !== oldValue) {
          this.handleLoadMoreInSubtopic(newValue);
        }
      },
      searchTerms(newVal, oldVal) {
        // When there are search terms and the Folders link is clicked,
        // this ensures that we don't redirect to the search page when the
        // user wanted to go to the Folders page.
        if (this.$route.name === PageNames.TOPICS_TOPIC) {
          this.sidePanelIsOpen = false;
          return;
        }
        if (!isEqual(newVal, oldVal)) {
          if (!isEqual(this.searchTabLink, this.$route)) {
            this.$router.push({ ...this.searchTabLink }).catch(() => {});
          }
          this.sidePanelIsOpen = false;
        }
      },
      metadataSidePanelContent() {
        if (this.metadataSidePanelContent) {
          // Ensure the content underneath isn't scrolled - unset this when destroyed
          document.documentElement.style.position = 'fixed';
          return;
        }
        document.documentElement.style.position = '';
      },
    },
    beforeDestroy() {
      window.removeEventListener('scroll', this.throttledHandleScroll);
      // Unsetting possible change in metadataSidePanelContent watcher
      // to avoid leaving `fixed` position
      document.documentElement.style.position = '';
    },
    created() {
      window.addEventListener('scroll', this.throttledHandleScroll);
      if (this.subTopicId) {
        this.handleLoadMoreInSubtopic(this.subTopicId);
      }
    },
    methods: {
      ...mapActions('topicsTree', ['loadMoreContents', 'loadMoreTopics']),
      throttledHandleScroll() {
        this.throttledStickyCalculation();
        this.throttledTabPositionCalculation();
      },
      handleShowSearchModal(value) {
        this.currentCategory = value;
        this.showSearchModal = true;
        !this.windowIsSmall ? (this.sidePanelIsOpen = false) : '';
      },
      handleCategory(category) {
        this.setCategory(category);
        this.currentCategory = null;
      },
      toggleInfoPanel(content) {
        this.metadataSidePanelContent = content;
      },
      toggleFolderSearchSidePanel(option) {
        option == 'search' ? (this.mobileSearchActive = true) : (this.mobileSearchActive = false);
        this.sidePanelIsOpen = !this.sidePanelIsOpen;
      },
      closeEventHandler() {
        if (this.windowIsSmall && this.currentCategory) {
          this.currentCategory = null;
        }
        this.toggleFolderSearchSidePanel();
      },
      tabPositionCalculation() {
        const tabBottom = this.$refs.sidePanel
          ? this.$refs.sidePanel.$el.getBoundingClientRect().top
          : 0;
        if (tabBottom > 0) {
          this.tabPosition = {
            position: 'fixed',
            top: `${tabBottom - 70}px`,
          };
        } else {
          this.tabPosition = {};
        }
      },
      // Stick the side panel to top. That can be on the very top of the viewport
      // or right under the 'Browse channel' toolbar, depending on whether the toolbar
      // is visible or no (the toolbar hides on smaller resolutions when scrolling
      // down and appears again when scrolling up).
      // Takes effect only when the side panel is not displayed full-screen.
      stickyCalculation() {
        const header = this.$refs.header && this.$refs.header.$el;
        const topbar = document.querySelector('.scrolling-header');

        const headerBottom = header ? header.getBoundingClientRect().bottom : 0;
        const topbarBottom = topbar ? topbar.getBoundingClientRect().bottom : 0;

        if (headerBottom < Math.max(topbarBottom, 0)) {
          this.sidePanelStyleOverrides = {
            position: 'fixed',
            top: `${Math.max(0, headerBottom, topbarBottom)}px`,
            height: '100%',
          };
        } else {
          this.sidePanelStyleOverrides = {};
        }
      },
      handleShowMore(topicId) {
        this.expandedTopics = {
          ...this.expandedTopics,
          [topicId]: true,
        };
      },
      handleLoadMoreInSubtopic(topicId) {
        this.subTopicLoading = topicId;
        this.loadMoreContents(topicId).then(() => {
          this.subTopicLoading = null;
        });
      },
      handleLoadMoreInTopic() {
        this.topicMoreLoading = true;
        this.loadMoreTopics().then(() => {
          this.topicMoreLoading = false;
        });
      },
      handleShowMoreResources() {
        this.showMoreResources = true;
      },
      findFirstEl() {
        if (this.$refs.embeddedPanel) {
          this.$refs.embeddedPanel.focusFirstEl();
        } else {
          this.$refs.resourcePanel.focusFirstEl();
        }
      },
    },
    $trs: {
      documentTitleForChannel: {
        message: 'Folders - { channelTitle }',
        context:
          'A folder is a collection of resources and other subfolders within a channel. This string indicates the folders grouped under a specific channel.',
      },
      documentTitleForTopic: {
        message: '{ topicTitle } - { channelTitle }',
        context: 'DO NOT TRANSLATE\nCopy the source string.',
      },
    },
  };

</script>


<style lang="scss" scoped>

  $header-height: 324px;
  $toolbar-height: 70px;
  $total-height: 394px;

  .page {
    position: relative;
    width: 100%;
    min-height: calc(100vh - #{$toolbar-height});
  }

  .side-panel {
    position: absolute;
    top: $total-height;
    padding-top: 16px;
  }

  .main-content-grid {
    position: relative;
    top: $toolbar-height;
    margin: 24px;
  }

  .text {
    max-width: 920px;
  }

  /deep/.card-thumbnail-wrapper {
    max-width: 100%;
    height: 110px;
  }

  .results-title {
    display: inline-block;
    margin-bottom: 24px;
  }

  .results-header-group {
    margin-bottom: 24px;
  }

  .filter-action-button {
    display: inline-block;
    margin: 4px;
    margin-left: 8px;
  }

  .overlay-toggle-button {
    margin: 16px 16px 16px 0;
  }

  .full-screen-side-panel {
    position: fixed;
    top: 0;
    bottom: 0;
    z-index: 12;
    width: 100vw;
  }

  .end-button-block {
    width: 100%;
    margin-top: 16px;
    text-align: center;
  }

  .side-panel-chips {
    display: flex;
    flex-wrap: wrap;
    justify-content: flex-start;
    margin-bottom: -8px;
    margin-left: -8px;
  }

  .chip {
    margin-bottom: 8px;
    margin-left: 8px;
  }

</style>
