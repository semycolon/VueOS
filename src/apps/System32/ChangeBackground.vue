<template>
  <div :class="$style.fill">
    <div :class="$style.content">
      <h4 :class="$style.head">
        Choose your desktop background
      </h4>
      <p :class="$style.desc">
        Click a picture to make it your desktop background.
      </p>
      <div :class="$style.box">
        <img
          v-for="(item, key) in list"
          :key="key"
          width="90"
          height="60"
          :src="item.src"
          alt=""
          @click="changeBackground(item)"
        >

        <div
          v-if="loading"
          class="loading"
        >
          <p>loading...</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { props, inject } from '../../utils/vue';
import { px } from '../../styles/utils';
import { downloadDefaultWallpapers, fetchFile, readDirectory } from '../../services/fs';

export default {
  name: 'ChangeBackground',
  ...inject('$wm', '$fs', '$cnf'),
  ...props({
    file: props.obj(null),
    wmId: props.any(),
  }),
  data() {
    return {
      list: [],
      loading: true,
    };
  },
  async mounted() {
    this.loading = true;
    // todo show downloading ...
    await downloadDefaultWallpapers();

    const wallpapersFiles = await readDirectory('/C:/Windows/Wallpapers');
    const self = this;
    const list = [];
    // eslint-disable-next-line no-restricted-syntax
    wallpapersFiles.forEach((file) => {
      const promise = new Promise((resolve) => {
        fetchFile(file)
          .then((buffer) => {
            self.list.push({
              file,
              src: URL.createObjectURL(new Blob([buffer])),
            });
            resolve();
          });
      });
      list.push(promise);
    });
    await Promise.all(list);
    this.loading = false;
  },
  methods: {
    changeBackground(item) {
      this.$cnf.setConfig({
        wallpaperPath: item.file,
      });
    },
  },
  style({ className }) {
    return [
      className('fill', {
        background: 'white',
        padding: '1rem',
      }),
      className('content', {
        width: px(800),
        margin: '0 auto',
        background: 'white',
      }),
      className('box', {
        width: '100%',
        height: px(300),
        overflowY: 'scroll',
        overflowX: 'hide',
        border: '1px solid black',
        borderRadius: px(2),
        padding: '1rem 0',
        textAlign: 'center',
        img: {
          margin: '.5rem',
        },
      }),
      className('head', {
        fontSize: '1.25rem',
        marginBottom: px(16),
      }),
      className('desc', {
        marginBottom: '2rem',
      }),
    ];
  },
};
</script>
