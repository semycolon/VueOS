<template>
  <div
    :class="$style.filesContainer"
    :style="{cursor:loading ? 'wait' : 'inherit'}"
    @contextmenu="openContextMenu"
  >
    <File
      v-for="file in dirFiles"
      :ref="addFileToRefs"
      :key="file.path"
      :file="file"
      v-bind="fileProps"
      @refresh="fetchDirectoryFiles"
    />
    <div
      v-if="selecting"
      ref="selection"
      :class="$style.selection"
    />
  </div>
</template>

<script>
import { dirname } from 'path-browserify';
import { rgba, px } from '../styles/utils';
import { props, provideAs, inject } from '../utils/vue';
import { offsetTo, isDirectory, getFileType } from '../utils/utils';
import swipe from '../utils/swipe';
import { drag, drop } from '../utils/dragndrop';
import File from './File.vue';
import {
  copyDirectory,
  copyFile,
  createNewFolder,
  createNewTextFile,
  deletePath,
  isFile,
  moveFile, registerToFsEvent,
} from '../services/fs';
import { openFileWith } from '../services/wm';

const fixSelectionPosition = (selection) => {
  if (!selection) {
    return null;
  }
  const ret = {};
  if (selection.width < 0) {
    ret.left = selection.left + selection.width;
    ret.width = Math.abs(selection.width);
  } else {
    ret.left = selection.left;
    ret.width = selection.width;
  }
  if (selection.height < 0) {
    ret.top = selection.top + selection.height;
    ret.height = Math.abs(selection.height);
  } else {
    ret.top = selection.top;
    ret.height = selection.height;
  }
  return ret;
};

