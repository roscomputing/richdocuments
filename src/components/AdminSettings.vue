<!--
  - @copyright Copyright (c) 2019 Julius Härtl <jus@bitgrid.net>
  -
  - @author Julius Härtl <jus@bitgrid.net>
  -
  - @license GNU AGPL version 3 or any later version
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
  -
  -->

<template>
	<div>
		<div class="section">
			<h2>Collabora Online</h2>
			<p>{{ t('richdocuments', 'Collabora Online is a powerful LibreOffice-based online office suite with collaborative editing, which supports all major documents, spreadsheet and presentation file formats and works together with all modern browsers.') }}</p>

			<div v-if="settings.wopi_url && settings.wopi_url !== ''">
				<div v-if="serverError == 2" id="security-warning-state-failure">
					<span class="icon icon-close-white" /><span class="message">{{ t('richdocuments', 'Could not establish connection to the Collabora Online server.') }}</span>
				</div>
				<div v-else-if="serverError == 1" id="security-warning-state-failure">
					<span class="icon icon-loading" /><span class="message">{{ t('richdocuments', 'Setting up a new server') }}</span>
				</div>
				<div v-else id="security-warning-state-ok">
					<span class="icon icon-checkmark-white" /><span class="message">{{ t('richdocuments', 'Collabora Online server is reachable.') }}</span>
				</div>
			</div>
			<div v-else id="security-warning-state-warning">
				<span class="icon icon-error-white" /><span class="message">{{ t('richdocuments', 'Please configure a Collabora Online server to start editing documents') }}</span>
			</div>

			<fieldset>
				<div>
					<input id="customserver" v-model="serverMode" type="radio"
						name="serverMode" value="custom" class="radio"
						:disabled="updating">
					<label for="customserver">{{ t('richdocuments', 'Use your own server') }}</label><br>
					<p class="option-inline">
						<em>{{ t('richdocuments', 'Collabora Online requires a seperate server acting as a WOPI-like Client to provide editing capabilities.') }}</em>
					</p>
					<div v-if="serverMode === 'custom'" class="option-inline">
						<form @submit="updateServer">
							<p>
								<label for="wopi_url">{{ t('richdocuments', 'URL (and Port) of Collabora Online-server') }}</label><br>
								<input id="wopi_url" v-model="settings.wopi_url" type="text"
									:disabled="updating">
								<input type="submit" value="Save" :disabled="updating"
									@click="updateServer"><br>
							</p>
							<p>
								<input id="disable_certificate_verification" v-model="settings.disable_certificate_verification" type="checkbox"
									class="checkbox" :disabled="updating" @change="updateServer">
								<label for="disable_certificate_verification">{{ t('richdocuments', 'Disable certificate verification (insecure)') }}</label><br>
								<em>{{ t('Enable if your Collabora Online server uses a self signed certificate') }}</em>
							</p>
						</form>
					</div>
				</div>
				<div>
					<input id="demoserver" v-model="serverMode" type="radio"
						name="serverMode" value="demo" class="radio"
						:disabled="updating" @input="fetchDemoServers">
					<label for="demoserver">{{ t('richdocuments', 'Use a demo server') }}</label><br>
					<p class="option-inline">
						<em>{{ t('richdocuments', 'You can use a demo server provided by Collabora and other service providers for giving Collabora Online a try.') }}</em>
					</p>
					<div v-if="serverMode === 'demo'" class="option-inline">
						<p v-if="demoServers === null">
							{{ t('richdocuments', 'Loading available demo servers …') }}
						</p>
						<p v-else-if="demoServers.length > 0">
							<multiselect v-if="serverMode === 'demo'" v-model="settings.demoUrl" :custom-label="demoServerLabel"
								track-by="demo_url" label="demo_url" placeholder="Select a demo server"
								:options="demoServers" :searchable="false" :allow-empty="false"
								:disabled="updating" @input="setDemoServer" />
						</p>
						<p v-else>
							{{ t('richdocuments', 'No available demo servers found.') }}
						</p>

						<p v-if="settings.demoUrl">
							<em>
								{{ t('richdocuments', 'Documents opened with the demo server configured will be sent to a 3rd party server. Only use this for evaluating Collabora Online.') }}<br>
								<a :href="settings.demoUrl.provider_url" target="_blank" rel="noreferrer noopener"
									class="external">{{ providerDescription }}</a>
							</em>
						</p>
					</div>
				</div>
			</fieldset>
		</div>

		<div v-if="isSetup" id="advanced-settings" class="section">
			<h2>{{ t('richdocuments', 'Advanced settings') }}</h2>
			<settings-checkbox :value="isOoxml" :label="t('richdocuments', 'Use Office Open XML (OOXML) instead of OpenDocument Format (ODF) by default for new files')" hint=""
				:disabled="updating" @input="updateOoxml" />

			<settings-checkbox :value="settings.use_groups !== null" :label="t('richdocuments', 'Restrict usage to specific groups')" :hint="t('richdocuments', 'Collabora Online is enabled for all users by default. When this setting is active, only members of the specified groups can use it.')"
				:disabled="updating" @input="updateUseGroups">
				<settings-select-group v-if="settings.use_groups !== null" v-model="settings.use_groups" :label="t('richdocuments', 'Select groups')"
					class="option-inline" :disabled="updating" @input="updateUseGroups" />
			</settings-checkbox>

			<settings-checkbox :value="settings.edit_groups !== null" :label="t('richdocuments', 'Restrict edit to specific groups')" hint="All users can edit documents with Collabora Online by default. When this setting is active, only the members of the specified groups can edit and the others can only view documents.')"
				:disabled="updating" @input="updateEditGroups">
				<settings-select-group v-if="settings.edit_groups !== null" v-model="settings.edit_groups" :label="t('richdocuments', 'Select groups')"
					class="option-inline" :disabled="updating" @input="updateEditGroups" />
			</settings-checkbox>

			<settings-checkbox v-model="uiVisible.canonical_webroot" :label="t('richdocuments', 'Use Canonical webroot')" hint=""
				:disabled="updating" @input="updateCanonicalWebroot">
				<settings-input-text v-if="uiVisible.canonical_webroot" v-model="settings.canonical_webroot" label=""
					:hint="t('richdocuments', 'Canonical webroot, in case there are multiple, for Collabora to use. Provide the one with least restrictions. Eg: Use non-shibbolized webroot if this instance is accessed by both shibbolized and non-shibbolized webroots. You can ignore this setting if only one webroot is used to access this instance.')"
					:disabled="updating" class="option-inline" @update="updateCanonicalWebroot" />
			</settings-checkbox>

			<settings-checkbox v-model="uiVisible.external_apps" :label="t('richdocuments', 'Enable access for external apps')" hint=""
				:disabled="updating" @input="updateExternalApps">
				<div v-if="uiVisible.external_apps">
					<settings-external-apps class="option-inline" :external-apps="settings.external_apps" :disabled="updating"
						@input="updateExternalApps" />
				</div>
			</settings-checkbox>
		</div>

		<div v-if="isSetup" id="secure-view-settings" class="section">
			<h2>{{ t('richdocuments', 'Secure view settings') }}</h2>
			<p>{{ t('richdocuments', 'Secure view enables you to secure documents by embedding a watermark') }}</p>
			<settings-checkbox v-model="settings.watermark.enabled" :label="t('richdocuments', 'Enable watermarking')" hint=""
				:disabled="updating" @input="update" />
			<settings-input-text v-if="settings.watermark.enabled" v-model="settings.watermark.text" label="Watermark text"
				:hint="t('richdocuments', 'Supported placeholders: {userId}, {date}')"
				:disabled="updating" @update="update" />
			<div v-if="settings.watermark.enabled">
				<settings-checkbox v-model="settings.watermark.allTags" :label="t('richdocuments', 'Show watermark on tagged files')" :disabled="updating"
					@input="update" />
				<p v-if="settings.watermark.allTags" class="checkbox-details">
					<settings-select-tag v-model="settings.watermark.allTagsList" :label="t('richdocuments', 'Select tags to enforce watermarking')" @input="update" />
				</p>
				<settings-checkbox v-model="settings.watermark.allGroups" :label="t('richdocuments', 'Show watermark for users of groups')" :disabled="updating"
					@input="update" />
				<p v-if="settings.watermark.allGroups" class="checkbox-details">
					<settings-select-group v-model="settings.watermark.allGroupsList" :label="t('richdocuments', 'Select tags to enforce watermarking')" @input="update" />
				</p>
				<settings-checkbox v-model="settings.watermark.shareAll" :label="t('richdocuments', 'Show watermark for all shares')" hint=""
					:disabled="updating" @input="update" />
				<settings-checkbox v-if="!settings.watermark.shareAll" v-model="settings.watermark.shareRead" :label="t('richdocuments', 'Show watermark for read only shares')"
					hint=""
					:disabled="updating" @input="update" />

				<h3>Link shares</h3>
				<settings-checkbox v-model="settings.watermark.linkAll" :label="t('richdocuments', 'Show watermark for all link shares')" hint=""
					:disabled="updating" @input="update" />
				<settings-checkbox v-if="!settings.watermark.linkAll" v-model="settings.watermark.linkSecure" :label="t('richdocuments', 'Show watermark for download hidden shares')"
					hint=""
					:disabled="updating" @input="update" />
				<settings-checkbox v-if="!settings.watermark.linkAll" v-model="settings.watermark.linkRead" :label="t('richdocuments', 'Show watermark for read only link shares')"
					hint=""
					:disabled="updating" @input="update" />
				<settings-checkbox v-if="!settings.watermark.linkAll" v-model="settings.watermark.linkTags" :label="t('richdocuments', 'Show watermark on link shares with specific system tags')"
					:disabled="updating" @input="update" />
				<p v-if="!settings.watermark.linkAll && settings.watermark.linkTags" class="checkbox-details">
					<settings-select-tag v-model="settings.watermark.linkTagsList" :label="t('richdocuments', 'Select tags to enforce watermarking')" @input="update" />
				</p>
			</div>
		</div>
	</div>
