<template>
  <div
    :class="$style.myComputer"
    class="no-border"
  >
    <div :class="$style.pathBar">
      <span
        class="back"
        :class="path.length === 0 ? 'disabled' : ''"
        @click="back"
      />
      <div class="path">
        {{ pathBar }}
      </div>
      <input
        v-model="search"
        class="search"
        :placeholder="searchPlaceholder"
      >
    </div>
    <FilesContainer
      :class="$style.content"
      v-bind="filesContainerProps"
      :file-props="{ darkText: true, onClick: click,}"
    />
  </div>
</template>

<script>
import { basename } from 'path-browserify';
import NavigateSound from '../../assets/sounds/navigate.wav?url';
import { rgba } from '../../styles/utils';
import backIcon from '../../assets/icons/back.png?url';
import FilesContainer from '../../components/FilesContainer.vue';
import { props, inject } from '../../utils/vue';
import { reverseSlash } from '../../services/fs';
import { getFileType } from '../../utils/utils';
import playSound from '../../services/snd';

export default {
  ...inject('$fs', '$wm', '$snd'),
  ...props({
    filePath: props.obj(null),
    wmId: props.any(),
  }),
  components: {
    FilesContainer,
  },
  data() {
    const hasFile = this.filePath;
    return {
      path: hasFile ? this.filePath : '/',
      search: null,
    };
  },
  computed: {
    pathBar() {
      if (!this.path || this.path === '/') {
        return 'My Computer';
      }
      const reversed = reverseSlash(this.path);
      return reversed.startsWith('\\') ? reversed.slice(1) : reversed;
    },
    searchPlaceholder() {
      const { path } = this;
      const name = basename(path) || 'Computer';
      return `Search in ${name}`;
    },
    filesContainerProps() {
      const obj = {
        path: this.path || '/',
      };
      if (this.search && this.search.trim().length) {
        obj.search = this.search;
      }

      return obj;
    },
  },
  watch: {
    path() {
      this.updateTaskbar();
    },
  },
  methods: {
    click(filePath) {
      const fileType = getFileType(filePath);
      if (fileType === 'directory') {
        playSound(NavigateSound);
        this.path = filePath;
        return true;
      }
      return false;
    },
    back() {
      playSound(NavigateSound);
      if (this.search) {
        this.search = null;
        return;
      }
      if (this.path.includes('/') && this.path.length > 1) {
        this.path = this.path.substr(0, this.path.lastIndexOf('/'));
      } else {
        this.path = '/';
      }
    },
    updateTaskbar() {
      this.$wm.updateToolbarTitle(this.wmId, this.path ? basename(this.path) : 'Computer');
    },
  },
  style({ className }) {
    return [
      className('myComputer', {
        display: 'flex',
        flexDirection: 'column',
      }),
      className('pathBar', {
        display: 'flex',
        flexDirection: 'row',
        alignItems: 'center',
        flexWrap: 'nowrap',
        marginBottom: '6px',
        '& > .back': {
          backgroundImage: `url("${backIcon}")`,
          width: '32px',
          height: '32px',
          backgroundRepeat: 'no-repeat',
          backgroundPosition: 'center',
          backgroundSize: 'contain',
          cursor: 'pointer',
          marginRight: '5px',
          transition: 'filter 0.1s',
          '&:not(.disabled):hover, &:not(.disabled):focus': {
            filter: 'brightness(1.2)',
          },
          '&:not(.disabled):active': {
            filter: 'brightness(0.8)',
          },
          '&.disabled': {
            filter: 'grayscale(1)',
          },
        },
        '& > .path, & > .search': {
          height: '24px',
          lineHeight: '24px',
          padding: '0 5px',
          fontSize: '15px',
          border: `solid 1px ${rgba(1, 0.5)}`,
          background: rgba(255, 1),
          borderRadius: '2px',
        },
        '& > .path': {
          flexGrow: 1,
          color: rgba(0, 1),
          marginRight: '5px',
        },
      }),
      className('content', {
        background: rgba(255, 1),
        flexGrow: 1,
        position: 'relative',
      }),
    ];
  },
};
</script>
