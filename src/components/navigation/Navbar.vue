<template>
  <nav id="lv-navbar">
    <div class="lv-navbar-container">
      <div class="lv-navbar-navs" ref="navs">
        <div
          class="lv-navbar-close"
          @click="toggleNavCollapse">
          <icon
            class="close"
            name="times"
            scale="1.5">
          </icon>
        </div>
        <a
          v-for="(item, index) in navigation"
          :key="index"
          :href="item.url"
          :data-brand="item.brand">
          {{ item.label }}
        </a>
      </div>
      <button class="lv-navbar-hamburger" @click="toggleNavCollapse">
        <icon name="bars"></icon>
      </button>
    </div>
  </nav>
</template>

<script>
import Icon from 'vue-awesome/components/Icon'
import 'vue-awesome/icons/bars'
import 'vue-awesome/icons/times'

export default {
  props: {
    navigation: {
      type: Array,
      required: true
    }
  },

  components: {
    Icon
  },

  methods: {
    /**
     * Toggle the collapsible sidenav.
     */
    toggleNavCollapse () {
      this.$refs.navs.classList.toggle('show')
    }
  }
}
</script>
