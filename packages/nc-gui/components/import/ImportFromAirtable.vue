<template>
  <v-dialog v-model="airtableModal" max-width="min(600px, 90%)">
    <v-card class="nc-import-card h-100">
      <v-toolbar class="elevation-0 align-center" height="68">
        <h3 class="mt-2">
          {{ $t('title.importFromAirtable') }}
        </h3>
        <div v-t="['c:airtable-import:turbo-mode']" class="ml-2 mt-3 title pointer nc-btn-enable-turbo" @click="enableTurbo">
          🚀
        </div>
        <v-spacer />
      </v-toolbar>

      <v-divider />
      <div class="h-100" style="width: 100%">
        <div>
          <v-card v-if="step === 1" class="py-6 elevation-0" height="500">
            <div class="d-flex flex-column justify-center align-center pt-2 pb-6">
              <span class="subtitle-1 font-weight-medium" @dblclick="$set(syncSource.details,'syncViews',true)">
                Credentials
              </span>

              <a href="https://docs.nocodb.com/setup-and-usages/import-airtable-to-sql-database-within-a-minute-for-free" class="caption grey--text" target="_blank">Where to find this?</a>
            </div>

            <v-form v-model="valid">
              <div v-if="syncSource" class="px-10 mt-1 mx-auto" style="max-width: 400px">
                <v-text-field
                  v-model="syncSource.details.apiKey"
                  outlined
                  dense
                  label="Api Key"
                  class="caption nc-input-api-key"
                  :type="isPasswordVisible ? 'text':'password'"
                  :rules="[v=> !!v || 'Api Key is required']"
                >
                  <template #append="">
                    <v-icon class="mt-1" small @click="isPasswordVisible = !isPasswordVisible">
                      {{ isPasswordVisible ? 'visibility_off' : 'visibility' }}
                    </v-icon>
                  </template>
                </v-text-field>
                <v-text-field
                  v-model="syncSourceUrlOrId"
                  outlined
                  dense
                  label="Shared Base ID / URL"
                  class="caption nc-input-shared-base"
                  :rules="[(v) => !!v || 'Shared Base ID / URL is required']"
                />
              </div>
            </v-form>   <v-card-actions class="justify-center pb-6">
              <v-btn
                v-t="['c:sync-airtable:save-and-sync']"
                class="nc-btn-airtable-import"
                :disabled="!valid"
                large
                color="primary"
                @click="saveAndSync"
              >
                Import
              </v-btn>
            </v-card-actions>
          </v-card>

          <v-card
            v-if="step === 2"
            class="pb-4 mt-4 elevation-0"
          >
            <v-card-title class=" justify-center">
              <span class="subtitle-1 font-weight-medium">Logs</span>
            </v-card-title>

            <v-card
              ref="log"
              dark
              class="mt-2 mx-4 pa-4 elevation-0 green--text"
              height="500"
              style="overflow-y: auto"
            >
              <div v-for="({msg , status}, i) in progress" :key="i">
                <v-icon v-if="status==='FAILED'" color="red" size="15">
                  mdi-close-circle-outline
                </v-icon>
                <v-icon v-else color="green" size="15">
                  mdi-currency-usd
                </v-icon>
                <span class="caption nc-text">{{ msg }}</span>
              </div>
              <div
                v-if="!progress || !progress.length || progress[progress.length-1].status !== 'COMPLETED' && progress[progress.length-1].status !== 'FAILED'"
                class=""
              >
                <v-icon color="green" size="15">
                  mdi-loading mdi-spin
                </v-icon>
                <span class="caption nc-text">Syncing
                </span>
                <!--                  <div class="nc-progress" />-->
              </div>
            </v-card>

            <div
              v-if="progress && progress.length && progress[progress.length-1].status === 'COMPLETED'"
              class="pa-4 pt-8 text-center"
            >
              <v-btn large color="primary" class="nc-btn-go-dashboard" @click="airtableModal=false">
                Go to dashboard
              </v-btn>
            </div>
          </v-card>
        </div>
      </div>
    </v-card>
  </v-dialog>
</template>

<script>
import io from 'socket.io-client'

