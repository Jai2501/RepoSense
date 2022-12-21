<template>
  <div id="zoom">
    <div class="panel-title">
      <span>Commits Panel</span>
    </div>
    <div class="toolbar--multiline" v-if="filteredUser.commits.length && totalCommitMessageBodyCount">
      <a
        v-if="expandedCommitMessagesCount < totalCommitMessageBodyCount"
        v-on:click="toggleAllCommitMessagesBody(true)">
        show all commit messages
      </a>
      <a
        v-if="expandedCommitMessagesCount > 0"
        v-on:click="toggleAllCommitMessagesBody(false)">
        hide all commit messages
      </a>
    </div>
    <div class="panel-heading">
      <div class="group-name">
        <span v-if="info.zFilterGroup === 'groupByAuthors'">
          {{ filteredUser.displayName }} ({{ filteredUser.name }})
        </span>
        <a
          v-else
          v-bind:href="info.zLocation"
          target="_blank"
          v-bind:title="'Click to open the repo'">
            <span>{{ filteredUser.repoName }}</span>
          </a>
      </div>
      <div class="author" v-if="!info.zIsMerge">
        <span>
          &#8627; &nbsp;
        </span>
        <span v-if="info.zFilterGroup === 'groupByAuthors'">
          {{ filteredUser.repoName }}
        </span>
        <span v-else>
          {{ filteredUser.displayName }} ({{ filteredUser.name }})
        </span>
      </div>
      <div class="period">
        <span>
          &#8627; &nbsp;
        </span>
        <span>
          {{ info.zSince }} to {{ info.zUntil }} &nbsp;
        </span>
      </div>
    </div>
    <div class="zoom__title">
      <div class="zoom__title--granularity">
        granularity: one ramp per {{ info.zTimeFrame }}
      </div>
      <div class="zoom__title--tags">
        <template v-for="commit in filteredUser.commits">
          <template v-for="commitResult in commit.commitResults">
            <template v-if="commitResult.tags">
              <div class="tag" :key="tag"
                v-for="tag in commitResult.tags"
                v-on:click="scrollToCommit(tag, `tag ${commitResult.hash}`)">
                <font-awesome-icon icon="tags">
                </font-awesome-icon>
                <span>&nbsp;{{ tag }}</span>
              </div>
            </template>
          </template>
        </template>
      </div>
    </div>
    <c-ramp
      :groupby="info.zFilterGroup"
      :user="filteredUser"
      :tframe="info.zTimeFrame"
      :sdate="info.zSince"
      :udate="info.zUntil"
      :avgsize="info.zAvgCommitSize"
      :mergegroup="info.zIsMerge"
      :fromramp="info.zFromRamp"
      :filtersearch="info.zFilterSearch">
    </c-ramp>
    <div class="sorting mui-form--inline">
      <div class="mui-select sort-by">
        <select v-model="commitsSortType">
          <option value="time">Time</option>
          <option value="lineOfCode">LoC</option>
        </select>
        <label>sort by</label>
      </div>
      <div class="mui-select sort-order">
        <select v-model="toReverseSortedCommits">
          <option :value="true">Descending</option>
          <option :value="false">Ascending</option>
        </select>
        <label>order</label>
      </div>
    </div>
    <div class="fileTypes">
      <div class="checkboxes mui-form--inline" v-if="fileTypes.length > 0">
        <label style='background-color: #000000; color: #ffffff'>
          <input type="checkbox" class="mui-checkbox--fileType" v-model="isSelectAllChecked" value="all">
          <span>All&nbsp;</span>
        </label>
        <label
          v-for="fileType in fileTypes"
          v-bind:key="fileType"
          v-bind:style="{
            'background-color': fileTypeColors[fileType],
            'color': getFontColor(fileTypeColors[fileType])
          }">
          <input type="checkbox" class="mui-checkbox--fileType" v-bind:value="fileType"
            v-on:change="updateSelectedFileTypesHash" v-model="selectedFileTypes">
          <span>{{ fileType }} &nbsp;</span>
        </label>
      </div>
    </div>
    <div class="zoom__day" v-for="day in selectedCommits" v-bind:key="day.date">
      <h3 v-if="info.zTimeFrame === 'week'">Week of {{ day.date }}</h3>
      <h3 v-else>{{ day.date }}</h3>
      <div class="commit-message" tabindex="-1"
        v-for="slice in day.commitResults"
        v-bind:key="slice.hash"
        v-bind:class="{ 'message-body active': slice.messageBody !== '' }">
        <a :href="getSliceLink(slice)" class="message-title"
          v-bind:class="!isBrokenLink(getSliceLink(slice)) ? '' : 'broken-link'"
          target="_blank">
          <div class="within-border">
            {{ slice.messageTitle.substr(0, 50) }}
            </div>
          <div class="not-within-border" v-if="slice.messageTitle.length > 50">
            |{{ slice.messageTitle.substr(50) }}
          </div>
        </a>
        <span>
          &nbsp; ({{ slice.insertions }} lines) &nbsp;
        </span>
        <div class="hash">
          <span>{{ slice.hash.substr(0, 7) }}</span>
        </div>
        <div class="fileTypeLabelDiv" v-if="containsAtLeastOneSelected(Object.keys(slice.fileTypesAndContributionMap))">
          <span class="fileTypeLabel"
            :key="fileType"
            v-for="fileType in Object.keys(slice.fileTypesAndContributionMap)"
            :style="{
              'background-color': fileTypeColors[fileType],
              'color': getFontColor(fileTypeColors[fileType])
            }">
            {{ fileType }}
        </span>
        </div>
        <template v-if="slice.tags">
          <div class="tag"
            :key="tag"
            v-for="tag in slice.tags"
            tabindex="-1"
            :class="[`${slice.hash}`, tag]">
            <font-awesome-icon icon="tags">
            </font-awesome-icon>
            <span>&nbsp;{{ tag }}</span>
          </div>
        </template>
        <a
          v-if="slice.messageBody !== ''"
          v-on:click="updateExpandedCommitMessagesCount"
          onclick="toggleNext(this)">
          <div class="tooltip">
            <font-awesome-icon icon="ellipsis-h" class="commit-message--button">
            </font-awesome-icon>
            <span class="tooltip-text">
              Click to show/hide the commit message body
            </span>
          </div>
        </a>
        <div class="body" v-if="slice.messageBody !== ''">
          <pre>
            {{ slice.messageBody }}
            <div class="dashed-border"></div>
          </pre>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { mapState } from 'vuex';
