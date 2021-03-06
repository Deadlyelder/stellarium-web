// Stellarium Web - Copyright (c) 2018 - Noctua Software Ltd
//
// This program is licensed under the terms of the GNU AGPL v3, or
// alternatively under a commercial licence.
//
// The terms of the AGPL v3 license can be found in the main directory of this
// repository.

<template>
<div style="height: 100%">
<v-tabs dark v-model="active" style="height: 0px;">
  <v-tab key="0"></v-tab>
  <v-tab key="1"></v-tab>
</v-tabs>
<v-tabs-items v-model="active">
  <v-tab-item key="0" style="height: 100%">
    <div style="height: 100%;">
      <v-toolbar dark dense>
        <v-spacer></v-spacer>
        <v-menu offset-y left>
          <v-btn icon slot="activator">
            <v-icon>account_circle</v-icon>
          </v-btn>
          <v-list subheader>
            <v-subheader>Logged In as {{ userName }}</v-subheader>
            <v-list-tile avatar @click="showProfilePage()">
              <v-list-tile-avatar>
                <v-icon>account_circle</v-icon>
              </v-list-tile-avatar>
              <v-list-tile-content>
                <v-list-tile-title>My Profile</v-list-tile-title>
              </v-list-tile-content>
            </v-list-tile>
            <v-list-tile avatar @click="logout()">
              <v-list-tile-avatar>
                <v-icon>exit_to_app</v-icon>
              </v-list-tile-avatar>
              <v-list-tile-content>
                <v-list-tile-title>Logout</v-list-tile-title>
              </v-list-tile-content>
            </v-list-tile>
          </v-list>
        </v-menu>
      </v-toolbar>
      <div class="scroll-container">
        <v-container fluid style="height: 100%">
          <v-layout row wrap>
            <v-flex xs12 v-for="obsg in observationGroups" :key="obsg.f1[0].objectId">
              <grouped-observations :obsGroupData="obsg" @thumbClicked="thumbClicked"></grouped-observations>
            </v-flex>
          </v-layout>
          <v-layout row wrap>
            <v-btn fab dark absolute color="pink" bottom right class="pink" slot="activator" style="bottom: 16px; margin-right: 10px" @click.stop.native="showObservationDetailsPage()">
              <v-icon dark>add</v-icon>
            </v-btn>
              <v-tooltip left>
                <span>Log a new Observation</span>
              </v-tooltip>
          </v-layout>
        </v-container>
      </div>
    </div>
  </v-tab-item>

  <v-tab-item key="1" style="height: 100%">
    <observation-details @back="backFromObservationDetails()" v-model="observationToAdd" :create="createObservation"></observation-details>
  </v-tab-item>
</v-tabs-items>
</div>
</template>

<script>
import GroupedObservations from './grouped-observations.vue'
import ObservationDetails from './observation-details.vue'
import { nsh } from '../ns_helpers.js'

export default {
  data: function () {
    return {
      active: null,
      observationToAdd: undefined,
      createObservation: false,
      savedViewSettings: []
    }
  },
  methods: {
    logout: function () {
      this.$store.dispatch('observing/signOut')
    },
    showObservationDetailsPage: function () {
      // Set this to undefined to tell the observation-detail component to use default values
      this.observationToAdd = undefined
      this.createObservation = true
      this.active = '1'
    },
    saveViewSettings: function () {
      this.savedViewSettings['utc'] = this.$store.state.stel.observer.utc
      this.savedViewSettings['loc'] = this.$store.state.currentLocation
      this.savedViewSettings['azalt'] = this.$store.state.stel.observer.azalt
      this.savedViewSettings['fov'] = this.$store.state.stel.fov
      this.savedViewSettings['lock'] = this.$stel.core.lock
    },
    restoreViewSettings: function () {
      if (!this.savedViewSettings) return
      if (this.savedViewSettings.utc) this.$stel.core.observer.utc = this.savedViewSettings.utc
      if (this.savedViewSettings.loc) {
        let loc = this.savedViewSettings.loc
        this.$stel.core.observer.latitude = loc.lat * Math.PI / 180.0
        this.$stel.core.observer.longitude = loc.lng * Math.PI / 180.0
        this.$stel.core.observer.elevation = loc.alt
        this.$store.commit('setCurrentLocation', loc)
      }
      if (this.savedViewSettings.lock) this.$stel.core.lock = this.savedViewSettings.lock
      if (this.savedViewSettings.azalt && this.savedViewSettings.fov) this.$stel.core.lookat(this.savedViewSettings.azalt, 1.0, this.savedViewSettings.fov)
    },
    backFromObservationDetails: function () {
      this.restoreViewSettings()
      this.active = '0'
    },
    thumbClicked: function (obs) {
      this.saveViewSettings()
      this.createObservation = false
      // Use assign to force triggering the reactive event
      this.observationToAdd = Object.assign({}, obs)
      this.active = '1'
    },
    showProfilePage: function () {
      console.log('showProfilePage')
    }
  },
  computed: {
    observationGroups: function () {
      var aggregator = [
        { $group: { objectId: { locationRef: '$locationRef', julian_day: { $floor: '$mjd' } }, groupDate: { min: { $min: '$mjd' }, max: { $max: '$mjd' } }, f0: { $sum: 1 }, f1: { $push: '$$ROOT' } } },
        { $sort: { 'groupDate.max': -1 } }
      ]
      return nsh.mingo.aggregate(this.$store.state.plugins.observing.noctuaSky.Observation, aggregator)
    },
    userName: function () {
      return this.$store.state.plugins.observing.noctuaSky.userName
    }
  },
  components: { GroupedObservations, ObservationDetails }
}
</script>

<style>
.scroll-container {
  height: calc(100% - 48px);
  overflow-y: auto;
  backface-visibility: hidden;
}
</style>
