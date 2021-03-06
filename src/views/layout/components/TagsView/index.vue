<template>
    <div id="tags-view-container" class="tags-view-container">
        <scroll-pane ref="scrollPane" class="tags-view-wrapper">
            <router-link
                    v-for="tag in visitedViews"
                    ref="tag"
                    :key="tag.path"
                    :class="isActive(tag)?'active':''"
                    :to="{ path: tag.path, query: tag.query, fullPath: tag.fullPath }"
                    tag="span"
                    class="tags-view-item"
                    @click.middle.native="closeSelectedTag(tag)"
                    @contextmenu.prevent.native="openMenu(tag,$event)">

                {{ tag.title }}
                <span v-if="!tag.meta.affix" class="el-icon-close" @click.prevent.stop="closeSelectedTag(tag)" />
            </router-link>
        </scroll-pane>
        <ul v-show="visible" :style="{left:left+'px',top:top+'px'}" class="contextmenu">
            <li v-if="!(selectedTag.meta&&selectedTag.meta.affix)" @click="closeSelectedTag(selectedTag)">
                Close
            </li>
            <li @click="closeOthersTags">
                Close Others
            </li>
            <li @click="closeAllTags(selectedTag)">
                Close All
            </li>
        </ul>
    </div>
</template>

<script lang="ts" >
    import {Vue, Component, Watch} from 'vue-property-decorator';
import ScrollPane from "./ScrollPane.vue";
import {TagsViewModule} from "@/store/tagsview.module";
import {PermissionModule} from "@/store/permission.module";
    import * as path from "path";
    import {Route} from "vue-router";

@Component({
    components: {
        ScrollPane
    }
})
export default class TagsView extends Vue {
    private visible = false;
    private top = 0;
    private left = 0;
    private selectedTag = {};
    private affixTags = [];


    get visitedViews() {
        return TagsViewModule.visitedViews;
    }

    get routes() {
        return PermissionModule.routers;
    }

    @Watch('$route')
    OnRouteChanged(newVal, oldVal) {
        this.addTags();
        this.moveToCurrentTag();
    }

    @Watch('visible')
    OnVisibleChanged(newVal) {
        if (newVal) {
            document.body.addEventListener('click', this.closeMenu)
        } else {
            document.body.removeEventListener('click', this.closeMenu)
        }
    }

    mounted() {
        this.initTags();
        this.addTags();
    }

    isActive(route) {
        return route.path === this.$route.path
    }

    filterAffixTags(routes, basePath = '/') {
        let tags: any = []
        routes.forEach(route => {
            if (route.meta && route.meta.affix) {
                const tagPath = path.resolve(basePath, route.path)
                tags.push({
                    fullPath: tagPath,
                    path: tagPath,
                    name: route.name,
                    meta: { ...route.meta }
                })
            }
            if (route.children) {
                const tempTags = this.filterAffixTags(route.children, route.path)
                if (tempTags.length >= 1) {
                    tags = [...tags, ...tempTags]
                }
            }
        })
        return tags
    }

    initTags() {
        const affixTags = this.affixTags = this.filterAffixTags(this.routes)
        for (const tag of affixTags) {
            // Must have tag name
            if (tag.name) {
                TagsViewModule.addVisitedView(tag);
            }
        }
    }

    addTags() {
        const { name } = this.$route;
        if (name) {
            TagsViewModule.addView(this.$route);
        }
        return false
    }

    moveToCurrentTag() {
        const tags: any = this.$refs.tag;
        if (tags) {
            this.$nextTick(() => {
                for (const tag of tags) {
                    if (tag.to.path === this.$route.path) {
                        (this.$refs.scrollPane as any).moveToTarget(tag)
                        // when query is different then update
                        if (tag.to.fullPath !== this.$route.fullPath) {
                            TagsViewModule.updateVisitedView(this.$route);
                        }
                        break
                    }
                }
            })
        }
    }

