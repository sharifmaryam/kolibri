<template>

  <div
    v-if="!loading"
    ref="mainWrapper"
    class="main-wrapper"
  >

    <div v-if="blockDoubleClicks" class="click-mask"></div>
    <SkipNavigationLink />
    <LearningActivityBar
      :resourceTitle="resourceTitle"
      :learningActivities="mappedLearningActivities"
      :isLessonContext="lessonContext"
      :isBookmarked="bookmark ? true : bookmark"
      :isCoachContent="isCoachContent"
      :contentProgress="contentProgress"
      :allowMarkComplete="allowMarkComplete"
      :contentKind="content.kind"
      data-test="learningActivityBar"
      @navigateBack="navigateBack"
      @toggleBookmark="toggleBookmark"
      @viewResourceList="toggleResourceList"
      @viewInfo="openSidePanel"
    />
    <KLinearLoader
      v-if="loading"
      class="loader"
      type="indeterminate"
      :delay="false"
    />
    <KPageContainer v-if="notAuthorized">
      <AuthMessage
        :authorizedRole="authorizedRole"
        :header="authorizationErrorHeader"
        :details="authorizationErrorDetails"
      />
    </KPageContainer>
    <KPageContainer v-else-if="error">
      <AppError />
    </KPageContainer>

    <div
      v-else
      id="main"
      role="main"
      tabindex="-1"
      class="main"
    >
      <ContentPage
        class="content"
        data-test="contentPage"
        :content="content"
        :lessonId="lessonId"
      />
    </div>

    <GlobalSnackbar />

    <!-- Side Panel for content metadata -->
    <FullScreenSidePanel
      v-if="sidePanelContent"
      @closePanel="sidePanelContent = null"
    >
      <CurrentlyViewedResourceMetadata
        :content="sidePanelContent"
        :canDownloadContent="canDownload"
      />
    </FullScreenSidePanel>

    <!-- Side Panel for "view resources" or "lesson resources" -->
    <FullScreenSidePanel
      v-if="showViewResourcesSidePanel"
      class="also-in-this-side-panel"
      @closePanel="showViewResourcesSidePanel = false"
    >
      <AlsoInThis
        :contentNodes="viewResourcesContents"
        :nextContent="nextContent"
        :title="viewResourcesTitle"
        :isLesson="lessonContext"
      />
    </FullScreenSidePanel>

  </div>

</template>


