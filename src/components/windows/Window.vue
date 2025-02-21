<template>
  <div
    v-show="!window.hidden"
    :id="'win-'+window.id"
    :class="[
      $style.window, focused && 'focused',
      window.maximized && 'maximized',
      window.minimized && 'minimized',
      window.resizable && 'resizable'
    ]"
    :style="{ zIndex: window.zIndex }"
    @click.capture="focus"
  >
    <div :class="$style.titlebar">
      <div
        ref="title"
        class="title"
      >
        <img :src="window.icon">
        {{ window.title }}
      </div>
      <div
        class="buttons"
        :class="focused && 'focused'"
      >
        <div
          v-if="window.minimizable"
          class="minimize"
          @click="minimize"
        >
          <img :src="icons.minimize">
        </div>
        <div
          v-if="window.maximizable"
          class="maximize"
          @click="maximize"
        >
          <img
            v-if="window.maximized"
            :src="icons.unmaximize"
          >
          <img
            v-else
            :src="icons.maximize"
          >
        </div>
        <div
          v-if="window.closable"
          class="close"
          @click="close"
        >
          <img :src="icons.close">
        </div>
      </div>
    </div>
    <WindowsContent
      ref="content"
      :key="window.id"
      :window="window"
      :class="$style.content"
      :wm-id="window.id"
    />
  </div>
</template>

<script>
import { inject, props } from '../../utils/vue';
import { each } from '../../utils/utils';
import { rgba, px } from '../../styles/utils';
import { panelSize } from '../../styles/constants';
import swipe from '../../utils/swipe';
import MaximizeIcon from '../../assets/window/maximize.png';
import UnmaximizeIcon from '../../assets/window/unmaximize.png';
import MinimizeIcon from '../../assets/window/minimize.png';
import CloseIcon from '../../assets/window/close.png';
import WindowsContent from './WindowsContent.vue';

