<template>
  <div>
    <nuxt-child />
    <Images
      class="content"
      data-test="imagesTable"
      :images="images"
      :image-namespaces="imageNamespaces"
      :state="state"
      :show-all="settings.images.showAll"
      :selected-namespace="settings.images.namespace"
      :supports-namespaces="supportsNamespaces"
      @toggledShowAll="onShowAllImagesChanged"
      @switchNamespace="onChangeNamespace"
    />
  </div>
</template>

<script>
import { ipcRenderer } from 'electron';
import { mapGetters } from 'vuex';
import Images from '@/components/Images.vue';
import * as K8s from '@/k8s-engine/k8s';
import { defaultSettings } from '@/config/settings';

export default {
  components: { Images },
  data() {
    return {
      settings:           defaultSettings,
      images:             [],
      imageNamespaces:    [],
      supportsNamespaces: true,
    };
  },

  computed: {
    state() {
      if (![K8s.State.STARTED, K8s.State.DISABLED].includes(this.k8sState)) {
        return 'IMAGE_MANAGER_UNREADY';
      }

      return this.imageManagerState ? 'READY' : 'IMAGE_MANAGER_UNREADY';
    },
    ...mapGetters('k8sManager', { k8sState: 'getK8sState' }),
    ...mapGetters('imageManager', { imageManagerState: 'getImageManagerState' })
  },

  watch: {
    imageManagerState: {
      handler(state) {
        this.$store.dispatch(
          'page/setHeader',
          { title: this.t('images.title') }
        );

        if (!state) {
          return;
        }

        this.$store.dispatch(
          'page/setAction',
          { action: 'images-button-add' }
        );
      },
      immediate: true
    }
  },

  mounted() {
    ipcRenderer.on('images-changed', (event, images) => {
      this.$data.images = images;
      if (this.supportsNamespaces && this.imageNamespaces.length === 0) {
        // This happens if the user clicked on the Images panel before data was ready,
        // so no namespaces were available when it initially asked for them.
        // When the data is ready, images are pushed in, but namespaces aren't.
        ipcRenderer.send('images-namespaces-read');
      }
    });

    ipcRenderer.on('images-check-state', (event, state) => {
      this.$store.dispatch('imageManager/setImageManagerState', state);
    });

    ipcRenderer.invoke('images-check-state').then((state) => {
      this.$store.dispatch('imageManager/setImageManagerState', state);
    });

    ipcRenderer.on('settings-update', (event, settings) => {
      // TODO: put in a status bar
      this.$data.settings = settings;
      this.checkSelectedNamespace();
    });
    (async() => {
      this.$data.images = await ipcRenderer.invoke('images-mounted', true);
    })();

    ipcRenderer.on('images-namespaces', (event, namespaces) => {
      // TODO: Use a specific message to indicate whether messages are supported or not.
      this.$data.imageNamespaces = namespaces;
      this.$data.supportsNamespaces = namespaces.length > 0;
      this.checkSelectedNamespace();
    });
    ipcRenderer.send('images-namespaces-read');
    ipcRenderer.on('settings-read', (event, settings) => {
      this.$data.settings = settings;
    });
    ipcRenderer.send('settings-read');
  },
  beforeDestroy() {
    ipcRenderer.invoke('images-mounted', false);
  },

  methods: {
    checkSelectedNamespace() {
      if (!this.supportsNamespaces || this.imageNamespaces.length === 0) {
        // Nothing to verify yet
        return;
      }
      if (!this.imageNamespaces.includes(this.settings.images.namespace)) {
        const K8S_NAMESPACE = 'k8s.io';
        const defaultNamespace = this.imageNamespaces.includes(K8S_NAMESPACE) ? K8S_NAMESPACE : this.imageNamespaces[0];

        ipcRenderer.invoke('settings-write',
          { images: { namespace: defaultNamespace } } );
      }
    },
    onShowAllImagesChanged(value) {
      if (value !== this.settings.images.showAll) {
        ipcRenderer.invoke('settings-write',
          { images: { showAll: value } } );
      }
    },
    onChangeNamespace(value) {
      if (value !== this.settings.images.namespace) {
        ipcRenderer.invoke('settings-write',
          { images: { namespace: value } } );
      }
    }
  }
};
</script>
