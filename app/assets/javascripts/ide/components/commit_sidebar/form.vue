<script>
import { mapState, mapActions, mapGetters } from 'vuex';
import { sprintf, __ } from '~/locale';
import LoadingButton from '~/vue_shared/components/loading_button.vue';
import CommitMessageField from './message_field.vue';
import Actions from './actions.vue';
import SuccessMessage from './success_message.vue';
import { activityBarViews, MAX_WINDOW_HEIGHT_COMPACT } from '../../constants';

export default {
  components: {
    Actions,
    LoadingButton,
    CommitMessageField,
    SuccessMessage,
  },
  data() {
    return {
      isCompact: true,
      componentHeight: null,
    };
  },
  computed: {
    ...mapState(['changedFiles', 'stagedFiles', 'currentActivityView', 'lastCommitMsg']),
    ...mapState('commit', ['commitMessage', 'submitCommitLoading']),
    ...mapGetters(['hasChanges']),
    ...mapGetters('commit', ['commitButtonDisabled', 'discardDraftButtonDisabled']),
    overviewText() {
      return sprintf(
        __(
          '<strong>%{changedFilesLength} unstaged</strong> and <strong>%{stagedFilesLength} staged</strong> changes',
        ),
        {
          stagedFilesLength: this.stagedFiles.length,
          changedFilesLength: this.changedFiles.length,
        },
      );
    },
  },
  watch: {
    currentActivityView() {
      if (this.lastCommitMsg) {
        this.isCompact = false;
      } else {
        this.isCompact = !(
          this.currentActivityView === activityBarViews.commit &&
          window.innerHeight >= MAX_WINDOW_HEIGHT_COMPACT
        );
      }
    },
    lastCommitMsg() {
      this.isCompact =
        this.currentActivityView !== activityBarViews.commit && this.lastCommitMsg === '';
    },
  },
  methods: {
    ...mapActions(['updateActivityBarView']),
    ...mapActions('commit', ['updateCommitMessage', 'discardDraft', 'commitChanges']),
    toggleIsSmall() {
      this.updateActivityBarView(activityBarViews.commit)
        .then(() => {
          this.isCompact = !this.isCompact;
        })
        .catch(e => {
          throw e;
        });
    },
    beforeEnterTransition() {
      const elHeight = this.isCompact
        ? this.$refs.formEl && this.$refs.formEl.offsetHeight
        : this.$refs.compactEl && this.$refs.compactEl.offsetHeight;

      this.componentHeight = elHeight;
    },
    enterTransition() {
      this.$nextTick(() => {
        const elHeight = this.isCompact
          ? this.$refs.compactEl && this.$refs.compactEl.offsetHeight
          : this.$refs.formEl && this.$refs.formEl.offsetHeight;

        this.componentHeight = elHeight;
      });
    },
    afterEndTransition() {
      this.componentHeight = null;
    },
  },
  activityBarViews,
};
</script>

<template>
  <div
    class="multi-file-commit-form"
    :class="{
      'is-compact': isCompact,
      'is-full': !isCompact
    }"
    :style="{
      height: componentHeight ? `${componentHeight}px` : null,
    }"
  >
    <transition
      name="commit-form-slide-up"
      @before-enter="beforeEnterTransition"
      @enter="enterTransition"
      @after-enter="afterEndTransition"
    >
      <div
        v-if="isCompact"
        class="commit-form-compact"
        ref="compactEl"
      >
        <button
          type="button"
          :disabled="!hasChanges"
          class="btn btn-primary btn-sm btn-block"
          @click="toggleIsSmall"
        >
          {{ __('Commit') }}
        </button>
        <p
          class="text-center"
          v-html="overviewText"
        ></p>
      </div>
      <form
        v-if="!isCompact"
        @submit.prevent.stop="commitChanges"
        ref="formEl"
      >
        <transition name="fade">
          <success-message
            v-show="lastCommitMsg"
          />
        </transition>
        <commit-message-field
          :text="commitMessage"
          @input="updateCommitMessage"
        />
        <div class="clearfix prepend-top-15">
          <actions />
          <loading-button
            :loading="submitCommitLoading"
            :disabled="commitButtonDisabled"
            container-class="btn btn-success btn-sm float-left"
            :label="__('Commit')"
            @click="commitChanges"
          />
          <button
            v-if="!discardDraftButtonDisabled"
            type="button"
            class="btn btn-default btn-sm float-right"
            @click="discardDraft"
          >
            {{ __('Discard draft') }}
          </button>
          <button
            v-else
            type="button"
            class="btn btn-default btn-sm float-right"
            @click="toggleIsSmall"
          >
            {{ __('Collapse') }}
          </button>
        </div>
      </form>
    </transition>
  </div>
</template>
