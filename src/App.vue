<style>
input[type='number'] {
    -moz-appearance: textfield;
}

input::-webkit-outer-spin-button,
input::-webkit-inner-spin-button {
    -webkit-appearance: none;
}

.b-tooltip {
	pointer-events: none;
}
</style>

<template>
	<div id="app">
		<b-navbar toggleable="md" variant="light" class="mb-4">
			<b-container>
				<b-navbar-nav>
					<b-nav-item to="Start" :active="$route.path === '/Start'">{{$t('start.title')}}</b-nav-item>
					<b-nav-item to="General" :active="$route.path === '/General'">{{$t('general.title')}}</b-nav-item>
					<b-nav-item to="Mapping" :active="$route.path === '/Mapping'">{{$t('mapping.title')}}</b-nav-item>
					<b-nav-item to="Motors" :active="$route.path === '/Motors'">{{$tc('motors.title',2)}}</b-nav-item>
					<b-nav-item to="Endstops" :active="$route.path === '/Endstops'">{{$tc('endstops.title', 2)}}</b-nav-item>
					<b-nav-item to="Heaters" :active="$route.path === '/Heaters'">{{$tc('heaters.title',2)}}</b-nav-item>
					<b-nav-item to="Fans" :active="$route.path === '/Fans'">{{$tc('fans.title',2)}}</b-nav-item>
					<b-nav-item to="Tools" :active="$route.path === '/Tools'">{{$tc('tools.title',2)}}</b-nav-item>
					<b-nav-item to="Compensation" :active="$route.path === '/Compensation'">{{$t('compensation.title')}}</b-nav-item>
					<b-nav-item to="Display" :active="$route.path === '/Display'" v-show="$route.path === '/Display' || board.supportsDisplay">{{$t('display.title')}}</b-nav-item>
					<b-nav-item to="Network" :active="$route.path === '/Network'" v-show="$route.path === '/Network' || template.standalone">{{$t('network.title')}}</b-nav-item>
					<b-nav-item to="Finish" :active="$route.path === '/Finish'">{{$t('finish.title')}}</b-nav-item>
					<ul class="navbar-right">
						<LocaleSwitcher />
					</ul>
				</b-navbar-nav>
			</b-container>
		</b-navbar>
		
		<b-form ref="mainForm">
			<keep-alive>
				<router-view></router-view>
			</keep-alive>
		</b-form>

		<b-container class="mt-3">
			<ul class="pagination float-left">
				<li class="page-item" :class="{ disabled : isFirstPage }">
					<b-link class="page-link" href="#" @click.prevent="goToPreviousPage">
						« {{$t('app.back')}}
					</b-link>
				</li>
			</ul>

			<ul class="pagination float-right">
				<li class="page-item">
					<b-link class="page-link" href="#" @click.prevent="goToNextPage">
						{{ isLastPage ? $t('finish.title') : $t('app.next') }} »
					</b-link>
				</li>
			</ul>
		</b-container>

		<b-modal ref="errorModal" :ok-only="true" :title="$t('app.invalid')" @hidden="highlightErrors">
			<h4>{{$t('app.pageError')}}</h4>
			<h5>{{$t('app.pageErrorFix')}}</h5>
		</b-modal>

		<b-modal ref="ioErrorModal" :ok-only="true" :title="$t('app.invalidIO')">
			<h4>{{$t('app.configError')}}</h4>
			<h5>{{$t('app.configErrorFix')}}</h5>
		</b-modal>

		<b-modal ref="finishModal" size="lg" :title="$t('app.configReady')" @show="finishShow">
			<template v-if="dwcLink || iapLink || rrfLink || wifiLink">
				<p>{{$t('app.configInstructions')}}</p>
				<b-card bg-variant="light" class="mb-3">
					<ul class="pl-4 mb-0">
						<li v-show="iapLink">
							<a :href="iapLink" target="_blank">
								RepRapFirmware IAP utility for firmware updates
							</a>
						</li>
						<li v-show="rrfLink">
							<a :href="rrfLink" target="_blank">
								RepRapFirmware {{ rrfVersion }}
							</a>
						</li>
						<li v-if="this.template.network.enabled" v-show="wifiLink">
							<a :href="wifiLink" target="_blank">
								ESP8266 Firmware {{ wifiVersion }}
							</a>
						</li>
						<li v-if="this.template.network.enabled32" v-show="wifiLink">
							<a :href="wifiLink" target="_blank">
								ESP32 Firmware {{ wifiVersion }}
							</a>
						</li>
						<li v-show="dwcLink">
							<a :href="dwcLink" target="_blank">
								Duet Web Control {{ dwcVersion }}
							</a>
						</li>
						</li>
					</ul>
				</b-card>
			</template>

			<p>{{$t('app.individual')}}</p>
			<b-card ref="finishFiles" bg-variant="light" class="mb-3">
				<span v-if="files.length == 0" v-html="message"></span>
				<ul v-else class="pl-4 mb-0">
					<li v-for="filename in files">
						<b-link @click="showFile(filename)" v-text="filename"></b-link>
					</li>
				</ul>
			</b-card>

			<p>{{$t('app.dwc')}}<u>{{$t('app.dwc1')}}</u>{{$t('app.dwc2')}}</p>
			<p>{{$t('app.dwc3')}}<a href="https://duet3d.com/wiki/SD_card_folder_structure" target="_blank">{{$t('app.dwc4')}}</a>{{$t('app.dwc5')}}</p>

			<template slot="modal-footer">
				<a :href="configLink" class="btn btn-secondary" download="config.json">
					<font-awesome-icon icon="save"></font-awesome-icon> {{$t('app.json')}}
				</a>
				<b-button variant="primary" @click="generateZIP" :disabled="generatingZIP || files.length == 0">
					<font-awesome-icon :icon="generatingZIP ? 'hourglass' : 'download'"></font-awesome-icon> {{ generatingZIP ? $t('app.generating') : $t('app.zip') }}
				</b-button>
			</template>
		</b-modal>
	</div>
