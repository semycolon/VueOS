<script>
import { h } from 'vue';
import { fetchApp } from '../../services/apps';
import Loading from './Loading.vue';

export default {
  name: 'WindowsContent',
  props: {
    window: {
      required: true,
      type: Object,
    },
  },
  data() {
    return {
      loading: false,
      module: null,
    };
  },
  async created() {
    const m = await fetchApp(this.window.appName, this.window.isSystemApp);
    this.module = m.default ? m.default : m;
  },
  render() {
    if (this.loading) {
      return h(Loading);
    }
    if (!this.module) {
      return null;
    }
    return h(this.module, this.window);
  },
};
</script>
