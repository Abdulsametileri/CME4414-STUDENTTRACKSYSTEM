<template>
  <div :class="classObj" class="app-wrapper">
    <div v-if="classObj.mobile && sidebar.opened" class="drawer-bg" @click="handleClickOutside"/>
    <sidebar class="sidebar-container" :collapse="classObj.hideSidebar"/>
    <div class="main-container">
      <navbar/>
      <tags-view />
      <app-main/>
    </div>
  </div>
</template>

<script lang="ts">
import { Navbar, AppMain, Sidebar } from './components';
import ResizeMixin from './mixin/ResizeHandler';
import { Component, Vue } from 'vue-property-decorator';
import { mixins } from 'vue-class-component';
import { DeviceType, AppModule } from '@/store/app.module';
import TagsView from "@/views/layout/components/TagsView/index.vue";
import {AuthModule} from "@/store/auth.module";

@Component({
  components: {
    Navbar,
    Sidebar,
    AppMain,
    TagsView
  },
})
export default class Layout extends mixins(ResizeMixin) {
  get classObj() {
    return {
      hideSidebar: !this.sidebar.opened,
      openSidebar: this.sidebar.opened,
      withoutAnimation: this.sidebar.withoutAnimation,
      mobile: this.device === DeviceType.Mobile,
    };
  }

  private handleClickOutside() {
    AppModule.CloseSideBar(false);
  }

  created() {
    console.log(AuthModule.User.user_type)
    if (AuthModule.User.user_type == 8) {
      this.$router.push({path: '/daily-study'});
    }
    else
      this.$router.push({path: '/report/dailystudy'})
  }



}
</script>

<style rel="stylesheet/scss" lang="scss" scoped>
  @import "src/styles/mixin.scss";
  .app-wrapper {
    @include clearfix;
    position: relative;
    height: 100%;
    width: 100%;
    &.mobile.openSidebar{
      position: fixed;
      top: 0;
    }
  }
  .drawer-bg {
    background: #000;
    opacity: 0.3;
    width: 100%;
    top: 0;
    height: 100%;
    position: absolute;
    z-index: 999;
  }
</style>