</template>

<script>
'use strict';

import { mapState, mapMutations } from 'vuex'
import { mapFields } from 'vuex-map-fields'
import saveAs from 'file-saver'

import Compiler from './Compiler.js'
import { goToNextPage, goToPreviousPage } from './Router.js'
import Template from './store/Template.js'

import LocaleSwitcher from "@/components/LocaleSwitcher"

export default {
	components: { LocaleSwitcher },
	computed: {
		...mapState(['board', 'template', 'addDWC', 'addRRF', 'addWiFi', 'requiresBeta']),
		isFirstPage() { return this.$route.path === '/Start'; },
		isLastPage() { return this.$route.path === '/Finish'; }
	},
	data() {
		return {
			configLink: '#',
			files: [],
			generatingZIP: false,
			message: 'Loading...',
			rrfVersion: null,
			rrfLink: null,
			rrfFile: null,
			wifiVersion: null,
			wifiLink: null,
			wifiFile: null,
			iapLink: null,
			iapFile: null,
			dwcVersion: null,
			dwcLink: null,
			dwcFile: null
		}
	},
	methods: {
		...mapMutations(['setTemplate']),
		goToPreviousPage() {
			let pageIndex = this.$router.options.routes.findIndex(route => route.path === this.$route.path);
			if (pageIndex > 0) {
				let prevPage = this.$router.options.routes[--pageIndex];
				if (prevPage.path === '/Network' && !this.template.standalone) {
					prevPage = this.$router.options.routes[--pageIndex];
				}
				if (prevPage.path === '/Display' && !this.board.supports12864) {
					prevPage = this.$router.options.routes[--pageIndex];
				}
				this.$router.push(prevPage);
			}
		},
		goToNextPage() {
			let pageIndex = this.$router.options.routes.findIndex(route => route.path === this.$route.path);
			if (pageIndex + 2 < this.$router.options.routes.length) {
				let nextPage = this.$router.options.routes[++pageIndex];
				if (nextPage.path === '/Display' && !this.board.supports12864) {
					nextPage = this.$router.options.routes[++pageIndex];
				}
				if (nextPage.path === '/Network' && !this.template.standalone) {
					nextPage = this.$router.options.routes[++pageIndex];
				}
				this.$router.push(nextPage);
			} else if (this.$refs.mainForm && this.$refs.mainForm.checkValidity() && this.$refs.mainForm.querySelector('.is-invalid') === null) {
				const templateCopy = JSON.parse(JSON.stringify(this.$store.state.template));
				if (Template.validatePins(templateCopy)) {
					this.$refs.finishModal.show();
				} else {
					this.$refs.ioErrorModal.show();
				}
			} else {
				this.$refs.errorModal.show();
			}
		},
		highlightErrors() {
			this.$refs.mainForm.reportValidity();
		},

		async finishShow() {
			this.rrfVersion = this.rrfLink = this.rrfFile = null;
			this.iapLink = this.iapFile = null;
			this.dwcVersion = this.dwcLink = this.dwcFile = null;
			this.wifiVersion = this.wifiLink = this.wifiFile = null;

			this.message = 'Loading...';
			this.files = [];
			this.configLink = 'data:text/plain;charset=utf-8,' + encodeURIComponent(JSON.stringify(this.template));

			// Load latest stable RRF version from GitHub
			if (this.addRRF && this.template.standalone && !this.template.requiresBeta) {
				try {
					// Get GitHub list of releases and assets. Do NOT get drafts and prereleases
					const releaseInfo = await Compiler.downloadFile('https://api.github.com/repos/gloomyandy/RepRapFirmware/releases', 'json');
					const firmware = this.template.firmware;
					let latestRelease = null;
					releaseInfo.forEach(function(item) {
						if (!item.draft && !item.prerelease && (!latestRelease || item.created_at > latestRelease.created_at)) {
							if ((firmware < 2 && item.name.indexOf('1.') !== -1) ||
								(firmware >= 2 && firmware < 3 && item.name.indexOf('2.') !== -1) ||
								(firmware >= 3 && item.name.indexOf('3.') !== -1)) {
								latestRelease = item;
							}
						}
					});

					// Attempt to download the required files (IAP+RRF)
					let latestReleaseNew = latestRelease.tag_name.substring(1);
					if (latestRelease) {
						// Try to download RRF from our own assets
						try {
							this.rrfFile = await Compiler.downloadFile(`assets/RepRapFirmware-${latestRelease.tag_name}/${this.board.firmwareStandaloneFile+'-'+latestReleaseNew+'.bin'}`, 'blob', 'application/octet-stream');
							this.rrfFile.name = this.board.firmwareStandaloneFile;
						} catch {
							this.rrfFile = null;
						}

						// TODO Add tool/expansion board files

						// Try to download IAP from our own assets
						const iapFile = (!this.template.board.startsWith('duet3') || this.template.standalone) ? this.board.iapFile : null;
						try {
							this.iapFile = await Compiler.downloadFile(`assets/RepRapFirmware-${latestRelease.tag_name}/${iapFile}`, 'blob', 'application/octet-stream');
							this.iapFile.name = iapFile;
						} catch {
							this.iapFile = null;
						}

						// Fall back to GitHub (although this probably won't work due to their odd CORS restrictions)
						for (let i = 0; i < latestRelease.assets.length; i++) {
							const item = latestRelease.assets[i];
							let rrfLink = null, iapLink = null;
							try {
								if (this.rrfFile === null && item.name === this.board.firmwareStandaloneFile+'-'+latestReleaseNew+'.bin') {
									rrfLink = item.browser_download_url;
									this.rrfVersion = latestRelease.tag_name;
									this.rrfFile = await Compiler.downloadFile(`assets/RepRapFirmware-${latestRelease.tag_name}/${item.name}`, 'blob', 'application/octet-stream');
									this.rrfFile.name = item.name;
								}
								else if (this.iapFile === null && item.name === iapFile) {
									iapLink = item.browser_download_url;
									this.iapFile = await Compiler.downloadFile(`assets/RepRapFirmware-${latestRelease.tag_name}/${item.name}`, 'blob', 'application/octet-stream');
									this.iapFile.name = item.name;
								}
							} catch (e) {
								if (rrfLink && this.rrfFile === null) {
									this.rrfLink = rrfLink;
								}
								if (iapLink && this.iapFile === null) {
									this.iapLink = iapLink;
								}
							}
						}
					} else {
						throw 'Could not find suitable RepRapFirmware version on GitHub';
					}
				} catch (e) {
					console.warn(`Failed to load RRF: ${e}`);
				}
			}

			// Load latest stable RRF version from GitHub
			if (this.addRRF && this.template.standalone && this.template.requiresBeta) {
				try {
					// Get GitHub list of releases and assets. Do NOT get drafts and prereleases
					const releaseInfo = await Compiler.downloadFile('https://api.github.com/repos/gloomyandy/RepRapFirmware/releases', 'json');
					const firmware = this.template.firmware;
					let latestRelease = null;
					releaseInfo.forEach(function(item) {
						if (!item.draft && item.prerelease && (!latestRelease || item.created_at > latestRelease.created_at)) {
							latestRelease = item;
						}
					});

					// Attempt to download the required files (IAP+RRF)
					let latestReleaseNew = latestRelease.tag_name.substring(1);
					let latestReleaseNew1 = latestReleaseNew.toLowerCase();
					if (latestRelease) {
						// Fall back to GitHub (although this probably won't work due to their odd CORS restrictions)
						for (let i = 0; i < latestRelease.assets.length; i++) {
							const item = latestRelease.assets[i];
							let rrfLink = null;
							try {
								if (item.name.indexOf('esp') !== -1) {
									rrfLink = item.browser_download_url;
									this.rrfVersion = latestRelease.tag_name;
									this.rrfFile = await Compiler.downloadFile(`assets/RepRapFirmware-${latestRelease.tag_name}/${(this.board.firmwareStandaloneFile+'-'+latestReleaseNew+'.bin')}`, 'blob', 'application/octet-stream');
									this.rrfFile.name = item.name;
								}
							} catch (e) {
								if (rrfLink && this.rrfFile === null) {
									this.rrfLink = rrfLink;
								}
							}
						}
					} else {
						throw 'Could not find suitable RepRapFirmware version on GitHub';
					}
				} catch (e) {
					console.warn(`Failed to load RRF: ${e}`);
				}
			}

			if (this.addRRF && !this.template.standalone && !this.template.requiresBeta) {
				try {
					// Get GitHub list of releases and assets. Do NOT get drafts and prereleases
					const releaseInfo = await Compiler.downloadFile('https://api.github.com/repos/gloomyandy/RepRapFirmware/releases', 'json');
					const firmware = this.template.firmware;
					let latestRelease = null;
					releaseInfo.forEach(function(item) {
						if (!item.draft && !item.prerelease && (!latestRelease || item.created_at > latestRelease.created_at)) {
							if ((firmware < 2 && item.name.indexOf('1.') !== -1) ||
								(firmware >= 2 && firmware < 3 && item.name.indexOf('2.') !== -1) ||
								(firmware >= 3 && item.name.indexOf('3.') !== -1)) {
								latestRelease = item;
							}
						}
					});

					// Attempt to download the required files (IAP+RRF)
					let latestReleaseNew = latestRelease.tag_name.substring(1);
					if (latestRelease) {
						// Try to download RRF from our own assets
						try {
							this.rrfFile = await Compiler.downloadFile(`assets/RepRapFirmware-${latestRelease.tag_name}/${this.board.firmwareStandaloneFile+'-'+latestReleaseNew+'.bin'}`, 'blob', 'application/octet-stream');
							this.rrfFile.name = this.board.firmwareStandaloneFile;
						} catch {
							this.rrfFile = null;
						}

						// TODO Add tool/expansion board files

						// Try to download IAP from our own assets
						const iapFile = (!this.template.board.startsWith('duet3') || this.template.standalone) ? this.board.iapFile : null;
						try {
							this.iapFile = await Compiler.downloadFile(`assets/RepRapFirmware-${latestRelease.tag_name}/${iapFile}`, 'blob', 'application/octet-stream');
							this.iapFile.name = iapFile;
						} catch {
							this.iapFile = null;
						}

						// Fall back to GitHub (although this probably won't work due to their odd CORS restrictions)
						for (let i = 0; i < latestRelease.assets.length; i++) {
							const item = latestRelease.assets[i];
							let rrfLink = null, iapLink = null;
							try {
								if (this.rrfFile === null && item.name === this.board.firmwareSBCFile+'-'+latestReleaseNew+'.bin') {
									rrfLink = item.browser_download_url;
									this.rrfVersion = latestRelease.tag_name;
									this.rrfFile = await Compiler.downloadFile(`assets/RepRapFirmware-${latestRelease.tag_name}/${item.name}`, 'blob', 'application/octet-stream');
									this.rrfFile.name = item.name;
								}
								else if (this.iapFile === null && item.name === iapFile) {
									iapLink = item.browser_download_url;
									this.iapFile = await Compiler.downloadFile(`assets/RepRapFirmware-${latestRelease.tag_name}/${item.name}`, 'blob', 'application/octet-stream');
									this.iapFile.name = item.name;
								}
							} catch (e) {
								if (rrfLink && this.rrfFile === null) {
									this.rrfLink = rrfLink;
								}
								if (iapLink && this.iapFile === null) {
									this.iapLink = iapLink;
								}
							}
						}
					} else {
						throw 'Could not find suitable RepRapFirmware version on GitHub';
					}
				} catch (e) {
					console.warn(`Failed to load RRF: ${e}`);
				}
			}

			// Load latest stable RRF version from GitHub
			if (this.addRRF && !this.template.standalone && this.template.requiresBeta) {
				try {
					// Get GitHub list of releases and assets. Do NOT get drafts and prereleases
					const releaseInfo = await Compiler.downloadFile('https://api.github.com/repos/gloomyandy/RepRapFirmware/releases', 'json');
					const firmware = this.template.firmware;
					let latestRelease = null;
					releaseInfo.forEach(function(item) {
						if (!item.draft && item.prerelease && (!latestRelease || item.created_at > latestRelease.created_at)) {
							latestRelease = item;
						}
					});

					// Attempt to download the required files (IAP+RRF)
					let latestReleaseNew = latestRelease.tag_name.substring(1);
					let latestReleaseNew1 = latestReleaseNew.toLowerCase();
					if (latestRelease) {
						// Fall back to GitHub (although this probably won't work due to their odd CORS restrictions)
						for (let i = 0; i < latestRelease.assets.length; i++) {
							const item = latestRelease.assets[i];
							let rrfLink = null;
							try {
								if (item.name.indexOf('sbc') !== -1) {
									rrfLink = item.browser_download_url;
									this.rrfVersion = latestRelease.tag_name;
									this.rrfFile = await Compiler.downloadFile(`assets/RepRapFirmware-${latestRelease.tag_name}/${(this.board.firmwareSBCFile+'-'+latestReleaseNew+'.bin')}`, 'blob', 'application/octet-stream');
									this.rrfFile.name = item.name;
								}
							} catch (e) {
								if (rrfLink && this.rrfFile === null) {
									this.rrfLink = rrfLink;
								}
							}
						}
					} else {
						throw 'Could not find suitable RepRapFirmware version on GitHub';
					}
				} catch (e) {
					console.warn(`Failed to load RRF: ${e}`);
				}
			}

			// Load ESP8266 stable from the server
			if (this.addWiFi && this.template.standalone && this.template.network.enabled && !this.template.requiresBeta) {
				try {
					// Get GitHub list of releases and assets. Do NOT get drafts and prereleases
					const releaseInfo = await Compiler.downloadFile('https://api.github.com/repos/gloomyandy/DuetWiFiSocketServer/releases', 'json');
					let latestRelease = null;
					releaseInfo.forEach(function(item) {
						if (!item.draft && !item.prerelease && (!latestRelease || item.created_at > latestRelease.created_at)) {
							latestRelease = item;
						}
					});

					// Attempt to download the required files (IAP+RRF)
					let latestReleaseNew = latestRelease.tag_name.substring(1);
					if (latestRelease) {
						for (let i = 0; i < latestRelease.assets.length; i++) {
							const item = latestRelease.assets[i];

							let wifiLink = null;
							try {
								if (item.name === this.board.firmwareWifiFile+'-'+latestReleaseNew+'.bin') {
									wifiLink = item.browser_download_url;
									this.wifiVersion = latestRelease.tag_name;
									this.wifiFile = await Compiler.downloadFile(`assets/DuetWiFiSocketServer-${latestRelease.tag_name}/${item.name}`, 'blob', 'application/octet-stream');
									break;
								}
							} catch (e) {
								this.wifiLink = wifiLink;
							}
						}
					}
				} catch (e) {
					console.warn(`Failed to load ESP: ${e}`);
				}
			}

			// Load ESP8266 unstable from the server
			if (this.addWiFi && this.template.standalone && this.template.network.enabled && this.template.requiresBeta) {
				try {
					// Get GitHub list of releases and assets. Do NOT get drafts and prereleases
					const releaseInfo = await Compiler.downloadFile('https://api.github.com/repos/gloomyandy/DuetWiFiSocketServer/releases', 'json');
					let latestRelease = null;
					releaseInfo.forEach(function(item) {
						if (!item.draft && item.prerelease && (!latestRelease || item.created_at > latestRelease.created_at)) {
							latestRelease = item;
						}
					});

					// Attempt to download the required files (IAP+RRF)
					let latestReleaseNew = latestRelease.tag_name.substring(1);
					if (latestRelease) {
						for (let i = 0; i < latestRelease.assets.length; i++) {
							const item = latestRelease.assets[i];

							let wifiLink = null;
							try {
								if (item.name === this.board.firmwareWifiFile+'-'+latestReleaseNew+'.bin') {
									wifiLink = item.browser_download_url;
									this.wifiVersion = latestRelease.tag_name;
									this.wifiFile = await Compiler.downloadFile(`assets/DuetWiFiSocketServer-${latestRelease.tag_name}/${item.name}`, 'blob', 'application/octet-stream');
									break;
								}
							} catch (e) {
								this.wifiLink = wifiLink;
							}
						}
					}
				} catch (e) {
					console.warn(`Failed to load ESP: ${e}`);
				}
			}

			// Load ESP32 stable from the server
			if (this.addWiFi && this.template.standalone && this.template.network.enabled32 && !this.template.requiresBeta) {
				try {
					// Get GitHub list of releases and assets. Do NOT get drafts and prereleases
					const releaseInfo = await Compiler.downloadFile('https://api.github.com/repos/gloomyandy/DuetWiFiSocketServer/releases', 'json');
					let latestRelease = null;
					releaseInfo.forEach(function(item) {
						if (!item.draft && !item.prerelease && (!latestRelease || item.created_at > latestRelease.created_at)) {
							latestRelease = item;
						}
					});

					// Attempt to download the required files (IAP+RRF)
					let latestReleaseNew = latestRelease.tag_name.substring(1);
					if (latestRelease) {
						for (let i = 0; i < latestRelease.assets.length; i++) {
							const item = latestRelease.assets[i];

							let wifiLink = null;
							try {
								if (item.name === this.board.firmwareWifi32File+'-'+latestReleaseNew+'.bin') {
									wifiLink = item.browser_download_url;
									this.wifiVersion = latestRelease.tag_name;
									this.wifiFile = await Compiler.downloadFile(`assets/DuetWiFiSocketServer-${latestRelease.tag_name}/${item.name}`, 'blob', 'application/octet-stream');
									break;
								}
							} catch (e) {
								this.wifiLink = wifiLink;
							}
						}
					}
				} catch (e) {
					console.warn(`Failed to load ESP: ${e}`);
				}
			}

			// Load ESP32 unstable from the server
			if (this.addWiFi && this.template.standalone && this.template.network.enabled32 && this.template.requiresBeta) {
				try {
					// Get GitHub list of releases and assets. Do NOT get drafts and prereleases
					const releaseInfo = await Compiler.downloadFile('https://api.github.com/repos/gloomyandy/DuetWiFiSocketServer/releases', 'json');
					let latestRelease = null;
					releaseInfo.forEach(function(item) {
						if (!item.draft && item.prerelease && (!latestRelease || item.created_at > latestRelease.created_at)) {
							latestRelease = item;
						}
					});

					// Attempt to download the required files (IAP+RRF)
					let latestReleaseNew = latestRelease.tag_name.substring(1);
					if (latestRelease) {
						for (let i = 0; i < latestRelease.assets.length; i++) {
							const item = latestRelease.assets[i];

							let wifiLink = null;
							try {
								if (item.name === this.board.firmwareWifi32File+'-'+latestReleaseNew+'.bin') {
									wifiLink = item.browser_download_url;
									this.wifiVersion = latestRelease.tag_name;
									this.wifiFile = await Compiler.downloadFile(`assets/DuetWiFiSocketServer-${latestRelease.tag_name}/${item.name}`, 'blob', 'application/octet-stream');
									break;
								}
							} catch (e) {
								this.wifiLink = wifiLink;
							}
						}
					}
				} catch (e) {
					console.warn(`Failed to load ESP: ${e}`);
				}
			}

			// Load DWC stable from the server
			if (this.addDWC && this.template.standalone && !this.template.requiresBeta) {
				try {
					// Get GitHub list of releases and assets. Do NOT get drafts and prereleases
					const releaseInfo = await Compiler.downloadFile('https://api.github.com/repos/chrishamm/DuetWebControl/releases', 'json');
					let latestRelease = null;
					releaseInfo.forEach(function(item) {
						if (!item.draft && !item.prerelease && (!latestRelease || item.created_at > latestRelease.created_at)) {
							latestRelease = item;
						}
					});

					// Attempt to download the required files (IAP+RRF)
					if (latestRelease) {
						for (let i = 0; i < latestRelease.assets.length; i++) {
							const item = latestRelease.assets[i];

							let dwcLink = null;
							try {
								if (item.name.indexOf('SD') !== -1) {
									dwcLink = item.browser_download_url;
									this.dwcVersion = latestRelease.tag_name;
									this.dwcFile = await Compiler.downloadFile(`assets/DuetWebControl-${latestRelease.tag_name}.zip`, 'blob', 'application/octet-stream');
									break;
								}
							} catch (e) {
								this.dwcLink = dwcLink;
							}
						}
					}
				} catch (e) {
					console.warn(`Failed to load DWC: ${e}`);
				}
			}

			// Load DWC unstable from the server
			if (this.addDWC && this.template.standalone && this.template.requiresBeta) {
				try {
					// Get GitHub list of releases and assets. Do NOT get drafts and prereleases
					const releaseInfo = await Compiler.downloadFile('https://api.github.com/repos/chrishamm/DuetWebControl/releases', 'json');
					let latestRelease = null;
					releaseInfo.forEach(function(item) {
						if (!item.draft && item.prerelease && (!latestRelease || item.created_at > latestRelease.created_at)) {
							latestRelease = item;
						}
					});

					// Attempt to download the required files (IAP+RRF)
					if (latestRelease) {
						for (let i = 0; i < latestRelease.assets.length; i++) {
							const item = latestRelease.assets[i];

							let dwcLink = null;
							try {
								if (item.name.indexOf('SD') !== -1) {
									dwcLink = item.browser_download_url;
									this.dwcVersion = latestRelease.tag_name;
									this.dwcFile = await Compiler.downloadFile(`assets/DuetWebControl-${latestRelease.tag_name}.zip`, 'blob', 'application/octet-stream');
									break;
								}
							} catch (e) {
								this.dwcLink = dwcLink;
							}
						}
					}
				} catch (e) {
					console.warn(`Failed to load DWC: ${e}`);
				}
			}

			// Generate config files
			try {
				const output = await Compiler.compileFile('templates/files.ejs', { template: this.template });
				this.files = output.trim().split('\n').map(file => file.trim());
			} catch (e) {
				this.message = 'Failed to load template from server:<br><br>' + e;
			}
		},
		async generateZIP() {
			this.generatingZIP = true;

			try {
				const zip = await Compiler.compileZIP(this.files, this.template, this.board, this.rrfFile, this.iapFile, this.dwcFile);
				if (Compiler.canDownloadFiles) {
					saveAs(zip, 'config.zip');
				} else {
					alert('Error: This browser does not support blobs! Save your configuration template and try another one.');
				}
			} catch (e) {
				alert('Failed to generate ZIP bundle:\n\n' + e);
				throw e;
			}

			this.generatingZIP = false;
		},
		async showFile(filename) {
			try {
				const output = await Compiler.compileTemplate(filename, this.template, this.board);

				let tab = window.open('about:blank', '_blank');
				if (tab == null) {
					alert('Could not open a new tab!\n\nPlease allow pop-ups for this page and try again.');
				} else {
					tab.document.write(output.replace(/\n/g, '<br>').replace(/ /g, '&nbsp;'));
					tab.document.body.style='font-family: monospace;'
					tab.document.title = filename;
					tab.document.close();
				}
			} catch (e) {
				alert(`Failed to generate file ${filename}:\n\n${e}`);
			}
		}
	},
	mounted() {
		const form = this.$refs.mainForm;
		const errorModal = this.$refs.errorModal;
		this.$router.beforeEach((to, from, next) => {
			if (from.path === '/Mapping' || (form.checkValidity() && form.querySelector('.is-invalid') === null)) {
				// Show error dialog for every page but IO Mapping - there is a discrete check for it anyway
				next();
			} else {
				errorModal.show();
				next(false);
			}
		});

		if (window.jsonTemplate) {
			this.setTemplate({ name: 'existing', data: window.jsonTemplate });
			this.$router.push('/General');
		}
	}
}
</script>
