<template>
    <div>
        <v-container id="vps-stats" v-if="!loading">
            <v-row>
                <v-col xs12 lg4>
                    <h4>{{$t('vps.control')}}</h4>
                    <v-btn v-if="off" success
                           v-bind:loading="changingStatus"
                           v-on:click.native="changingStatus = true"
                           v-bind:disabled="changingStatus"
                           class="white--text"
                    >
                        {{$t('vps.turn_on')}}
                    </v-btn>
                    <v-btn v-if="on" error
                           v-bind:loading="changingStatus"
                           v-on:click.native="changingStatus = true"
                           v-bind:disabled="changingStatus"
                           class="white--text"
                    >
                        {{$t('vps.turn_off')}}
                    </v-btn>
                    <div class="mb-2"></div>
                </v-col>
            </v-row>
            <v-row>
                <v-col xs12 lg4>
                    <h4>{{$t('vps.state')}}</h4>
                    <v-chip v-if="on" label class="green white--text">{{$t('vps.on')}}</v-chip>
                    <v-chip v-if="off" label class="red white--text">{{$t('vps.off')}}</v-chip>
                    <div class="mb-4"></div>
                </v-col>
            </v-row>
            <div v-if="on">
                <v-row>
                    <v-col xs12 lg4>
                        <h4>{{$t('vps.resources')}}</h4>
                    </v-col>
                </v-row>
                <v-row>
                    <v-col md6 xs12="xs12">
                        <h5>{{vps.cpu}}% {{$t('vps.cpu')}}</h5>
                        <div class="display-1"></div>
                        <progress-linear-color v-model="vps.cpu"></progress-linear-color>
                    </v-col>
                    <v-col md6 xs12="xs12">
                        <h5>{{ram}}% {{$t('vps.ram')}}</h5>
                        <progress-linear-color v-model="ram"></progress-linear-color>
                    </v-col>
                </v-row>
                <v-row>
                    <v-col md6 xs12="xs12">
                        <h5>{{$t('vps.network')}}</h5>
                        <v-chip label outline class="green green--text">
                            <i class="fa fa-fw fa-lg fa-cloud-download"></i>
                            {{vps.net_in_b | b_to_gb}}
                        </v-chip>
                        <v-chip label outline class="red red--text">
                            <i class="fa fa-fw fa-lg fa-cloud-upload"></i>
                            {{vps.net_out_b | b_to_gb}}
                        </v-chip>
                        <v-chip label outline class="green green--text">
                            <i class="fa fa-fw fa-lg fa-cloud-download"></i>
                            {{vps.net_in_bps | b_to_kb}}/s
                        </v-chip>
                        <v-chip label outline class="red red--text">
                            <i class="fa fa-fw fa-lg fa-cloud-upload"></i>
                            {{vps.net_out_bps | b_to_kb}}/s
                        </v-chip>
                    </v-col>
                </v-row>
                <div class="mb-4"></div>
                <v-row v-if="vps.virt == 'kvm'">
                    <v-col md6 xs12="xs12">
                        <h5>{{$t('vps.disk')}}</h5>
                        <v-chip label outline class="green green--text">
                            <i class="fa fa-fw fa-lg fa-upload"></i>
                            {{vps.disk_read_b | b_to_gb}}
                        </v-chip>
                        <v-chip label outline class="red red--text">
                            <i class="fa fa-fw fa-lg fa-download"></i>
                            {{vps.disk_write_b | b_to_gb}}
                        </v-chip>
                        <v-chip label outline class="green green--text">
                            <i class="fa fa-fw fa-lg fa-cloud-download"></i>
                            {{vps.disk_read_bps | b_to_kb}}/s
                        </v-chip>
                        <v-chip label outline class="red red--text">
                            <i class="fa fa-fw fa-lg fa-cloud-upload"></i>
                            {{vps.disk_write_bps | b_to_kb}}/s
                        </v-chip>
                    </v-col>
                </v-row>
            </div>
            <!--
             TODO
             - KVM & OpenVZ use vps.uptime_s - show and calculate for minutes, hours and days
             - OpenVZ use vps.nproc vps.max_swap_mb vps.swap_mb
             -->
        </v-container>
    </div>
</template>
<style>
    #vps-stats .progress-linear {
        position: relative
    }

    span.chip i {
        padding-right: 6px;
    }
</style>
<script>
    import moment from 'moment'
    export default {
        data () {
            return {
                interval: null,
                changingStatus: false
            }
        },
        mounted () {
            moment.locale(this.$lang)
            this.$store.commit('setToolbarTitle', 'header.vps_init')
            this.$emit('view', this.meta())
            this.$store.dispatch('checkSession').then((nosession) => {
                if (nosession) {
                    this.$vuetify.toast.create(this.$t('auth.no'), "right")
                    this.$router.push('/login')
                } else {
                    this.$store.commit('setLoading')
                    this.$store.dispatch('vpsInfo', {
                        'id': this.$route.params.id
                    }).then(() => {
                        this.interval = setInterval(this.stats, 1000)
                        this.$store.commit('setLoaded')
                        //FIXME set by API not user input
                        this.$store.commit('setToolbarTitle', 'header.vps')
                        this.$store.commit('setToolbarTitleArgs', {'id': this.$route.params.id})
                        this.$emit('view', this.meta())
                    })
                }
            })
        },
        destroyed () {
            clearInterval(this.interval)
        },
        computed: {
            on () {
                if (this.$store.state.vps.status == "running") {
                    return true;
                }
            },
            off () {
                if (this.$store.state.vps.status == "stopped") {
                    return true;
                }
            },
            loading () {
                return this.$store.state.loading
            },
            vps () {
                //FIXME may introduce strange behaviour, need to diff previous and current state to be good
                this.changingStatus = false
                return this.$store.state.vps
            },
            ram () {
                return Math.round((this.$store.state.vps.mem_mb / this.$store.state.vps.max_mem_mb) * 100)
            }
        },
        watch: {},
        filters: {
            prettyDate (unixtimestamp) {
                var timestamp = moment.unix(unixtimestamp);
                //DD.MM.YYYY or "L"
                return timestamp.format("H:mm L")
            },
            b_to_kb(b) {
                if (b < 0) {
                    return '... KB'
                }
                return Math.round(b / 1024) + ' KB'
            },
            b_to_gb(b) {
                if (b < 0) {
                    return '... GB'
                }
                return Math.round(b / 1024 / 1024 / 1024) + ' GB'
            }
        },
        preFetch (store) {
            store.commit('setMeta', this.methods.meta())
        },
        methods: {
            meta() {
                return {
                    title: 'VPS',
                    description: 'Example VPS description',
                    keywords: 'vuetify, vps'
                }
            },
            stats() {
                this.$store.dispatch('vpsInfo', {
                    'id': this.$route.params.id
                })
            }
        }
    }
</script>