export default {
  name: 'ImportFromAirtable',
  props: {
    value: Boolean
  },
  data: () => ({
    isPasswordVisible: false,
    valid: false,
    socket: null,
    step: 1,
    progress: [],
    syncSource: null,
    syncSourceUrlOrId: ''
  }),
  computed: {
    airtableModal: {
      set(v) {
        this.$emit('input', v)
      },
      get() {
        return this.value
      }
    }
  },
  watch: {
    syncSourceUrlOrId(v) {
      if (this.syncSource && this.syncSource.details) {
        const m = v && v.match(/(exp|shr).{14}/g)
        this.syncSource.details.shareId = m ? m[0] : null
      }
    }
  },
  created() {
    this.socket = io(new URL(this.$axios.defaults.baseURL, window.location.href.split(/[?#]/)[0]).href, {
      extraHeaders: { 'xc-auth': this.$store.state.users.token }
    })
    this.socket.on('connect_error', () => {
      this.socket.disconnect()
      this.socket = null
    })

    const socket = this.socket
    socket.on('connect', function(data) {
      console.log(socket.id)
      console.log('socket connected', data)
    })

    socket.on('progress', (d) => {
      this.progress.push(d)

      this.$nextTick(() => {
        if (this.$refs.log) {
          const el = this.$refs.log.$el
          el.scrollTop = el.scrollHeight
        }
      })

      if (d.status === 'COMPLETED') {
        this.$store.dispatch('project/_loadTables', {
          dbKey: '0.projectJson.envs._noco.db.0',
          key: '0.projectJson.envs._noco.db.0.tables',
          _nodes: {
            dbAlias: 'db',
            env: '_noco',
            type: 'tableDir'
          }
        }).then(() => this.$store.dispatch('tabs/loadFirstTableTab'))
      }
    })
    this.loadSyncSrc()
  },
  beforeDestroy() {
    if (this.socket) {
      this.socket.disconnect()
    }
  },
  methods: {
    async saveAndSync() {
      await this.createOrUpdate()
      this.sync()
    },
    sync() {
      this.step = 2
      this.$axios.post(`/api/v1/db/meta/syncs/${this.syncSource.id}/trigger`, this.payload, {
        params: {
          id: this.socket.id
        }
      })
    },
    async loadSyncSrc() {
      const { data: { list: srcs } } = await this.$axios.get(`/api/v1/db/meta/projects/${this.projectId}/syncs`)
      if (srcs && srcs[0]) {
        srcs[0].details = srcs[0].details || {}
        this.syncSource = srcs[0]
        this.syncSourceUrlOrId = srcs[0].details.shareId
      } else {
        this.syncSource = {
          type: 'Airtable',
          details: {
            syncInterval: '15mins',
            syncDirection: 'Airtable to NocoDB',
            syncRetryCount: 1,

            syncViews: false,

            apiKey: '',
            shareId: ''
          }
        }
      }
    },
    async createOrUpdate() {
      try {
        const { id, ...payload } = this.syncSource
        if (id) {
          await this.$axios.patch(`/api/v1/db/meta/syncs/${id}`, payload)
        } else {
          this.syncSource = (await this.$axios.post(`/api/v1/db/meta/projects/${this.projectId}/syncs`, payload)).data
        }
      } catch (e) {
        this.$toast.error(await this._extractSdkResponseErrorMsg(e)).goAway(3000)
      }
    },
    enableTurbo() {
      this.$set(this.syncSource.details, 'syncViews', true)
      this.$toast.success('🚀🚀 Ludicrous mode activated! Let\'s go! 🚀🚀').goAway(3000)
    }
  }
}
</script>

<style scoped>

.nc-progress {
  margin-left: 12px;
  position: relative;
  width: 5px;
  height: 5px;
  border-radius: 5px;
  background-color: #9880ff;
  color: #9880ff;
  animation: dotFlashing 1s infinite linear alternate;
  animation-delay: .5s;
}

.nc-progress::before, .nc-progress::after {
  content: '';
  display: inline-block;
  position: absolute;
  top: 0;
}

.nc-progress::before {
  left: -7.5px;
  width: 5px;
  height: 5px;
  border-radius: 5px;
  background-color: #9880ff;
  color: #9880ff;
  animation: dotFlashing 1s infinite alternate;
  animation-delay: 0s;
}

.nc-progress::after {
  left: 7.5px;
  width: 5px;
  height: 5px;
  border-radius: 5px;
  background-color: var(--v-primary-base);
  color: var(--v-primary-base);
  animation: dotFlashing 1s infinite alternate;
  animation-delay: 1s;
}

@keyframes dotFlashing {
  0% {
    background-color: var(--v-primary-base);
  }
  50%,
  100% {
    background-color: var(--v-backgroundColor-base);
  }
}

</style>
