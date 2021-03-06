<template>
  <v-app>
    <v-navigation-drawer
      v-model="drawer"
      :mini-variant="miniVariant"
      :clipped="clipped"
      fixed
      app
    >
      <nuxt-link to="/">
        <v-img
          v-if="miniVariant"
          :src="require('../static/cubezone2-square.png')"
          class="pa-2"
        />
        <v-img
          v-else
          class="ma-2"
          :src="require('../static/cubezone2-rect.png')"
        />
      </nuxt-link>
      <v-divider></v-divider>
      <v-list class="pt-0">
        <v-list-item
          v-for="(item, i) in items"
          :key="i"
          :to="item.to"
          router
          exact
        >
          <v-list-item-action>
            <v-icon>{{ item.icon }}</v-icon>
          </v-list-item-action>
          <v-list-item-content>
            <v-list-item-title v-text="item.title" />
          </v-list-item-content>
        </v-list-item>
      </v-list>
    </v-navigation-drawer>
    <v-app-bar :clipped-left="clipped" fixed app>
      <v-btn icon @click.stop="miniVariant = !miniVariant">
        <v-icon>mdi-{{ `chevron-${miniVariant ? 'right' : 'left'}` }}</v-icon>
      </v-btn>
      <v-spacer />
      <client-only>
        <v-menu
          v-if="user"
          :close-on-content-click="true"
          :max-width="300"
          offset-y
        >
          <template v-slot:activator="{ on }">
            <v-btn icon v-on="on">
              <v-avatar>
                <v-img v-if="user.avatar" :src="user.avatar" />
                <v-icon v-else>mdi-account</v-icon>
              </v-avatar>
            </v-btn>
          </template>

          <v-card>
            <v-list>
              <v-list-item>
                <v-list-item-avatar>
                  <v-img v-if="user.avatar" :src="user.avatar" />
                  <v-icon v-else>mdi-account</v-icon>
                </v-list-item-avatar>
                <v-list-item-content>
                  <v-list-item-title>{{ user.name }}</v-list-item-title>
                  <v-list-item-subtitle
                    >WCA ID: {{ user.wca_id }}</v-list-item-subtitle
                  >
                  <v-list-item-subtitle
                    >Nationality:
                    {{ countriesMap[user.nationality] }}</v-list-item-subtitle
                  >
                </v-list-item-content>
              </v-list-item>
            </v-list>

            <v-divider></v-divider>

            <v-list dense>
              <v-list-item :key="-1" :to="currentUserProfileRoute" exact nuxt>
                <v-list-item-content>
                  <v-list-item-title>My Profile Page</v-list-item-title>
                </v-list-item-content>
              </v-list-item>

              <v-list-item
                v-for="(item, i) in accountItems"
                :key="i"
                :to="item.to"
                :exact="item.exact"
                nuxt
              >
                <v-list-item-content>
                  <v-list-item-title>{{ item.title }}</v-list-item-title>
                </v-list-item-content>
              </v-list-item>
              <v-divider></v-divider>
              <v-list-item @click="logout()">
                <v-list-item-content>
                  <v-list-item-title>Logout</v-list-item-title>
                </v-list-item-content>
              </v-list-item>
            </v-list>
          </v-card>
        </v-menu>
        <div v-else>
          <v-btn nuxt to="/login">
            Login
          </v-btn>
        </div>
      </client-only>
    </v-app-bar>
    <v-content>
      <nuxt />
    </v-content>
    <v-navigation-drawer v-model="rightDrawer" :right="right" temporary fixed>
      <v-list>
        <v-list-item @click.native="right = !right">
          <v-list-item-action>
            <v-icon light>
              mdi-repeat
            </v-icon>
          </v-list-item-action>
          <v-list-item-title>Switch drawer (click me)</v-list-item-title>
        </v-list-item>
      </v-list>
    </v-navigation-drawer>
    <v-footer app>
      <a @click="copyIdTokenToClipboard()">{{ getBuildInfo() }}</a>
      <span>&nbsp;&copy; {{ new Date().getFullYear() }}</span>
      <nuxt-link to="/legal/privacy" class="pl-2">
        Privacy Policy
      </nuxt-link>
      <nuxt-link to="/legal/terms" class="pl-2">
        Terms and Conditions
      </nuxt-link>
      <v-spacer></v-spacer>
      <v-btn small text @click="toggleTheme()">Toggle Theme</v-btn>
      <a href="mailto:feedback@cube.zone" class="pl-2">
        feedback@cube.zone
      </a>
      <v-icon
        small
        class="pl-2"
        @click="goToLink('https://github.com/Kubiverse/Cube-Zone')"
        >mdi-github</v-icon
      >
      <v-icon
        small
        class="pl-2"
        @click="goToLink('https://www.facebook.com/CubeZone-115206463518382')"
        >mdi-facebook</v-icon
      >
      <a class="pl-2" @click="goToLink('https://discord.gg/J2v2eQN')">
        <v-icon small>mdi-discord</v-icon>
        Chat with us!
      </a>
    </v-footer>
    <v-snackbar v-model="snackbarOpen" :timeout="3000" :color="snackbarColor">
      {{ snackbarMessage }}
      <v-btn color="pink" text @click="snackbarOpen = false">Close</v-btn>
    </v-snackbar>
    <ViewCuberSolvesDialog
      :cuber="lookupCuber"
      :status="dialogs.viewCuberSolves"
      @close="dialogs.viewCuberSolves = false"
    ></ViewCuberSolvesDialog>
    <LoginDialog
      :status="dialogs.login"
      :redirect-route="loginRedirectRoute"
      @close="dialogs.login = false"
    ></LoginDialog>
  </v-app>
