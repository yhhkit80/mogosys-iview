<style lang="less">
	@import '../main.less';
</style>

<template>
	<div ref="scrollCon" @DOMMouseScroll="handlescroll" @mousewheel="handlescroll" class="tags-outer-scroll-con">
		<!--修改 start-->
		<span class="tags-scroll-btn tags-left-btn" v-if="showLeftBtn" @click="prevTag"><Icon type="chevron-left"></Icon></span>
		<span class="tags-scroll-btn tags-right-btn" v-if="showRightBtn" @click="nextTag"><Icon type="chevron-right"></Icon></span>
		<!--修改 end-->
		<div class="close-all-tag-con">
			<Dropdown transfer @on-click="handleTagsOption">
				<Button size="small" type="primary">
                    标签选项
                    <Icon type="arrow-down-b"></Icon>
                </Button>
				<DropdownMenu slot="list">
					<DropdownItem name="clearAll">关闭所有</DropdownItem>
					<DropdownItem name="clearOthers">关闭其他</DropdownItem>
				</DropdownMenu>
			</Dropdown>
		</div>
		<div ref="scrollBody" class="tags-inner-scroll-body" :style="{left: tagBodyLeft + 'px'}">
			<transition-group name="taglist-moving-animation">
				<Tag type="dot" v-for="(item, index) in pageTagsList" ref="tagsPageOpened" :key="item.name" :name="item.name" @on-close="closePage" @click.native="linkTo(item)" :closable="item.name==='home_index'?false:true" :color="item.children?(item.children[0].name===currentPageName?'blue':'default'):(item.name===currentPageName?'blue':'default')">{{ itemTitle(item) }}</Tag>
			</transition-group>
		</div>
	</div>
</template>

