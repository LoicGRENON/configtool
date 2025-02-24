<style>
.w-100-checkbox,
.w-100-checkbox > label {
	width: 100%;
}
</style>

<template>
	<b-container>
		<b-row>
			<b-col>
				<b-card :header="$t('display.direct')">
					<b-form-row>
						<b-col>
							<b-form-checkbox v-model="is12864" v-preset.left="preset.display.type === 1" :title="$t('display.12864Description')" class="mt-1" :disabled="!board.supports12864">{{$t('display.12864')}}</b-form-checkbox>
						</b-col>
						<b-col v-show="is12864">
							<b-form-group :label="$t('display.encoder')" class="mb-0">
								<b-select v-model.number="encoderSteps" v-preset="preset.display.encoder_steps" :title="$t('display.encoderDescription')">
									<option value="-4">-4</option>
									<option value="-2">-2</option>
									<option value="-1">-1</option>
									<option value="1">1</option>
									<option value="2">2</option>
									<option value="4">4</option>
								</b-select>
							</b-form-group>
						</b-col>
						<b-col v-show="is12864 && template.firmware >= 2">
							<b-form-group :label="$t('display.frequency')" class="mb-0">
								<b-input-group append="Hz">
									<b-form-input v-model.number="spiFrequency" type="number" step="1" required></b-form-input>
								</b-input-group>
							</b-form-group>
						</b-col>
					</b-form-row>
				</b-card>
			</b-col>

			<b-col cols="4" v-show="is12864">
				<b-card no-body>
					<template slot="header">
						<span>{{$t('display.preview')}}</span>

						<a href="#" @click.prevent="" id="visibility" class="float-right">
							<font-awesome-icon icon="eye"></font-awesome-icon> {{$t('display.visibility')}}
						</a>
						<b-popover target="visibility" placement="right" triggers="click blur">
							<b-form-radio-group v-model.number="machineState" name="machineState" class="w-100" buttons stacked>
								<b-form-radio value="0">{{$t('display.notPrinting')}}</b-form-radio>
								<b-form-radio value="1" button-variant="success">{{$t('display.printing')}}</b-form-radio>
								<b-form-radio value="2" button-variant="warning">{{$t('display.paused')}}</b-form-radio>
								<b-form-radio value="3" button-variant="warning">{{$t('display.resuming')}}</b-form-radio>
							</b-form-radio-group>

							<b-form-radio-group v-model="sdMounted" name="sdMounted" class="mt-2 w-100" buttons stacked block>
								<b-form-radio :value="true" button-variant="success">{{$t('display.sdMounted')}}</b-form-radio>
								<b-form-radio :value="false" button-variant="danger">{{$t('display.sdNotMounted')}}</b-form-radio>
							</b-form-radio-group>

							<b-form-checkbox v-model="toolHeaterFault" name="toolHeaterFault" button button-variant="danger" class="mt-2 w-100-checkbox">{{$t('display.toolFault')}}</b-form-checkbox>
							<b-form-checkbox v-model="bedFault" name="bedFault" button button-variant="danger" class="mt-1 w-100-checkbox">{{$t('display.bedFault')}}</b-form-checkbox>
						</b-popover>
					</template>

					<display-preview :value="fileContent" :current-line="currentLine" :machine-state="machineState" :sd-mounted="sdMounted" :toolHeater-fault="toolHeaterFault" :bed-fault="bedFault"></display-preview>
				</b-card>
			</b-col>
		</b-row>


		<b-row v-show="displayType !== 0" class="mt-3">
			<b-col cols="4">
				<b-card no-body>
					<template slot="header">
						<span class="mt-2">{{$t('display.menus')}}</span>

						<b-button size="sm" variant="success" class="float-right" @click="$refs.modalAddMenu.show()">
							<font-awesome-icon icon="plus"></font-awesome-icon> {{$t('display.add')}}
						</b-button>
					</template>

					<b-list-group>
						<b-list-group-item v-for="item in menus" :key="item.name" button :active="item === selectedMenu" class="d-flex justify-content-between align-items-center" @click="selectedMenu = item">
							{{ item.name }}
							<b-button v-if="item.name !== 'main'" size="sm" variant="danger" @click.stop="removeMenu(item.name)">
								<font-awesome-icon icon="trash"></font-awesome-icon>
							</b-button>
						</b-list-group-item>
					</b-list-group>
				</b-card>

				<b-card no-body class="mt-3">
					<template slot="header">
						<span class="mt-2">Images</span>

						<b-button size="sm" variant="success" class="float-right" @click="$refs.inputImage.click()">
							<font-awesome-icon icon="plus"></font-awesome-icon> {{$t('display.add')}}
						</b-button>
					</template>

					<b-list-group>
						<b-list-group-item v-for="item in images" :key="item.name" button class="d-flex justify-content-between align-items-center" @click="insertImage(item.name)">
							{{ item.name }}
							<b-button-group>
								<b-button size="sm" variant="secondary" @click.stop="invertImage(item)">
									<font-awesome-icon icon="dot-circle"></font-awesome-icon>
								</b-button>
								<b-button :href="getDataURL(item.value)" :download="item.name" size="sm" variant="primary" @click.stop="">
									<font-awesome-icon icon="download"></font-awesome-icon>
								</b-button>
								<b-button size="sm" variant="danger" @click.stop="removeDisplayImage(item.name)">
									<font-awesome-icon icon="trash"></font-awesome-icon>
								</b-button>
							</b-button-group>
						</b-list-group-item>
					</b-list-group>
				</b-card>
			</b-col>
			<b-col cols="8">
				<b-card no-body>
					<template slot="header">
						<span>{{$t('display.menuEditor')}}</span>
						<a href="https://duet3d.dozuki.com/Wiki/Duet_2_Maestro_12864_display_menu_system#Section_Menu_files" target="_blank" class="float-right">{{$t('display.supported')}}</a>
					</template> 

					<b-form-textarea v-model="fileContent" :disabled="!selectedMenu" id="editor" rows="4" max-rows="8" @mouseup="updateLineNumber" @keyup="updateLineNumber" @blur="currentLine = -1"></b-form-textarea>
				</b-card>
			</b-col>
		</b-row>

		<b-row>
			<b-col>
				<b-card :header="$t('display.serial')">
					<b-form-row>
						<b-col>
							<b-checkbox v-model="panelDue" :disabled="isTft || isFly" class="mb-3">{{$t('finish.paneldue')}}</b-checkbox>
							<b-checkbox v-model="tft" :disabled="isPanel || isFly" class="mb-3">{{$t('finish.tft')}}</b-checkbox>
							<b-checkbox v-model="flyscreen" :disabled="isPanel || isTft" class="mb-3">{{$t('finish.flymaker')}}</b-checkbox>
						</b-col>
						<b-col v-show="isPanel || isFly || isTft" cols="auto">
							<b-form-group :label="$t('display.auxRX')">
								<b-form-input v-model.trim="auxRX" v-preset="preset.auxRX" v-b-tooltip.hover :title="$t('display.auxRX')" maxlength="5" type="text"> </b-form-input>
							</b-form-group>
						</b-col>
						<b-col v-show="isPanel || isFly || isTft" cols="auto">
							<b-form-group :label="$t('display.auxTX')">
								<b-form-input v-model.trim="auxTX" v-preset="preset.auxTX" v-b-tooltip.hover :title="$t('display.auxTX')" maxlength="5" type="text"></b-form-input>
							</b-form-group>
						</b-col>
					</b-form-row>
					<b-form-row>
						<b-col v-show="board.serialAmount == 1 && (isPanel || isFly || isTft)" cols="auto">
							{{$t('display.warning')}}
						</b-col>
					</b-form-row>
					<b-form-row>
						<b-col v-show="isTft" cols="auto">
							{{$t('display.tftWarning')}}
						</b-col>
					</b-form-row>
				</b-card>
			</b-col>
		</b-row>

		<b-modal ref="modalAddMenu" title="Enter a Name" size="md" cancel-variant="danger" ok-variant="success" :ok-disabled="!canAddMenu" @ok="addMenu">
			<form ref="formAddMenu" @submit.prevent="submitAddMenu">
				<b-form-group :label="$t('display.menuName')">
					<b-form-input v-model="menuToAdd" type="text" required maxlength="250" autofocus></b-form-input>
				</b-form-group>
			</form>
		</b-modal>

		<input ref="inputImage" type="file" accept=".img,image/*" hidden @change="imageSelected"></input>
	</b-container>