import { FontAwesomeIcon } from '@fortawesome/vue-fontawesome';
import brokenLinkDisabler from '../mixin/brokenLinkMixin.ts';
import cRamp from '../components/c-ramp.vue';

const getFontColor = window.getFontColor;

function zoomInitialState() {
  return {
    showAllCommitMessageBody: true,
    commitsSortType: 'time',
    toReverseSortedCommits: true,
    isCommitsFinalized: false,
    selectedFileTypes: [],
    fileTypes: [],
  };
}

export default {
  name: 'c-zoom',
  mixins: [brokenLinkDisabler],
  components: {
    FontAwesomeIcon,
    cRamp,
  },
  data() {
    return {
      expandedCommitMessagesCount: this.totalCommitMessageBodyCount,
      ...zoomInitialState(),
    };
  },

  computed: {
    sortingFunction() {
      const commitSortFunction = this.commitsSortType === 'time'
        ? (commit) => commit.date
        : (commit) => commit.insertions;

      return (a, b) => (this.toReverseSortedCommits ? -1 : 1)
        * window.comparator(commitSortFunction)(a, b);
    },
    filteredUser() {
      const {
        zUser, zSince, zUntil, zTimeFrame,
      } = this.info;
      const filteredUser = Object.assign({}, zUser);

      const date = zTimeFrame === 'week' ? 'endDate' : 'date';
      filteredUser.commits = zUser.commits.filter(
          (commit) => commit[date] >= zSince && commit[date] <= zUntil,
      ).sort(this.sortingFunction);

      return filteredUser;
    },
    selectedCommits() {
      const commits = [];
      this.filteredUser.commits.forEach((commit) => {
        const filteredCommit = { ...commit };
        filteredCommit.commitResults = [];
        commit.commitResults.forEach((slice) => {
          if (Object.keys(slice.fileTypesAndContributionMap).some(
              (fileType) => this.selectedFileTypes.indexOf(fileType) !== -1,
          )) {
            filteredCommit.commitResults.push(slice);
          }
        });
        if (filteredCommit.commitResults.length > 0) {
          commits.push(filteredCommit);
        }
      });
      return commits;
    },
    totalCommitMessageBodyCount() {
      let nonEmptyCommitMessageCount = 0;
      this.selectedCommits.forEach((commit) => {
        commit.commitResults.forEach((commitResult) => {
          if (commitResult.messageBody !== '') {
            nonEmptyCommitMessageCount += 1;
          }
        });
      });

      return nonEmptyCommitMessageCount;
    },
    isSelectAllChecked: {
      get() {
        return this.selectedFileTypes.length === this.fileTypes.length;
      },
      set(value) {
        if (value) {
          this.selectedFileTypes = this.fileTypes.slice();
        } else {
          this.selectedFileTypes = [];
        }
        this.updateSelectedFileTypesHash();
      },
    },

    ...mapState({
      fileTypeColors: 'fileTypeColors',
      info: 'tabZoomInfo',
    }),
  },

  watch: {
    info() {
      const newData = {
        expandedCommitMessagesCount: this.totalCommitMessageBodyCount,
        ...zoomInitialState(),
      };
      Object.assign(this.$data, newData);
      this.initiate();
      this.setInfoHash();
    },
    selectedFileTypes: {
      deep: true,
      handler() {
        this.$nextTick(() => {
          this.updateExpandedCommitMessagesCount();
        });
      },
    },
    commitsSortType() {
      window.addHash('zCST', this.commitsSortType);
      window.encodeHash();
    },
    toReverseSortedCommits() {
      window.addHash('zRSC', this.toReverseSortedCommits);
      window.encodeHash();
    },
  },

  methods: {
    initiate() {
      // This code crashes if info.zUser is not defined
      this.updateFileTypes();
      this.selectedFileTypes = this.fileTypes.slice();
    },

    getSliceLink(slice) {
      if (this.info.zIsMerged) {
        return window.getCommitLink(slice.repoId, slice.hash);
      }
      return window.getCommitLink(this.info.zUser.repoId, slice.hash);
    },

    scrollToCommit(tag, commit) {
      const el = this.$el.getElementsByClassName(`${commit} ${tag}`)[0];
      if (el) {
        el.focus();
      }
    },

    updateFileTypes() {
      const commitsFileTypes = new Set();
      this.filteredUser.commits.forEach((commit) => {
        commit.commitResults.forEach((slice) => {
          Object.keys(slice.fileTypesAndContributionMap).forEach((fileType) => {
            commitsFileTypes.add(fileType);
          });
        });
      });
      this.fileTypes = Object.keys(this.filteredUser.fileTypeContribution).filter(
          (fileType) => commitsFileTypes.has(fileType),
      );
    },

    retrieveHashes() {
      this.retrieveSortHash();
      this.retrieveSelectedFileTypesHash();
    },

    retrieveSortHash() {
      const hash = window.hashParams;
      if (hash.zCST) {
        this.commitsSortType = hash.zCST;
      }
      if (hash.zRSC) {
        this.toReverseSortedCommits = (hash.zRSC === 'true');
      }
    },

    retrieveSelectedFileTypesHash() {
      const hash = window.hashParams;

      if (hash.zFT) {
        this.selectedFileTypes = hash.zFT
            .split(window.HASH_DELIMITER)
            .filter((fileType) => this.fileTypes.includes(fileType));
      }
    },

    updateSelectedFileTypesHash() {
      const fileTypeHash = this.selectedFileTypes.length > 0
          ? this.selectedFileTypes.reduce((a, b) => `${a}~${b}`)
          : '';

      window.addHash('zFT', fileTypeHash);
      window.encodeHash();
    },

    setInfoHash() {
      const { addHash, encodeHash } = window;
      const {
        zAvgCommitSize, zSince, zUntil, zFilterGroup,
        zTimeFrame, zIsMerged, zAuthor, zRepo, zFromRamp, zFilterSearch,
      } = this.info;
      addHash('zA', zAuthor);
      addHash('zR', zRepo);
      addHash('zACS', zAvgCommitSize);
      addHash('zS', zSince);
      addHash('zFS', zFilterSearch);
      addHash('zU', zUntil);
      addHash('zMG', zIsMerged);
      addHash('zFTF', zTimeFrame);
      addHash('zFGS', zFilterGroup);
      addHash('zFR', zFromRamp);
      encodeHash();
    },

    toggleAllCommitMessagesBody(isActive) {
      this.showAllCommitMessageBody = isActive;

      const toRename = this.showAllCommitMessageBody
        ? 'commit-message message-body active'
        : 'commit-message message-body';

      const commitMessageClasses = document.getElementsByClassName('commit-message message-body');
      Array.from(commitMessageClasses).forEach((commitMessageClass) => {
        commitMessageClass.className = toRename;
      });

      this.expandedCommitMessagesCount = isActive ? this.totalCommitMessageBodyCount : 0;
    },

    updateExpandedCommitMessagesCount() {
      this.expandedCommitMessagesCount = document.getElementsByClassName('commit-message message-body active')
          .length;
    },

    removeZoomHashes() {
      window.removeHash('zA');
      window.removeHash('zR');
      window.removeHash('zFS');
      window.removeHash('zACS');
      window.removeHash('zS');
      window.removeHash('zU');
      window.removeHash('zFGS');
      window.removeHash('zFTF');
      window.removeHash('zMG');
      window.removeHash('zFT');
      window.removeHash('zCST');
      window.removeHash('zRSC');
      window.encodeHash();
    },

    containsAtLeastOneSelected(fileTypes) {
      for (let i = 0; i < fileTypes.length; i += 1) {
        if (this.selectedFileTypes.includes(fileTypes[i])) {
          return true;
        }
      }
      return false;
    },

    getFontColor,
  },
  created() {
    this.initiate();
    this.retrieveHashes();
    this.setInfoHash();
  },
  beforeUnmount() {
    this.removeZoomHashes();
  },
};

