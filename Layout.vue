<template>
	<div
		class="theme-container"
		:class="pageClasses"
		@touchstart="onTouchStart"
		@touchend="onTouchEnd"
	>
		<Navbar v-if="shouldShowNavbar" @toggle-sidebar="toggleSidebar" />
		<div class="sidebar-mask" @click="toggleSidebar(false)"></div>
		<Sidebar :items="sidebarItems" @toggle-sidebar="toggleSidebar">
			<slot slot="top" name="sidebar-top"></slot>
			<slot slot="bottom" name="sidebar-bottom"></slot>
		</Sidebar>
		<div v-if="$page.frontmatter.layout" class="custom-layout">
			<component :is="$page.frontmatter.layout" @hook:beforeUpdate="setPageTheme()" />
		</div>
		<Home v-else-if="$page.frontmatter.home" />
		<Page v-else :sidebar-items="sidebarItems">
			<slot slot="top" name="page-top"></slot>
			<slot slot="bottom" name="page-bottom"></slot>
		</Page>
		<SWUpdatePopup :update-event="swUpdateEvent" />
	</div>
</template>

<script>
import Vue from 'vue';
import nprogress from 'nprogress';
import Home from './layouts/Home.vue';
import Page from './layouts/Page.vue';
import Navbar from './components/nav/Navbar.vue';
import Sidebar from './components/sidebar/Sidebar.vue';
import SWUpdatePopup from './components/SWUpdatePopup.vue';
import { resolveSidebarItems } from './util.js';
import yuuConfig from './mixins/yuuConfig.js';
import themeHandler from './mixins/themeHandler.js';

export default {
	name: 'Layout',

	components: {
		Home,
		Page,
		Sidebar,
		Navbar,
		SWUpdatePopup,
	},

	mixins: [yuuConfig, themeHandler],

	data() {
		return {
			isSidebarOpen: false,
			swUpdateEvent: null,
		};
	},

	computed: {
		shouldShowNavbar() {
			const { themeConfig: theme } = this.$site;
			const { frontmatter } = this.$page;

			if (frontmatter.navbar === false || theme.navbar === false) {
				return false;
			}

			return this.$title || theme.logo || theme.repo || theme.nav || this.$themeLocaleConfig.nav;
		},

		shouldShowSidebar() {
			const { frontmatter } = this.$page;

			return !frontmatter.layout && !frontmatter.home && frontmatter.sidebar !== false && this.sidebarItems.length;
		},

		sidebarItems() {
			return resolveSidebarItems(this.$page, this.$route, this.$site, this.$localePath);
		},

		pageClasses() {
			const { pageClass } = this.$page.frontmatter;

			return [
				{
					'no-navbar': !this.shouldShowNavbar,
					'sidebar-open': this.isSidebarOpen,
					'no-sidebar': !this.shouldShowSidebar,
				},
				pageClass,
			];
		},
	},

	mounted() {
		window.addEventListener('scroll', this.onScroll);

		// configure progress bar
		nprogress.configure({ showSpinner: false });

		this.$router.beforeEach((to, from, next) => {
			if (to.path !== from.path && !Vue.component(to.name)) {
				nprogress.start();
			}

			next();
		});

		this.$router.afterEach(() => {
			nprogress.done();
			this.isSidebarOpen = false;
		});

		this.$on('sw-updated', this.onSWUpdated);
	},

	methods: {
		toggleSidebar(to) {
			this.isSidebarOpen = typeof to === 'boolean' ? to : !this.isSidebarOpen;
		},

		// side swipe
		onTouchStart(e) {
			this.touchStart = {
				x: e.changedTouches[0].clientX,
				y: e.changedTouches[0].clientY,
			};
		},

		onTouchEnd(e) {
			const dx = e.changedTouches[0].clientX - this.touchStart.x;
			const dy = e.changedTouches[0].clientY - this.touchStart.y;

			if (Math.abs(dx) > Math.abs(dy) && Math.abs(dx) > 40) {
				const openSidebar = dx > 0 && this.touchStart.x <= 80;
				this.toggleSidebar(openSidebar);
			}
		},

		onSWUpdated(e) {
			this.swUpdateEvent = e;
		},
	},
};
</script>

<style src="prismjs/themes/prism-tomorrow.css"></style>
<style src="./styles/theme.styl" lang="stylus"></style>