<script>
	import Vue from 'vue';
	import VueI18n from 'vue-i18n';
	Vue.use(VueI18n);
	export default {
		name: 'tagsPageOpened',
		data() {
			return {
				currentPageName: this.$route.name,
				tagBodyLeft: 0,
				refsTag: [],
				tagsCount: 1,
				showLeftBtn: false,
				showRightBtn: false
			};
		},
		props: {
			pageTagsList: Array,
			beforePush: {
				type: Function,
				default: (item) => {
					return true;
				}
			}
		},
		computed: {
			title() {
				return this.$store.state.app.currentTitle;
			},
			tagsList() {
				return this.$store.state.app.pageOpenedList;
			},
		},
		methods: {
			itemTitle(item) {
				if(typeof item.title === 'object') {
					return this.$t(item.title.i18n);
				} else {
					return item.title;
				}
			},
			closePage(event, name) {
				let pageOpenedList = this.$store.state.app.pageOpenedList;
				let lastPageObj = pageOpenedList[0];
				if(this.currentPageName === name) {
					let len = pageOpenedList.length;
					for(let i = 1; i < len; i++) {
						if(pageOpenedList[i].name === name) {
							if(i < (len - 1)) {
								lastPageObj = pageOpenedList[i + 1];
							} else {
								lastPageObj = pageOpenedList[i - 1];
							}
							break;
						}
					}
				} else {
					let tagWidth = event.target.parentNode.offsetWidth;
					this.tagBodyLeft = Math.min(this.tagBodyLeft + tagWidth, 0);
				}
				this.$store.commit('removeTag', name);
				this.$store.commit('closePage', name);
				pageOpenedList = this.$store.state.app.pageOpenedList;
				localStorage.pageOpenedList = JSON.stringify(pageOpenedList);
				if(this.currentPageName === name) {
					this.linkTo(lastPageObj);
				}
			},
			linkTo(item) {
				let routerObj = {};
				routerObj.name = item.name;
				if(item.argu) {
					routerObj.params = item.argu;
				}
				if(item.query) {
					routerObj.query = item.query;
				}
				if(this.beforePush(item)) {
					this.$router.push(routerObj);
				}
			},
			handlescroll(e) {
				var type = e.type;
				let delta = 0;
				if(type === 'DOMMouseScroll' || type === 'mousewheel') {
					delta = (e.wheelDelta) ? e.wheelDelta : -(e.detail || 0) * 40;
				}
				let left = 0;
				if(delta > 0) {
					left = Math.min(0, this.tagBodyLeft + delta);
				} else {
					if(this.$refs.scrollCon.offsetWidth - 105 < this.$refs.scrollBody.offsetWidth) {
						if(this.tagBodyLeft < -(this.$refs.scrollBody.offsetWidth - this.$refs.scrollCon.offsetWidth + 105)) {
							left = this.tagBodyLeft;
						} else {
							left = Math.max(this.tagBodyLeft + delta, this.$refs.scrollCon.offsetWidth - this.$refs.scrollBody.offsetWidth - 107);
						}
					} else {
						this.tagBodyLeft = 0;
					}
				}
				this.tagBodyLeft = left;
			},
			handleTagsOption(type) {
				if(type === 'clearAll') {
					this.$store.commit('clearAllTags');
					this.$router.push({
						name: 'home_index'
					});
				} else {
					this.$store.commit('clearOtherTags', this);
				}
				this.tagBodyLeft = 0;
			},
			moveToView(tag) {
				if(tag.offsetLeft < -this.tagBodyLeft) {
					// 标签在可视区域左侧
					this.tagBodyLeft = -tag.offsetLeft + 15;
				} else if(tag.offsetLeft + 15 > -this.tagBodyLeft && tag.offsetLeft + tag.offsetWidth < -this.tagBodyLeft + this.$refs.scrollCon.offsetWidth - 120) {
					// 标签在可视区域
					//this.tagBodyLeft = Math.min(0, this.$refs.scrollCon.offsetWidth - 105 - tag.offsetWidth - tag.offsetLeft - 10);
				} else {
					// 标签在可视区域右侧
					this.tagBodyLeft = -(tag.offsetLeft - (this.$refs.scrollCon.offsetWidth - 100 - tag.offsetWidth) + 25);
				}
				this.resizeFun();
			},
			//修改
			resizeFun() {
				const that = this;
				setTimeout(() => {
					const boxWidth = that.$refs.scrollCon.offsetWidth,
						tagsWidth = that.$refs.scrollBody.offsetWidth;
					const overflow = boxWidth - 110 < tagsWidth;
					//console.log(boxWidth + '-' + tagsWidth, that.tagBodyLeft)
					that.showLeftBtn = overflow;
					that.showRightBtn = overflow;
				}, 100);
			},
			prevTag() {
				if(this.tagBodyLeft < 0)
					this.tagBodyLeft += 200;
				this.tagBodyLeft = this.tagBodyLeft >= 0 ? 0 : this.tagBodyLeft;
				this.resizeFun();
			},
			nextTag() {
				const that = this;
				const boxWidth = that.$refs.scrollCon.offsetWidth,
					tagsWidth = that.$refs.scrollBody.offsetWidth;
				const overflow = boxWidth - 110 < tagsWidth;
				var lastTag = this.refsTag[this.refsTag.length - 1].$el;
				const nleft = this.tagBodyLeft - 225;
				this.tagBodyLeft = Math.max(nleft, this.$refs.scrollCon.offsetWidth - 125 - lastTag.offsetWidth - lastTag.offsetLeft);
				this.resizeFun();
				//console.log(boxWidth + '-' + tagsWidth, this.refsTag)
			}
		},
		mounted() {
			this.refsTag = this.$refs.tagsPageOpened;
			setTimeout(() => {
				this.refsTag.forEach((item, index) => {
					if(this.$route.name === item.name) {
						let tag = this.refsTag[index].$el;
						this.moveToView(tag);
					}
				});
			}, 1); // 这里不设定时器就会有偏移bug
			this.tagsCount = this.tagsList.length;
			//修改
			const that = this;
			window.onresize = () => {
				return(() => {
					that.resizeFun();
				})();
			};
		},
		watch: {
			'$route' (to) {
				this.currentPageName = to.name;
				this.$nextTick(() => {
					this.refsTag.forEach((item, index) => {
						if(to.name === item.name) {
							let tag = this.refsTag[index].$el;
							this.moveToView(tag);
						}
					});
				});
				this.tagsCount = this.tagsList.length;
			},
			//修改
			'tagBodyLeft' (val) {
				this.resizeFun();
			}
		}
	};
</script>