export default {
  components: { WindowsContent },
  ...props({
    window: props.obj(),
  }),
  ...inject('$wm'),
  data() {
    return {
      mover: null,
      moveStartPosition: null,
    };
  },
  computed: {
    icons() {
      return {
        maximize: MaximizeIcon,
        unmaximize: UnmaximizeIcon,
        minimize: MinimizeIcon,
        close: CloseIcon,
      };
    },
    focused() {
      return this.$wm.isWindowFocused(this.window.id);
    },
  },
  mounted() {
    this.setPosition({
      height: this.window.height,
      width: this.window.width,
      left: this.window.left,
      top: this.window.top,
    });
    if (this.window.movable) {
      this.mover = swipe(
        this.$refs.title,
        this.moveStart,
        this.whileMove,
        this.moveEnd,
      );
    }
  },
  beforeUnmount() {
    if (this.mover) {
      this.mover.stop();
    }
  },
  methods: {
    close() {
      this.$wm.closeWindow(this.window.id);
    },
    focus() {
      this.$wm.focusWindow(this.window.id);
    },
    moveStart() {
      if (this.window.maximized) {
        this.maximize();
      }
      if (!this.focused) {
        this.focus();
      }
      this.moveStartPosition = this.getPosition();
      return true;
    },
    whileMove({ left, top }) {
      this.setPosition({
        left: left + this.moveStartPosition.left,
        top: top + this.moveStartPosition.top,
      });
    },
    moveEnd({ left, top }) {
      this.setPosition({
        left: left + this.moveStartPosition.left,
        top: top + this.moveStartPosition.top,
      });
      each(this.getPosition(), (key, value) => {
        this.window[key] = value;
      });
    },
    minimize() {
      this.$wm.minimizeWindow(this.window.id);
    },
    maximize() {
      this.$wm.maximizeWindow(this.window.id);
    },
    setPosition(position) {
      each(position, (key, value) => {
        this.$el.style[key] = px(value);
      });
    },
    getPosition() {
      const keys = ['top', 'left', 'width', 'height'];
      const ret = {};
      keys.forEach((key) => {
        ret[key] = this.$el[`offset${key[0].toUpperCase()}${key.slice(1)}`];
      });
      return ret;
    },
  },
  style({ className }) {
    const baseAlpha = 0.1;
    const titleHeight = '32px';

    return [
      className('window', {
        userSelect: 'none',
        position: 'absolute',
        ...(this.window.resizable && {
          resize: 'both',
        }),
        background: `linear-gradient(180deg,
          ${rgba(250, baseAlpha)} 0%,
          ${rgba(100, baseAlpha)} 50%,
          ${rgba(170, baseAlpha)} 100%
        )`,
        boxShadow: `
          0 0 0 1px ${rgba(255, 0.7)},
          0 0 8px 3px ${rgba(10, 0.6)}
        `,
        backdropFilter: 'blur(7px) brightness(1.1)',
        borderRadius: '8px',
        display: 'flex',
        flexDirection: 'column',
        overflow: 'hidden',
        fontSize: '15px',
        minWidth: '300px',
        minHeight: titleHeight,
        '&.maximized': {
          width: '100% !important',
          height: `calc(100% - ${panelSize}) !important`,
          top: '0 !important',
          left: '0 !important',
          borderRadius: '0 !important',
          resize: 'none !important',
          boxShadow: 'none !important',
        },
        '&.minimized': {
          display: 'none',
        },
        '&:not(.focused)': {
          opacity: 0.9,
          backdropFilter: 'blur(4px)',
          boxShadow: `
            0 0 0 1px ${rgba(255, 0.9)},
            0 0 8px 3px ${rgba(10, 0.4)}
          `,
        },
      }),
      className('titlebar', {
        height: titleHeight,
        display: 'flex',
        flexDirection: 'row',
        alignItems: 'center',
        padding: '0 10px',
        '& > .title': {
          textShadow: new Array(4).fill(`0 0 5px ${rgba(255, 1)}`).join(','),
          flexGrow: 1,
          height: '100%',
          lineHeight: titleHeight,
          '& > img': {
            width: '20px',
            marginRight: '5px',
            verticalAlign: 'middle',
          },
        },
        '& > .buttons': {
          display: 'flex',
          flexDirection: 'row',
          height: '18px',
          alignSelf: 'flex-start',
          boxShadow: `
            inset 0 0 1px 1px ${rgba(255, 0.4)},
            0 0 0 2px ${rgba(0, 0.2)}
          `,
          borderTop: 'none',
          borderBottomLeftRadius: '5px',
          borderBottomRightRadius: '5px',
          overflow: 'hidden',
          '& > *': {
            textAlign: 'center',
            lineHeight: '18px',
            fontSize: '12px',
            fontWeight: 'bold',
            color: '#fff',
            width: '26px',
            background: `linear-gradient(180deg,
              ${rgba(200, 0.5)} 0%,
              ${rgba(200, 0.5)} 40%,
              ${rgba(110, 0.4)} 60%,
              ${rgba(150, 0.4)} 90%,
              ${rgba(180, 0.4)} 100%
            )`,
            backdropFilter: 'grayscale(1)',
            '& > img': {
              height: '13px',
              marginTop: '2px',
              filter: `drop-shadow(0 0 2px ${rgba(0, 1)})`,
            },
            '&.maximize > img': {
              height: '11px',
              marginTop: '3px',
            },

            '&:not(:last-child)': {
              borderRight: `solid 1px ${rgba(0, 0.4)}`,
            },
            '&:hover': {
              filter: 'brightness(1.3)',
            },
            '&:active': {
              filter: 'brightness(0.9)',
            },
          },
          '& > .close': {
            width: '42px',
          },
          '&.focused > .close': {
            background: `linear-gradient(180deg,
              rgba(255, 107, 63, 0.8) 0%,
              rgba(255, 104, 79, 0.7) 40%,
              rgba(175, 61, 44, 0.9) 55%,
              rgba(204, 33, 33, 0.7) 80%,
              rgba(255, 125, 125, 0.9) 100%
            )`,
          },
        },
      }),
      className('content', {
        flexGrow: 1,
        border: `solid 1px ${rgba(10, 0.5)}`,
        overflow: 'hidden',
        margin: '0 5px 5px 5px',
        maxWidth: 'calc(100% - 10px)',
        '&.no-border': {
          border: 0,
        },
      }),
    ];
  },
};
</script>
