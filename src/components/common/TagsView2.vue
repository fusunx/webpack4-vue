/*
 * @Author: one-dragon 
 * @Date: 2018-11-14 10:54:18 
 * @Last Modified by: one-dragon
 * @Last Modified time: 2019-01-16 17:17:01
 */
<template>
    <div class="tags_view clear_fix" data-common-tags-view2-box>
        <ScrollBar class="tags_view_scrollbar" horizontalSlide>
            <draggable v-model="dragTagsData" class="drag_box clear_fix" ref="tagsDragBox">
                <el-tag
                    v-if="!tag.hidden"
                    :title="tag.meta.title"
                    v-for="tag in tagsData"
                    :key="tag.path"
                    :class="isActive(tag) ? 'active' : ''"
                    closable
                    :disable-transitions="false"
                    @close="handleClose(tag)">
                    <i class="refresh_icon" v-if="isActive(tag)" @click.stop="refreshRoute" ref="refreshIcon">
                        <svg-icon name="refresh" fill="#606266" width="12" height="12" />
                    </i>
                    <router-link :to="tag.path">{{ tag.meta.title }}</router-link>
                </el-tag>
            </draggable>
        </ScrollBar>
        <RightBar />
        <!--<el-tag
            class="active"
            closable
            :disable-transitions="false"
            @close="handleClose(tag)">
            <a href="#/aa">首页</a>
        </el-tag>-->
    </div>
</template>

<script>
    import Vue from 'vue';
    import { mapState } from 'vuex';
    import draggable from 'vuedraggable';
    import RightBar from '~/components/common/RightBar';
    import ScrollBar from '~/components/common/ScrollBar';
    import { scrollAnimate, TreeData } from '~/assets/js/public';
    export default {
        name: 'tagsview',
        watch: {
            $route() {
                this.addViewTags();
                // this.changeTagsWidth();
            }
        },
        computed: {
            ...mapState({
                tagsData: state => state.tagsView.data
            }),
            // 拖拽时改变数据结构
            dragTagsData: {
                get() {
                    return this.$store.state.tagsView.data;
                },
                set(value) {
                    this.$store.commit('tagsView/CHANGE_DATA', value);
                }
            }
        },
        methods: {
            // 获取当前路由信息
            generateRoute() {
                if (this.$route.name) {
                    return this.$route;
                }
                return false;
            },
            // 往vuex里贯入数据
            addViewTags() {
                const route = this.generateRoute();
                if (!route) {
                    return false;
                }
                // 如果当前路由为三级、三级以上菜单时，需要将之前的父级的name名加入到vuex中，并设置hidden: true，在标签循环中不显示
                let self = this;
                TreeData.get({
                    data: this.$router.options.routes,
                    labelVal: route.name,
                    label: 'name',
                    treeStr: 'name',
                    children: 'children',
                    callback(item, data, currTotalObj, treeStr) {
                        treeStr.split('$&&$').map((item, index) => {
                            if(index != 0 && index != treeStr.split('$&&$').length - 1) {
                                TreeData.get({
                                    data: self.$router.options.routes,
                                    labelVal: item,
                                    label: 'name',
                                    children: 'children',
                                    callback(item, data, currTotalObj, treeStr) {
                                        let obj = JSON.parse(JSON.stringify(item));
                                        obj.hidden = true;
                                        self.$store.commit('tagsView/ADD_DATA', obj);
                                    }
                                })
                            }
                        })
                    }
                })
                this.$store.dispatch('tagsView/addData', route).then(() => {
                    // console.log(this.$store.state.tagsView.data);
                    this.changeScrollbar();
                });
            },
            // 判断选中的tag是否隐藏在左侧或右侧，进行scrollbar滚动显示tag
            changeScrollbar() {
                this.$nextTick(() => {
                    setTimeout(() => {
                        let tags = this.$refs.tagsDragBox.$el.querySelectorAll('.el-tag'); // 所有tags
                        let currTag = null; // 当前tag
                        let view = this.$refs.tagsDragBox.$el.parentNode; // scrollbar里的.el-scrollbar__view
                        let warp = this.$refs.tagsDragBox.$el.parentNode.parentNode; // scrollbar里的.el-scrollbar__wrap
                        let warpScrollLeft = warp.scrollLeft; // warp的scrollLeft值
                        // 判断哪个为当前选中tag
                        Array.from(tags).map((item) => {
                            if(item.className.indexOf('active') > -1) {
                                currTag = item;
                            }
                        })
                        // 如果currTag.offsetLeft < warp.scrollLeft，表示当前标签在隐藏左侧
                        if(currTag.offsetLeft <= warpScrollLeft) {
                            // warp.scrollLeft = currTag.offsetLeft;
                            scrollAnimate(warp, 'left', currTag.offsetLeft);
                        }
                        // 如果currTag的offsetLeft + offsetWidth 大于 view的offsetWidth + warp.scrollLeft，表示当前标签在隐藏右侧
                        if((currTag.offsetLeft + currTag.offsetWidth) >= (view.offsetWidth + warpScrollLeft)) {
                            // warp.scrollLeft = Math.ceil((currTag.offsetLeft + currTag.offsetWidth) - view.offsetWidth);
                            scrollAnimate(warp, 'left', Math.ceil((currTag.offsetLeft + currTag.offsetWidth) - view.offsetWidth));
                        }
                    }, 300)
                })
            },
            // 判断是否选中当前tag
            isActive(route) {
                return route.path === this.$route.path;
            },
            // 关闭tab操
            handleClose(route) {
                // 清除当前路由对应的vuex数据
                if(route.meta && route.meta.storeName) {
                    if(this.$store._mutations[`${route.meta.storeName}/CLEAR_DATA`]) {
                        this.$store.commit(`${route.meta.storeName}/CLEAR_DATA`);
                    }
                }
                // 删除vuex中的tagsView数据中的当前路由
                this.$store.dispatch('tagsView/delData', route).then((list) => {
                    // 关闭tag时，往关闭tag数据(closeData)添加当前的路由数据(暂时添加的是路由的path值)
                    this.$store.commit('tagsView/ADD_CLOSE_DATA', route);
                    // 如果是关闭当前选中的tag时，判断跳转路由
                    if(this.isActive(route)) {
                        // const latestRoute = list.slice(-1)[0];
                        // if(latestRoute) {
                        //     this.$router.push(latestRoute);
                        // }else {
                        //     this.$router.push('/');
                        // }
                        // 因为vuex数据中可能存在hidden: true的路由信息，所有需要循环排除掉，从尾部开始
                        let latestRoute = null;
                        for(let i = list.length - 1; i >= 0; i--) {
                            if(!list[i].hidden) {
                                latestRoute = list[i];
                                break;
                            }
                        }
                        // const latestRoute = list.slice(-1)[0];
                        if(latestRoute) {
                            this.$router.push(latestRoute);
                        }else {
                            this.$router.push('/');
                        }
                    }
                })
            },
            // 刷新路由
            refreshRoute() {
                this.$store.commit('tagsView/CHANGE_REFRESH');
                let refreshIcon = this.$refs.refreshIcon[0];
                refreshIcon.classList.add('refresh_icon_anim');
                setTimeout(() => {
                    refreshIcon.classList.remove('refresh_icon_anim');
                }, 1000)
            },
            // 通过标签页外的宽度判断标签页的的宽度是否改变
            changeTagsWidth() {
                // 获取tags的父级box
                let box = this.$refs.tagsDragBox.$el;
                // 获取tags的父级box的宽度（18位滚动条宽度）
                let boxW = box.offsetWidth - 18;
                // 获取tags的宽度默认宽度
                let tagW = 130;
                // 判断当前父级box一行能放入几个tags数
                let tagNum = Math.floor(boxW / tagW);
                // 判断数量并添加class名
                if(this.tagsData.length > tagNum) {
                    box.className += ' average_tags';
                }else {
                    box.className = 'drag_box clear_fix';
                }
            }
        },
        components: {
            draggable,
            RightBar,
            ScrollBar
        },
        mounted() {
            this.addViewTags();
        }
    }