<script>

  import { mapGetters, mapState } from 'vuex';
  import get from 'lodash/get';
  import responsiveWindowMixin from 'kolibri.coreVue.mixins.responsiveWindowMixin';
  import { crossComponentTranslator } from 'kolibri.utils.i18n';

  import AuthMessage from 'kolibri.coreVue.components.AuthMessage';
  import FullScreenSidePanel from 'kolibri.coreVue.components.FullScreenSidePanel';
  import {
    LearningActivities,
    ContentKindsToLearningActivitiesMap,
  } from 'kolibri.coreVue.vuex.constants';
  import {
    ContentNodeProgressResource,
    ContentNodeResource,
    LessonResource,
  } from 'kolibri.resources';
  import commonCoreStrings from 'kolibri.coreVue.mixins.commonCoreStrings';
  import client from 'kolibri.client';
  import urls from 'kolibri.urls';
  import GlobalSnackbar from '../../../../../../kolibri/core/assets/src/views/GlobalSnackbar';
  import SkipNavigationLink from '../../../../../../kolibri/core/assets/src/views/SkipNavigationLink';
  import AppError from '../../../../../../kolibri/core/assets/src/views/AppError';
  import useCoreLearn from '../composables/useCoreLearn';
  import LessonResourceViewer from './classes/LessonResourceViewer';
  import CurrentlyViewedResourceMetadata from './CurrentlyViewedResourceMetadata';
  import ContentPage from './ContentPage';
  import LearningActivityBar from './LearningActivityBar';
  import AlsoInThis from './AlsoInThis';

  const sidepanelStrings = crossComponentTranslator(FullScreenSidePanel);
  const lessonStrings = crossComponentTranslator(LessonResourceViewer);

  export default {
    name: 'LearnImmersiveLayout',
    metaInfo() {
      return {
        // Use arrow function to bind $tr to this component
        titleTemplate: title => {
          if (this.error) {
            return this.$tr('kolibriTitleMessage', { title: this.$tr('errorPageTitle') });
          }
          if (!title) {
            // If no child component sets title, it reads 'Kolibri'
            return this.coreString('kolibriLabel');
          }
          // If child component sets title, it reads 'Child Title - Kolibri'
          return this.$tr('kolibriTitleMessage', { title });
        },
        title: this.pageTitle,
      };
    },
    components: {
      AlsoInThis,
      AppError,
      AuthMessage,
      ContentPage,
      FullScreenSidePanel,
      GlobalSnackbar,
      LearningActivityBar,
      CurrentlyViewedResourceMetadata,
      SkipNavigationLink,
    },
    mixins: [responsiveWindowMixin, commonCoreStrings],
    setup() {
      const { canDownload } = useCoreLearn();
      return { canDownload };
    },
    props: {
      content: {
        type: Object,
        required: false,
        default: null,
      },
      // AUTHORIZATION SPECIFIC
      authorized: {
        type: Boolean,
        required: false,
        default: true,
      },
      authorizedRole: {
        type: String,
        default: null,
      },
      // link to where the 'back' button should go
      back: {
        type: Object,
        required: true,
        default: null,
      },
    },
    data() {
      return {
        bookmark: null,
        sidePanelContent: null,
        showViewResourcesSidePanel: false,
        nextContent: null,
        viewResourcesContents: [],
      };
    },
    computed: {
      ...mapGetters(['currentUserId']),
      ...mapState({
        contentProgress: state => state.core.logging.progress,
        error: state => state.core.error,
        loading: state => state.core.loading,
        blockDoubleClicks: state => state.core.blockDoubleClicks,
      }),
      ...mapState('topicsTree', {
        isCoachContent: state => (state.content.coach_content ? 1 : 0),
      }),
      notAuthorized() {
        // catch "not authorized" error, display AuthMessage
        if (
          this.error &&
          this.error.response &&
          this.error.response.status &&
          this.error.response.status == 403
        ) {
          return true;
        }
        return !this.authorized;
      },
      resourceTitle() {
        return this.content ? this.content.title : '';
      },
      allowMarkComplete() {
        return get(this, ['content', 'options', 'completion_criteria', 'learner_managed'], false);
      },
      mappedLearningActivities() {
        let learningActivities = [];
        if (this.content && this.content.kind) {
          if (Object.values(LearningActivities).includes(this.content.kind)) {
            learningActivities.push(this.content.kind);
          } else {
            // otherwise reassign the old content types to the new metadata
            learningActivities.push(ContentKindsToLearningActivitiesMap[this.content.kind]);
          }
        }
        return learningActivities;
      },
      lessonContext() {
        return Boolean(this.lessonId);
      },
      lessonId() {
        return get(this.back, 'params.lessonId', null);
      },
      viewResourcesTitle() {
        /* eslint-disable kolibri/vue-no-undefined-string-uses */
        return this.lessonContext
          ? lessonStrings.$tr('nextInLesson')
          : sidepanelStrings.$tr('topicHeader');
        /* eslint-enable */
      },
    },
    watch: {
      content: function() {
        this.showViewResourcesSidePanel = false;
        this.getSidebarInfo();
      },
    },
    mounted() {
      /** I got 404s because content wasn't provided immediately upon mounting, so we check
       * this here, otherwise a watcher on `content` should trigger calling the same */
      this.getSidebarInfo();
    },
    beforeUpdate() {
      client({
        method: 'get',
        url: urls['kolibri:core:bookmarks-list'](),
        params: { contentnode_id: this.content.id },
      }).then(response => {
        this.bookmark = response.data[0] || false;
      });
    },
    methods: {
      getSidebarInfo() {
        if (this.lessonId) {
          this.fetchLessonSiblings();
        } else {
          if (this.content.id) {
            this.fetchSiblings();
          }
        }
      },
      /** Returns a promise which will get the progress for a set of nodes and map it back onto
       *  the nodes, which are then set to this.viewResourcesContents.
       *  @modifies this.viewResourcesContents - Sets it to the value of nodes, mapped with
       *  progress, if found
       *  @returns {Promise} - Just returns a meaningless Promise in case you want to chain
       *  synchronously
       */
      progressPromise(nodes) {
        // Avoid unnecessary fetches, set values, return empty Promise
        if (!this.$store.getters.isUserLoggedIn) {
          nodes.map(node => (node.progress = 0));
          this.viewResourcesContents = nodes;
          return Promise.resolve();
        }
        const getParams = {
          content_ids: nodes.map(node => node.content_id),
        };
        return ContentNodeProgressResource.fetchCollection({ getParams }).then(progresses => {
          this.viewResourcesContents = nodes
            .map(node => {
              const matchingProgress = progresses.find(p => p.content_id === node.content_id) || {
                progress: 0,
              };
              node.progress = matchingProgress.progress;
              return node;
            })
            .filter(node => node.content_id !== this.content.content_id);
        });
      },
      /**
       * Prepares a list of content nodes which are also in the same lesson as this.lessonId
       * @modifies this.viewResourcesContents - Sets it to the progress-mapped nodes
       */
      fetchLessonSiblings() {
        LessonResource.fetchModel({ id: this.lessonId }).then(lesson => {
          ContentNodeResource.fetchCollection({
            getParams: {
              ids: lesson.resources.map(resource => resource.contentnode_id),
            },
          }).then(contentNodes => {
            return this.progressPromise(contentNodes);
          });
        });
      },
      /**
       * Prepares a list of content nodes which are children of this.content.parent without
       * this.content. The prepared nodes have their progress values for the current user
       * (if there is one) mapped onto each node.
       *
       * Largely borrowed from modules/topicsTree/handlers.js
       *
       * @modifies this.viewResourcesContents - Sets it to the progress-mapped nodes
       * @modifies this.nextFolder - Sets the value with this.content's parents next sibling folder
       * if found
       */
      fetchSiblings() {
        // Guard against this in lesson context
        if (this.lessonId) {
          return;
        }

        ContentNodeResource.fetchTree({
          id: this.content.parent,
          params: {
            include_coach_content:
              this.$store.getters.isAdmin ||
              this.$store.getters.isCoach ||
              this.$store.getters.isSuperuser,
          },
        }).then(parent => {
          // Filter out this.content
          const nodes = parent.children.results.filter(node => node.id !== this.content.id);

          /** A promise to get the nextFolder content - done always */
          const nextFolderPromise = ContentNodeResource.fetchNextContent(parent.id, {
            topicOnly: true,
          }).then(nextContent => {
            // This may return the immediate parent if nothing else is found so let's be sure
            // not to assign that
            if (nextContent && this.content.parent !== nextContent.id) {
              this.nextContent = nextContent;
            }
          });

          const promises = [nextFolderPromise];
          // Push progressPromise, which handles user logged in logic
          promises.push(this.progressPromise(nodes));
          Promise.all(promises);
        });
      },
      navigateBack() {
        this.$router.push(this.back);
      },
      openSidePanel() {
        this.sidePanelContent = this.content;
      },
      toggleBookmark() {
        if (this.bookmark) {
          client({
            method: 'delete',
            url: urls['kolibri:core:bookmarks_detail'](this.bookmark.id),
          }).then(() => {
            this.bookmark = false;
          });
        } else if (this.bookmark === false) {
          client({
            method: 'post',
            url: urls['kolibri:core:bookmarks-list'](),
            data: {
              contentnode_id: this.content.id,
              user: this.currentUserId,
            },
          }).then(response => {
            this.bookmark = response.data;
          });
        }
      },
      toggleResourceList() {
        this.showViewResourcesSidePanel = true;
      },
    },
    $trs: {
      kolibriTitleMessage: {
        message: '{ title } - Kolibri',
        context: 'DO NOT TRANSLATE\nCopy the source string.',
      },
      errorPageTitle: {
        message: 'Error',
        context:
          "When Kolibri throws an error, this is the text that's used as the title of the error page. The description of the error follows below.",
      },
    },
  };

</script>


<style lang="scss" scoped>

  @import '~kolibri-design-system/lib/styles/definitions';

  .main-wrapper {
    display: inline-block;
    width: 100%;
    background-color: white;

    @media print {
      /* Without this, things won't print correctly
       *  - Firefox: Tables will get cutoff
       *  - Chrome: Table header won't repeat correctly on each page
       */
      display: block;
    }
  }

  // When focused by SkipNavigationLink, don't outline non-buttons/links
  /deep/ [tabindex='-1'] {
    outline-style: none !important;
  }

  .click-mask {
    position: fixed;
    top: 0;
    left: 0;
    z-index: 24;
    width: 100%;
    height: 100%;
  }

  .loader {
    position: fixed;
    right: 0;
    left: 0;
  }

  .also-in-this-side-panel {
    /deep/ .side-panel {
      padding-bottom: 0;
    }
  }

</style>
