<template>

  <CoreFullscreen
    ref="html5Renderer"
    class="html5-renderer"
    @changeFullscreen="isInFullscreen = $event"
  >
    <UiIconButton
      class="btn"
      :style="{ fill: $themeTokens.textInverted }"
      :ariaLabel="isInFullscreen ? $tr('exitFullscreen') : $tr('enterFullscreen')"
      color="primary"
      size="large"
      @click="$refs.html5Renderer.toggleFullscreen()"
    >
      <KIcon v-if="isInFullscreen" icon="fullscreen_exit" />
      <KIcon v-else icon="fullscreen" />
    </UiIconButton>
    <div class="iframe-container">
      <iframe
        ref="iframe"
        class="iframe"
        :style="{ backgroundColor: $themePalette.grey.v_100 }"
        sandbox="allow-scripts"
        frameBorder="0"
        :name="name"
        :src="rooturl"
      >
      </iframe>
    </div>
  </CoreFullscreen>

</template>


<script>

  import { mapGetters } from 'vuex';
  import { now } from 'kolibri.utils.serverClock';
  import UiIconButton from 'kolibri-design-system/lib/keen/UiIconButton';
  import CoreFullscreen from 'kolibri.coreVue.components.CoreFullscreen';
  import Hashi from 'hashi';
  import { nameSpace } from 'hashi/src/hashiBase';

  export default {
    name: 'Html5AppRendererIndex',
    components: {
      UiIconButton,
      CoreFullscreen,
    },
    data() {
      return {
        isInFullscreen: false,
      };
    },
    computed: {
      ...mapGetters(['summaryTimeSpent']),
      name() {
        return nameSpace;
      },
      rooturl() {
        return this.defaultFile.storage_url;
      },
    },
    mounted() {
      this.hashi = new Hashi({ iframe: this.$refs.iframe, now });
      this.hashi.onStateUpdate(data => {
        this.$emit('updateContentState', data);
      });
      this.hashi.initialize((this.extraFields && this.extraFields.contentState) || {});
      this.$emit('startTracking');
      this.pollProgress();
    },
    beforeDestroy() {
      if (this.timeout) {
        clearTimeout(this.timeout);
      }
      this.$emit('stopTracking');
    },
    methods: {
      recordProgress() {
        const totalTime = this.summaryTimeSpent * 1000;
        this.$emit('updateProgress', Math.max(0, totalTime / 300000));
        this.pollProgress();
      },
      pollProgress() {
        this.timeout = setTimeout(() => {
          this.recordProgress();
        }, 15000);
      },
    },
    $trs: {
      exitFullscreen: 'Exit fullscreen',
      enterFullscreen: 'Enter fullscreen',
    },
  };

</script>


<style lang="scss" scoped>

  @import '~kolibri-design-system/lib/styles/definitions';

  .btn {
    position: absolute;
    top: 8px;
    right: 21px;
    z-index: 1;
  }

  .html5-renderer {
    position: relative;
    height: 500px;
    text-align: center;
  }

  .iframe {
    width: 100%;
    height: 100%;
  }

  .iframe-container {
    @extend %momentum-scroll;

    position: absolute;
    top: 0;
    bottom: 0;
    width: 100%;
    height: 100%;
    overflow: visible;
  }

</style>