</script>

<style lang="scss">
    header.search_mode{
        .tags_view[data-common-tags-view2-box] .el-tag{
            pointer-events: none;
            overflow: hidden;
            white-space: nowrap;
            // padding: 1.5em 0;
            margin-left: -35px;
            opacity: 0;
        }
        .el-scrollbar__bar.is-horizontal{
            display: none;
        }
    }
</style>

<style lang="scss">
    .tags_view[data-common-tags-view2-box]{
        display: flex;
        font-size: 0;
        .tags_view_scrollbar{
            height: 34px;
            margin-bottom: -2px;
            flex: 1;
            .el-scrollbar__wrap{
                overflow-x: hidden;
            }
            .el-scrollbar__view:after{
                content: '';
                display: block;
                clear: both;
            }
        }
        .drag_box{
            // flex: 1;
            float: left;
            display: flex;
            white-space: nowrap;
            position: relative;
            // transition: transform .3s;
            .el-tag:last-child{
                margin-right: 0;
            }
        }
        // .average_tags{
        //     display: flex;
        //     .el-tag{
        //         width: auto;
        //         flex: 1;
        //     }
        // }
        .el-tag{
            display: flex;
            align-items: center;
            justify-content: center;
            height: 32px;
            margin-right: 4px;
            padding: 0;
            // background: rgba(231,237,245,1);
            background: rgba(136,152,172,0.08);
            border-radius: 4px 4px 0px 0px;
            border: 1px solid rgba(219,228,240,1);
            border-bottom: none;
            overflow: hidden;
            transition: all 250ms;
            &.active{
                // background: rgba(24,117,240,1);
                // border: 1px solid rgba(24,117,240,1);
                background: #F3F6FA;
                border: 1px solid #F3F6FA;
                .refresh_icon{
                    margin-left: 8px;
                    cursor: pointer;
                    svg{
                        display: block;
                    }
                }
                .refresh_icon_anim{
                    transition: transform 1s ease-in-out;   
                    transform: rotate(720deg);
                }
                > a{
                    // color: #fff;
                    color: #454D58;
                }
            }
            > a{
                flex: 1;
                padding: 0 10px;
                overflow: hidden;
                text-overflow: ellipsis;
                white-space: nowrap;
                font-size: 12px;
                // color: rgba(136,152,172,1);
                color: #8898AC;
            }
            .el-tag__close{
                position: static;
                margin-right: 8px;
                // background-color: #fff;
                background-color: rgba(255,255,255,0.45);
                border-radius: 2px;
                // color: rgba(136,152,172,1);
                color: #8898AC;
                // transition: width .3s, margin .3s;
            }
        }
    }
</style>