</template>

<script>
import { mapGetters } from 'vuex'
import sharedService from '~/services/shared.js'
import { countriesMap } from '~/services/constants.js'
import authService from '~/services/auth.js'
import { LOGOUT_MUTATION } from '~/gql/mutation/auth.js'
import { CUBER_NOTIFICATION_RECEIVED_SUBSCRIPTION } from '~/gql/subscription/cuber.js'
import ViewCuberSolvesDialog from '~/components/dialog/solve/viewCuberSolvesDialog'
import LoginDialog from '~/components/dialog/auth/loginDialog'

export default {
  components: {
    ViewCuberSolvesDialog,
    LoginDialog,
  },
  data() {
    return {
      countriesMap,
      dialogs: {
        viewCuberSolves: false,
        login: false,
      },
      lookupCuber: null,
      loginRedirectRoute: null,

      clipped: false,
      drawer: true,
      fixed: false,
      items: [
        {
          icon: 'mdi-home',
          title: 'Home',
          to: '/',
        },
        {
          icon: 'mdi-google-classroom',
          title: 'All Rooms',
          to: '/rooms',
        },
        {
          icon: 'mdi-card-account-details',
          title: 'My Recent Rooms',
          to: '/my-rooms',
        },
      ],

      accountItems: [{ title: 'Settings', to: '/settings', exact: false }],
      miniVariant: false,
      right: true,
      rightDrawer: false,

      //snackbars
      snackbarOpen: false,
      snackbarColor: '',
      snackbarMessage: '',
    }
  },

  computed: {
    ...mapGetters({
      user: 'auth/user',
    }),

    currentUserProfileRoute() {
      return '/cuber?id=' + this.user.id
    },
  },

  watch: {
    user(val) {
      if (val) {
        sharedService.generateSnackbar(this.$root, 'Login Success', 'success')

        this.subscribeToCuberNotifications()
      } else {
        sharedService.generateSnackbar(this.$root, 'User Logged Out', 'warning')
      }
    },
  },

  created() {
    this.$root.$on('snackbar-message', (val) => {
      if (val.message) {
        this.snackbarOpen = true
        this.snackbarMessage = val.message
        this.snackbarColor = val.color || ''
      }
    })

    this.$root.$on('cuber-lookup', (cuber) => {
      if (!cuber) return
      this.lookupCuber = cuber
      this.dialogs.viewCuberSolves = true
    })

    this.$root.$on('login-dialog', (route = null) => {
      this.loginRedirectRoute = route
      this.dialogs.login = true
    })

    if (this.user) {
      this.subscribeToCuberNotifications()
    }
  },

  mounted() {},

  methods: {
    goToLink: sharedService.goToLink,

    toggleTheme() {
      if (process.client) {
        this.$vuetify.theme.dark = !this.$vuetify.theme.dark
        localStorage.setItem(
          'theme',
          this.$vuetify.theme.dark ? 'dark' : 'light',
        )
      }
    },

    async logout() {
      try {
        await this.$apollo.mutate({
          mutation: LOGOUT_MUTATION,
          variables: {},
        })

        await authService.handleLogout(this)

        this.$router.push('/')
      } catch (err) {
        sharedService.handleError(err, this.$root)
      }
    },

    getBuildInfo() {
      return (
        'Build ' +
        (process.env.VER ? process.env.VER + ' - ' : '') +
        process.env.build_date
      )
    },

    copyIdTokenToClipboard() {
      sharedService.copyToClipboard(this, this.$apolloHelpers.getToken())
    },

    subscribeToCuberNotifications() {
      //do the subscription
      this.$apollo.addSmartSubscription('cuberNotificationReceived', {
        query: CUBER_NOTIFICATION_RECEIVED_SUBSCRIPTION,
        result({ _data }) {
          //console.log(data);
        },
      })
    },
  },
}
</script>