export default {
  ...inject('$wm', '$fs'),
  components: {
    File,
  },
  ...props({
    // todo default value not working
    path: props.str('/'),
    search: props.str(null),
    fileProps: props.obj(),
    direction: props.oneOf(['row', 'column'], 'row'),
    contextMenuExtras: props.obj(
      {},
      (obj) => typeof obj === 'object' && !Object.keys(obj)
        .find((it) => typeof obj[it] !== 'function'),
    ),
  }),
  ...provideAs('$filesContainer'),
  data() {
    return {
      mover: null,
      dragger: null,
      dropper: null,
      selecting: false,
      fileRefs: [],
      selectionRectangle: null,
      loading: false,
      files: [],
      unRegisterToFsEvent: () => {
      },
    };
  },
  computed: {
    staticPath() {
      return this.path !== null;
    },
    dirFiles() {
      return this.files;
    },
    isInDesktop() {
      return this.path === 'C:/User/Desktop' && this.$parent.$options.name === 'Desktop';
    },
  },
  watch: {
    search() {
      this.fetchDirectoryFiles();
    },
    path() {
      if (this.unRegisterToFsEvent && typeof this.unRegisterToFsEvent === 'function') {
        this.unRegisterToFsEvent();
      }
      this.unRegisterToFsEvent = registerToFsEvent(this.onFsEvent);
      this.fetchDirectoryFiles();
    },
  },
  mounted() {
    this.unRegisterToFsEvent = registerToFsEvent(this.onFsEvent);
    this.mover = swipe(
      this.$el,
      this.selectStart,
      this.whileSelect,
      this.selectEnd,
    );
    this.droper = drop(this.$el, this.path, this.onDrop);
    this.dragger = drag(this.$el, this.onDrag);
    this.fetchDirectoryFiles();
  },
  beforeUnmount() {
    this.unRegisterToFsEvent();
    if (this.mover) {
      this.mover.stop();
    }
    if (this.droper) {
      this.droper.stop();
    }
    if (this.dragger) {
      this.dragger.stop();
    }
  },
  beforeUpdate() {
    this.fileRefs = [];
  },
  methods: {
    onDrag(e) {
      const isOnFile = this.isEventOnFile(e);
      if (isOnFile) {
        if (this.mover) {
          this.mover.cancelMove();
        }
        this.$nextTick(() => {
          this.selecting = false;
          this.$el.style.overflow = null;
        });
        return this.getSelectedFiles()
          .map((file) => file.file);
      }
      return false;
    },
    onDrop(data) {
      this.copyOrMoveFilesHere('move', data);
    },
    getSelectedFiles() {
      const selectedFiles = this.fileRefs.filter((file) => file.selected);
      return selectedFiles;
    },
    isEventOnFile(e) {
      const selectedFiles = this.getSelectedFiles();
      return selectedFiles.some((file) => file.$el.contains(e.target));
    },
    openContextMenu(e) {
      const selectedFiles = this.getSelectedFiles();
      const isOnFile = this.isEventOnFile(e);
      let contextMenuItems = [];
      if (!isOnFile) {
        this.unselectAll();
        contextMenuItems = [
          ...contextMenuItems,
          'Refresh',
          ...(this.staticPath ? ['Create New Folder', 'Create New Text File'] : []),
          ...(this.staticPath && (this.$wm.markedFiles.copyList.length || this.$wm.markedFiles.cutList.length) ? ['Paste'] : []),
          ...Object.keys(this.contextMenuExtras),
        ];
      } else {
        contextMenuItems = [
          ...contextMenuItems,
          'Open',
          'Delete',
          'Cut',
          'Copy',
        ];
        if (selectedFiles.length === 1) {
          contextMenuItems = [
            ...contextMenuItems,
            'Rename',
          ];
        }
        if (selectedFiles.length) {
          const first = selectedFiles[0].file;
          if (!['app','shortcut'].includes(getFileType(first))) {
            contextMenuItems.splice(1, 0, 'Open With');
          }
        }
      }

      this.$wm.openContextMenu(e, contextMenuItems, async (item) => {
        if (item === 'Open') {
          selectedFiles.forEach((file) => file.click(null));
        } else if (item === 'Delete') {
          this.loading = true;
          await Promise.all(selectedFiles.map(async (file) => {
            const { file: filePath } = file;
            try {
              await deletePath(filePath);
            } catch (err) {
              console.error(err);
            }
          }));
          this.loading = false;
        } else if (item === 'Refresh') {
          await this.fetchDirectoryFiles();
        } else if (item === 'Create New Folder') {
          await createNewFolder(this.path);
          await this.fetchDirectoryFiles();
        } else if (item === 'Create New Text File') {
          await createNewTextFile(this.path);
          await this.fetchDirectoryFiles();
        } else if (item === 'Rename') {
          selectedFiles.forEach((file) => file.startRename());
        } else if (item === 'Cut') {
          selectedFiles.forEach((file) => {
            const filePath = file.file;
            this.$wm.markFileForCut(filePath);
          });
        } else if (item === 'Copy') {
          selectedFiles.forEach((file) => {
            const filePath = file.file;
            this.$wm.markFileForCopy(filePath);
          });
        } else if (item === 'Paste') {
          Promise.all([
            this.copyOrMoveFilesHere('copy', this.$wm.markedFiles.copyList),
            this.copyOrMoveFilesHere('move', this.$wm.markedFiles.cutList),
          ])
            .then(() => {
              this.$wm.unmarkFiles();
            });
        } else if (item === 'Open With') {
          const files = selectedFiles.map((it) => it.file);
          await openFileWith(...files);
        } else if (this.contextMenuExtras[item]) {
          if (typeof this.contextMenuExtras[item] === 'function') {
            this.contextMenuExtras[item]();
          } else {
            throw new Error('bad value for extra menu item');
          }
        }
      });
    },
    selectStart(pos) {
      const elRect = offsetTo(this.$el, document.body);
      this.selectionRectangle = {
        left: pos.left - elRect.left,
        top: pos.top - elRect.top,
        width: 0,
        height: 0,
      };
      this.selecting = true;
    },
    whileSelect(pos) {
      this.$el.style.overflow = 'hidden';
      const selection = fixSelectionPosition({
        ...this.selectionRectangle,
        width: pos.left,
        height: pos.top,
      });
      Object.keys(selection || {})
        .forEach((key) => {
          this.$refs.selection.style[key] = px(selection[key]);
        });
    },
    selectEnd(pos) {
      this.$el.style.overflow = null;
      const selectionPos = fixSelectionPosition({
        ...this.selectionRectangle,
        width: pos.left,
        height: pos.top,
      });

      const inRange = (fp, sp) => (
        (
          (fp.left > sp.left && fp.left + fp.width < sp.left + sp.width)
          || (fp.left + fp.width > sp.left && fp.left < sp.left + sp.width)
        )
        && (
          (fp.top > sp.top && fp.top + fp.height < sp.top + sp.height)
          || (fp.top + fp.height > sp.top && fp.top < sp.top + sp.height)
        )
      );
      this.fileRefs.forEach((file) => {
        const el = file.$el;
        const filePos = {
          width: el.offsetWidth,
          height: el.offsetHeight,
          left: el.offsetLeft,
          top: el.offsetTop,
        };
        // eslint-disable-next-line no-param-reassign
        file.selected = inRange(filePos, selectionPos);
      });
      this.selecting = false;
    },
    unselectAll() {
      this.fileRefs.forEach((file) => {
        // eslint-disable-next-line no-param-reassign
        file.selected = false;
      });
    },
    async copyOrMoveFilesHere(action, _files) {
      if (!_files.length) return;
      const files = [..._files];
      if (action === 'move') {
        await Promise.all(files.map((file) => moveFile(file, this.path)));
      }
      if (action === 'copy') {
        await Promise.all(files.filter((file) => isFile(file))
          .map((file) => copyFile(file, this.path)));

        if (files.filter(isDirectory).length) {
          await Promise.all(
            files.filter((file) => isDirectory(file))
              .map((file) => copyDirectory(file, this.path)),
          );
        }
      }
      await this.fetchDirectoryFiles();
    },
    addFileToRefs(el) {
      if (el) {
        this.fileRefs.push(el);
      }
    },
    async fetchDirectoryFiles() {
      this.loading = true;
      this.files = [];
      try {
        if (!this.search) {
          const list = await window.$fs.readDirectory(this.path);
          this.files = [...list];
        } else {
          await window.$fs.searchInDirectory(this.path, this.search, (file) => {
            this.files.push(file);
          });
        }
      } catch (e) {
        console.error(e);
      } finally {
        this.loading = false;
      }
    },
    onFsEvent(name, ...values) {
      const [file] = values;
      if (dirname(file) === this.path) {
        // then we should care what happened
        if (name === 'created') {
          this.files = [...this.files.filter((f) => f !== file), file];
        }
        if (name === 'deleted') {
          this.files = this.files.filter((f) => f !== file);
        }
      }
    },
  },
  style({ className }) {
    return [
      className('filesContainer', {
        position: 'relative',
        flexGrow: 1,
        display: 'flex',
        flexDirection: this.direction,
        flexWrap: 'wrap',
        alignItems: 'flex-start',
        justifyContent: 'flex-start',
        alignContent: 'flex-start',
        width: '100%',
        height: '100%',
        overflow: 'auto',
        '&.refreshing': {
          '& > *': {
            display: 'none',
          },
        },
      }),
      className('selection', {
        // zIndex: 999999,
        background: rgba([26, 86, 148], 0.4),
        border: `solid 1px ${rgba([26, 86, 148], 0.6)}`,
        position: 'absolute',
      }),
    ];
  },
};
</script>