</script>

<style lang="scss" scoped>
@import '../styles/_colors.scss';

#tab-zoom {
  .zoom {
    &__title {
      &--granularity {
        @include mini-font;
        margin-top: .5rem;
      }

      &--tags {
        margin: .25rem 0 .25rem 0;

        .tag {
          cursor: pointer;
        }
      }
    }

    &__toggle-commit-message-body {
      padding-top: 10px;
    }

    &__day,
    &__title {
      /* Tags in commits */
      .tag {
        @include mini-font;
        background: mui-color('grey', '600');
        border-radius: 5px;
        color: mui-color('white');
        display: inline-block;
        margin: .2rem .2rem .2rem 0;
        padding: 0 3px 0 3px;

        .fa-tags {
          width: .65rem;
        }
      }
    }
  }

  /* Commit Message Body in Zoom Tab */
  .commit-message {
    border: 1px solid transparent;
    padding: 5px;

    &:focus,
    &:focus-within {
      border: 1px solid mui-color('blue', '500');
    }

    &.active {
      .body {
        background-color: mui-color('white');
        border: 1px solid mui-color('grey', '700');
        display: grid;
        margin: .25rem 0 .25rem 0;
        overflow-x: auto;
        padding: .4rem;
        resize: none;

        pre {
          position: relative;

          .dashed-border {
            border-right: 1px dashed mui-color('grey', '500'); // 72nd character line
            height: 100%;
            pointer-events: none;
            position: absolute;
            top: 0;
            width: 72ch;
          }
        }
      }
    }

    .body {
      display: none;
    }

    .tag {
      cursor: pointer;

      &:focus {
        border: 1px solid mui-color('blue', '500');
        outline: none;
      }
    }

    &--button {
      color: mui-color('grey');
      padding-left: .5rem;

      &:hover {
        cursor: pointer;
      }
    }

    pre {
      margin: 0;
    }

    span.loc {
      color: mui-color('grey');
    }

    .message-title {
      display: inline;
      font-family: monospace;

      .within-border {
        display: inline;
      }

      .not-within-border {
        border-left: 1px dashed mui-color('grey', '500'); // 50th character line
        display: inline;
      }
    }
  }
}

</style>