</template>

<script>
import Vue from 'vue'
import { Multiselect } from 'nextcloud-vue'
import axios from 'nextcloud-axios'
import SettingsCheckbox from './SettingsCheckbox'
import SettingsInputText from './SettingsInputText'
import SettingsSelectTag from './SettingsSelectTag'
import SettingsSelectGroup from './SettingsSelectGroup'
import SettingsExternalApps from './SettingsExternalApps'
import { generateUrl } from 'nextcloud-router'

const SERVER_STATE_OK = 0
const SERVER_STATE_LOADING = 1
const SERVER_STATE_CONNECTION_ERROR = 2

export default {
	name: 'AdminSettings',
	components: {
		SettingsCheckbox,
		SettingsInputText,
		SettingsSelectTag,
		SettingsSelectGroup,
		Multiselect,
		SettingsExternalApps
	},
	props: {
		initial: {
			type: Object,
			required: true
		}
	},
	data() {
		return {
			serverMode: '',
			serverError: SERVER_STATE_OK,
			demoServers: null,
			updating: false,
			groups: [],
			tags: [],
			uiVisible: {
				canonical_webroot: false,
				external_apps: false
			},
			settings: {
				demoUrl: null,
				wopi_url: null,
				watermark: {
					enabled: false,
					shareAll: false,
					shareRead: false,
					linkSecure: false,
					linkRead: false,
					linkAll: false,
					linkTags: false,
					linkTagsList: [],
					allGroups: false,
					allGroupsList: [],
					allTags: false,
					allTagsList: [],
					text: ''
				}
			}
		}
	},
	computed: {
		providerDescription() {
			return t('richdocuments', 'Contact {0} to get an own installation.', [this.settings.demoUrl.provider_name])
		},
		isSetup() {
			return this.serverError === SERVER_STATE_OK
		},
		isOoxml() {
			return this.settings.doc_format === 'ooxml'
		}
	},
	beforeMount() {
		for (let key in this.initial.settings) {
			if (!this.initial.settings.hasOwnProperty(key)) {
				continue
			}

			let [ parent, setting ] = key.split('_')
			if (parent === 'watermark') {
				Vue.set(this.settings[parent], setting, this.initial.settings[key])
			} else {
				Vue.set(this.settings, key, this.initial.settings[key])
			}

		}
		Vue.set(this.settings, 'data', this.initial.settings)
		if (this.settings.wopi_url === '') {
			this.serverError = SERVER_STATE_CONNECTION_ERROR
		}
		Vue.set(this.settings, 'edit_groups', this.settings.edit_groups ? this.settings.edit_groups.split('|') : null)
		Vue.set(this.settings, 'use_groups', this.settings.use_groups ? this.settings.use_groups.split('|') : null)

		this.uiVisible.canonical_webroot = !!(this.settings.canonical_webroot && this.settings.canonical_webroot !== '')
		this.uiVisible.external_apps = !!(this.settings.external_apps && this.settings.external_apps !== '')

		this.demoServers = this.initial.demo_servers
		this.checkIfDemoServerIsActive()
	},
	methods: {
		async fetchDemoServers() {
			try {
				const result = await axios.get(generateUrl('/apps/richdocuments/settings/demo'))
				this.demoServers = result.data
			} catch (e) {
				this.demoServers = []
			}
		},
		update() {
			this.updating = true
			let settings = this.settings
			axios.post(generateUrl('/apps/richdocuments/settings/watermark'), { settings }).then((response) => {
				this.updating = false
			}).catch((error) => {
				this.updating = false
				OC.Notification.showTemporary(t('richdocuments', 'Failed to save settings'))
				console.error(error)
			})
		},
		async updateUseGroups(enabled) {
			if (enabled) {
				this.settings.use_groups = enabled === true ? [] : enabled
			} else {
				this.settings.use_groups = null
			}
			await this.updateSettings({
				use_groups: this.settings.use_groups !== null ? this.settings.use_groups.join('|') : ''
			})
		},
		async updateEditGroups(enabled) {
			if (enabled) {
				this.settings.edit_groups = enabled === true ? [] : enabled
			} else {
				this.settings.edit_groups = null
			}
			await this.updateSettings({
				edit_groups: this.settings.edit_groups !== null ? this.settings.edit_groups.join('|') : ''
			})
		},
		async updateCanonicalWebroot(canonicalWebroot) {
			this.settings.canonical_webroot = (typeof canonicalWebroot === 'boolean') ? '' : canonicalWebroot
			if (canonicalWebroot === true) {
				return
			}
			await this.updateSettings({
				canonical_webroot: this.settings.canonical_webroot
			})
		},
		async updateExternalApps(externalApps) {
			this.settings.external_apps = (typeof externalApps === 'boolean') ? '' : externalApps
			if (externalApps === true) {
				return
			}
			await this.updateSettings({
				external_apps: this.settings.external_apps
			})
		},
		async updateOoxml(enabled) {
			this.settings.doc_format = enabled ? 'ooxml' : ''
			await this.updateSettings({
				doc_format: this.settings.doc_format
			})
		},
		async updateServer() {
			this.serverError = SERVER_STATE_LOADING
			try {
				await this.updateSettings({
					wopi_url: this.settings.wopi_url,
					disable_certificate_verification: this.settings.disable_certificate_verification
				})
				this.serverError = SERVER_STATE_OK
			} catch (e) {
				console.error(e)
				this.serverError = SERVER_STATE_CONNECTION_ERROR
			}
			this.checkIfDemoServerIsActive()
		},
		async updateSettings(data) {
			this.updating = true
			try {
				const result = await axios.post(
					OC.filePath('richdocuments', 'ajax', 'admin.php'),
					data
				)
				this.updating = false
				return result
			} catch (e) {
				this.updating = false
				throw e
			}
		},
		checkIfDemoServerIsActive() {
			this.settings.demoUrl = this.demoServers ? this.demoServers.find((server) => server.demo_url === this.settings.wopi_url) : null
			if (this.settings.wopi_url && this.settings.wopi_url !== '') {
				this.serverMode = 'custom'
			}
			if (this.settings.demoUrl) {
				this.serverMode = 'demo'
			}
		},
		demoServerLabel(server) {
			return `${server.provider_name} — ${server.provider_location}`
		},
		async setDemoServer(server) {
			this.settings.wopi_url = server.demo_url
			this.settings.disable_certificate_verification = false
			await this.updateServer()
		}
	}
}
</script>

<style lang="scss" scoped>
	p {
		margin-bottom: 15px;
	}
	p.checkbox-details {
		margin-left: 25px;
		margin-top: -10px;
		margin-bottom: 20px;
	}

	input[type='text'],
	.multiselect {
		width: 100%;
		max-width: 400px;
	}

	input#wopi_url {
		width: 300px;
	}

	#secure-view-settings {
		margin-top: 20px;
	}

	.section {
		border-bottom: 1px solid var(--color-border);
	}

	#security-warning-state-failure,
	#security-warning-state-warning,
	#security-warning-state-ok {
		margin-top: 10px;
		margin-bottom: 20px;
	}

	.option-inline {
		margin-left: 25px;
		&:not(.multiselect) {
			margin-top: 10px;
		}
	}
</style>