</template>

<script>
'use strict';

import { mapState, mapMutations } from 'vuex'
import { mapFields, mapMultiRowFields } from 'vuex-map-fields'

import { base64ArrayBuffer } from '../mixins/base64ArrayBuffer.js'
import displayPreview from '../components/DisplayPreview.vue'

export default {
	components: {
		'display-preview': displayPreview
	},
	computed: {
		...mapState(['board', 'preset', 'template']),
		...mapFields({
			displayType: 'template.display.type',
			encoderSteps: 'template.display.encoder_steps',
			spiFrequency: 'template.display.spi_frequency',
			panelDue: 'template.panelDue',
			tft: 'template.tft',
			flyscreen: 'template.flyscreen',
			auxRX: 'board.auxRX',
			auxTX: 'board.auxTX'
		}),
		...mapMultiRowFields({
			menus: 'template.display.menus',
			images: 'template.display.images'
		}),
		is12864: {
			get() { return this.displayType === 1; },
			set(value) { this.displayType = value ? 1 : 0; }
		},
		fileContent: {
			get() { return this.selectedMenu ? this.selectedMenu.value : ''; },
			set(value) { this.selectedMenu.value = value; }
		},
		canAddMenu() {
			return this.menuToAdd.trim() !== '';
		},
		isPanel: function(){
			return this.panelDue;
		},
		isFly: function(){
			return this.flyscreen;
		},
		isTft: function(){
			return this.tft;
		}
	},
	data() {
		return {
			machineState: 0,
			sdMounted: true,
			toolHeaterFault: false,
			bedFault: false,

			currentLine: -1,

			selectedMenu: null,
			menuToAdd: ''
		}
	},
	mounted() {
		if (this.menus.length > 0) {
			this.selectedMenu = this.menus[0];
		}
	},
	methods: {
		...mapMutations(['setDisplayMenu', 'removeDisplayMenu', 'setDisplayImage', 'removeDisplayImage']),
		updateLineNumber(e) {
			let lineNumber = 0;
			for (let i = 0; i < e.target.selectionStart; i++) {
				if (e.target.value[i] === '\n') {
					lineNumber++;
				}
			}
			this.currentLine = lineNumber;
		},
		submitAddMenu() {
			this.$refs.modalAddMenu.hide();
			if (this.canAddMenu) {
				this.addMenu();
			}
		},
		addMenu() {
			this.setDisplayMenu({ name: this.menuToAdd, value: '' });
			for (let i = 0; i < this.menus.length; i++) {
				if (this.menus[i].name === this.menuToAdd) {
					this.selectedMenu = this.menus[i];
					break;
				}
			}
			this.menuToAdd = '';
		},
		removeMenu(name) {
			if (this.selectedMenu && this.selectedMenu.name == name) {
				this.selectedMenu = null;
			}
			this.removeDisplayMenu(name);
		},
		imageSelected(e) {
			const that = this;
			for (let i = 0; i < e.target.files.length; i++) {
				let name = e.target.files[i].name;
				const fileReader = new FileReader();
				if (name.toLowerCase().endsWith('.img')) {
					// Load .img files straight into the config
					fileReader.onload = function(evt) {
						const result = Array.from(new Uint8Array(evt.target.result));
						that.setDisplayImage({ name, value: result });
					}
					fileReader.readAsArrayBuffer(e.target.files[i]);
				} else {
					// Convert regular images to monochrome .img files
					const convertImageData = this.convertImageData;
					fileReader.onload = function(evt) {
						const img = new Image();
						img.src = evt.target.result;
						img.onload = function() {
							const canvas = document.createElement('canvas'), context = canvas.getContext('2d');
							canvas.width = img.width;
							canvas.height = img.height;
							context.drawImage(img, 0, 0);
							const imgData = context.getImageData(0, 0, img.width, img.height);
							const result = convertImageData(imgData.data, img.width, img.height);
							for (let i = name.length - 1; i > 0; i--) {
								if (name[i] === '.') {
									name = name.substr(0, i);
								}
							}
							that.setDisplayImage({ name: name + '.img', value: result });
							canvas.remove();
							img.remove();
						}
					}
					fileReader.readAsDataURL(e.target.files[i]);
				}
			}
		},
		convertImageData(imgData, cols, rows) {
			const bytesPerRow = Math.ceil(cols / 8);
			const result = new Array(bytesPerRow * rows + 2);
			result.fill(0);
			result[0] = cols;
			result[1] = rows;

			let i = 0;
			for (let y = 0; y < rows; y++) {
				for (let x = 0; x < cols; x++) {
					const grayPixel = ((0.3 * imgData[i * 4]) + (0.59 * imgData[i * 4 + 1]) + (0.11 * imgData[i * 4 + 2]));
					if (grayPixel < 128) {
						result[y * bytesPerRow + Math.trunc(x / 8) + 2] |= 128 >> (x % 8);
					}
					i++;
				}
			}

			return result;
		},
		insertImage(name) {
			if (this.selectedMenu) {
				if (this.fileContent.trim() !== '' && !this.fileContent.endsWith('\n') && !this.fileContent.endsWith('\r')) {
					this.fileContent += '\n';
				}
				this.fileContent += `image R4 C4 L"${name}"`;
			}
		},
		invertImage(item) {
			const invertedData = item.value.slice();
			for (let i = 2; i < item.value.length; i++) {
				invertedData[i] = ~item.value[i];
			}
			item.value = invertedData;
		},
		getDataURL(value) {
			return 'data:application/octet-stream;base64,' + encodeURIComponent(base64ArrayBuffer(value));
		}
	},
	watch: {
		menus() {
			if (this.selectedMenu) {
				if (this.menus.indexOf(this.selectedMenu) === -1) {
					this.selectedMenu = null;
				}
			} else {
				if (this.menus.length > 0) {
					this.selectedMenu = this.menus[0];
				}
			}
		}
	}
}
</script>