    refreshSelectedTag(view) {
        TagsViewModule.delCachedView(view).then(() => {
            const { fullPath } = view
            this.$nextTick(() => {
                this.$router.replace({
                    path: '/redirect' + fullPath
                })
            })
        })
    }

    closeSelectedTag(view) {
        TagsViewModule.delView(view).then(() => {
            if (this.isActive(view)) {
                this.toLastView(TagsViewModule.visitedViews)
            }
        })
    }

    closeOthersTags() {
        this.$router.push(this.selectedTag)
        TagsViewModule.delOthersViews(this.selectedTag).then(() => {
            this.moveToCurrentTag()
        })
    }

    closeAllTags(view: Route) {
        TagsViewModule.delAllViews(view).then(() => {
            if (this.affixTags.some((tag: Route) => tag.path === view.path)) {
                return
            }
            this.toLastView(TagsViewModule.visitedViews)
        })
    }

    toLastView(visitedViews) {
        const latestView = visitedViews.slice(-1)[0];
        if (latestView) {
            this.$router.push(latestView)
        } else {
            // You can set another route
            this.$router.push('/')
        }
    }

    openMenu(tag, e) {
        const menuMinWidth = 105
        const offsetLeft = this.$el.getBoundingClientRect().left // container margin left
        const offsetWidth = (this.$el as any).offsetWidth // container width
        const maxLeft = offsetWidth - menuMinWidth // left boundary
        const left = e.clientX - offsetLeft + 15 // 15: margin right

        if (left > maxLeft) {
            this.left = maxLeft
        } else {
            this.left = left
        }

        this.top = e.clientY
        this.visible = true
        this.selectedTag = tag
    }

    closeMenu() {
        this.visible = false
    }

}
</script>

<style lang="scss" scoped>
    .tags-view-container {
        height: 34px;
        width: 100%;
        background: #fff;
        border-bottom: 1px solid #d8dce5;
        box-shadow: 0 1px 3px 0 rgba(0, 0, 0, .12), 0 0 3px 0 rgba(0, 0, 0, .04);
        .tags-view-wrapper {
            .tags-view-item {
                display: inline-block;
                position: relative;
                cursor: pointer;
                height: 26px;
                line-height: 26px;
                border: 1px solid #d8dce5;
                color: #495060;
                background: #fff;
                padding: 0 8px;
                font-size: 12px;
                margin-left: 5px;
                margin-top: 4px;
                &:first-of-type {
                    margin-left: 15px;
                }
                &:last-of-type {
                    margin-right: 15px;
                }
                &.active {
                    background-color: #42b983;
                    color: #fff;
                    border-color: #42b983;
                    &::before {
                        content: '';
                        background: #fff;
                        display: inline-block;
                        width: 8px;
                        height: 8px;
                        border-radius: 50%;
                        position: relative;
                        margin-right: 2px;
                    }
                }
            }
        }
        .contextmenu {
            margin: 0;
            background: #fff;
            z-index: 3000;
            position: absolute;
            list-style-type: none;
            padding: 5px 0;
            border-radius: 4px;
            font-size: 12px;
            font-weight: 400;
            color: #333;
            box-shadow: 2px 2px 3px 0 rgba(0, 0, 0, .3);
            li {
                margin: 0;
                padding: 7px 16px;
                cursor: pointer;
                &:hover {
                    background: #eee;
                }
            }
        }
    }
</style>

<style lang="scss">
    //reset element css of el-icon-close
    .tags-view-wrapper {
        .tags-view-item {
            .el-icon-close {
                width: 16px;
                height: 16px;
                vertical-align: 2px;
                border-radius: 50%;
                text-align: center;
                transition: all .3s cubic-bezier(.645, .045, .355, 1);
                transform-origin: 100% 50%;
                &:before {
                    transform: scale(.6);
                    display: inline-block;
                    vertical-align: -3px;
                }
                &:hover {
                    background-color: #b4bccc;
                    color: #fff;
                }
            }
        }
    }
</style>
