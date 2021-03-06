<template>
  <v-container
    fluid
    fill-height
    :class="isChatboxOpened ? 'container-collapsed' : 'container-full'"
  >
    <v-layout align-center justify-center>
      <div v-if="errorMessage">
        <span class="display-1 pl-2">{{ errorMessage }} </span>
      </div>
      <div v-else-if="loading.joiningRoom">
        <span class="display-1 pl-2"
          >Joining Room...
          <v-progress-circular indeterminate></v-progress-circular>
        </span>
      </div>
      <div
        v-else-if="
          generation.room < 3 &&
          ($apollo.queries.room.loading || $apollo.queries.rounds.loading)
        "
      >
        <span class="display-1 pl-2"
          >Loading Room...
          <v-progress-circular indeterminate></v-progress-circular>
        </span>
      </div>
      <div v-else-if="!room">
        <span class="display-1 pl-2"
          >Initializing Room...
          <v-progress-circular indeterminate></v-progress-circular>
        </span>
      </div>
      <div v-else style="width: 100%;">
        <template>
          <v-row justify="center">
            <v-col cols="12" class="text-center">
              <div v-if="activeRound" class="text-center justify-center">
                <v-layout
                  v-if="settingsObject.showScramblePreview"
                  align-center
                  justify-center
                >
                  <ScrambleDisplay
                    :puzzle="
                      activeRound.scramble.event &&
                      activeRound.scramble.event.puzzle
                    "
                    :scramble="activeRound.scramble.scramble"
                    :solved="
                      !currentUserSolve ||
                      (currentUserSolve && currentUserSolve.state == 'FINISHED')
                    "
                    :visualization="settingsObject.scramblePreviewVisualization"
                  ></ScrambleDisplay>
                </v-layout>
                <div>
                  <div v-if="loading.updatingSolve">
                    <span class="display-1 pl-2"
                      >Submitting Results...
                      <v-progress-circular indeterminate></v-progress-circular>
                    </span>
                  </div>
                  <span
                    v-else-if="
                      !currentUserSolve ||
                      (currentUserSolve && currentUserSolve.state == 'FINISHED')
                    "
                    id="v-step-scramble"
                    class="display-1"
                    >Waiting for next round...</span
                  >
                  <div v-else :class="{ 'grey--text': chatBoxFocused }">
                    <Scramble
                      id="v-step-scramble"
                      :scramble="activeRound.scramble"
                      :size="settingsObject.scrambleFontSize"
                    />
                  </div>
                </div>
              </div>
            </v-col>
            <v-col xl="9" lg="12">
              <v-card outlined tile>
                <v-simple-table id="roomResults" fixed-header>
                  <template v-slot:default>
                    <thead>
                      <tr>
                        <th>Cuber</th>
                        <th class="text-right" width="150">
                          {{
                            activeRoundNumber
                              ? '#' + activeRoundNumber
                              : '&nbsp;'
                          }}
                        </th>

                        <th
                          v-for="i in tableRounds - 1"
                          :key="i"
                          class="text-right"
                          width="100"
                        >
                          <a @click="openViewRoundDialog(rounds[i])">{{
                            rounds[i] ? '#' + rounds[i].round_number : ''
                          }}</a>
                        </th>

                        <th
                          v-for="(accItem, i) in accumulatedResults"
                          :key="accItem.name"
                          width="100"
                          class="text-right"
                          :class="{ 'td-border-left': i == 0 }"
                        >
                          {{ accItem.name }}
                        </th>
                      </tr>
                    </thead>
                    <tbody>
                      <tr
                        v-for="item in cuberResults"
                        :key="item.id"
                        :class="{ 'item-highlight': item.updating }"
                      >
                        <td>
                          <PreviewCuberPopover :selected-item="item.cuber">
                          </PreviewCuberPopover>
                          <v-icon
                            v-if="item.id == room.creator.id"
                            small
                            color="yellow"
                            title="Room Creator"
                            >mdi-crown</v-icon
                          >
                          <v-icon
                            v-if="room.manager && room.manager.id == item.id"
                            color="green"
                            small
                            title="Room Manager"
                            >mdi-gavel</v-icon
                          >
                          <v-icon
                            v-else-if="isRoomManager"
                            small
                            title="Assign as manager"
                            @click="assignNewManager(item.id)"
                            >mdi-gavel</v-icon
                          >
                          <v-icon v-if="!item.isActive" small title="Away"
                            >mdi-sleep</v-icon
                          >
                        </td>
                        <td
                          v-for="i in tableRounds"
                          :key="i"
                          class="text-right title"
                          v-html="renderSolveResults(rounds[i - 1], item.id)"
                        ></td>

                        <td
                          v-for="(accItem, i) in accumulatedResults"
                          :key="accItem.name"
                          class="text-right title"
                          :class="{ 'td-border-left': i == 0 }"
                        >
                          <span
                            class="pointer-cursor"
                            @click="
                              openViewAccumulatedResultDialog(item.id, accItem)
                            "
                            v-html="renderAccumulatedResults(item.id, accItem)"
                          ></span>
                        </td>
                      </tr>
                    </tbody>
                  </template>
                </v-simple-table>
              </v-card>
            </v-col>
          </v-row>
        </template>
        <v-row justify="center">
          <span class="caption grey--text">
            Room Name: {{ room.name }} | Event: {{ room.event.name }} | Time
            Limit: {{ room.time_limit / 1000 || 'None' }} | Time Target:
            {{ room.time_target / 1000 || 'None' }} | Max Capacity:
            {{ room.max_capacity || 'None' }}
          </span>
        </v-row>
        <v-row justify="center">
          <v-col cols="12" class="text-center">
            <v-btn
              v-if="isRoomManager"
              id="v-step-startroom"
              :loading="loading.startingNextRound"
              @click="startNextRound()"
              >{{ activeRound ? 'Start Next Round' : 'Start Room' }}</v-btn
            >
            <span v-else id="v-step-startroom">&nbsp;</span>
            <v-btn nuxt to="/rooms">Leave Room</v-btn>
            <!--
            <v-btn @click="toggleIdleMode()" :loading="loading.idling">Spectator Mode</v-btn>
            -->
            <v-btn id="v-step-share" @click="copyShareLink()">Share Link</v-btn>
            <v-btn @click="openViewRoundsDialog()">All Rounds</v-btn>
            <v-btn @click="startTutorial()">Tutorial</v-btn>
            <v-btn
              id="v-step-settings"
              icon
              @click="openEditRoomSettingsDialog()"
            >
              <v-icon>mdi-cog</v-icon>
            </v-btn>
          </v-col>
        </v-row>
      </div>
    </v-layout>
    <v-bottom-navigation :value="true" height="auto" absolute>
      <v-expansion-panels v-model="expansionOpenedIndex">
        <v-expansion-panel>
          <v-expansion-panel-header
            expand-icon="mdi-chevron-up"
            class="title py-0"
          >
            <span>
              <v-icon left>mdi-chat</v-icon>
              Chat Room
              <span v-if="chatUnreadMessages">
                ({{ chatUnreadMessages }} New Messages)
              </span>
            </span>
          </v-expansion-panel-header>
          <v-expansion-panel-content>
            <ChatHistoryCard :chat-messages="chatMessages"></ChatHistoryCard>
            <div class="chat-box">
              <v-text-field
                ref="chatbox"
                v-model="chatMessage"
                name="chatbox"
                placeholder="Type your message here"
                dense
                filled
                hide-details
                solo-inverted
                append-icon="mdi-send"
                class="py-0"
                autocomplete="off"
                @keyup.enter="sendChatMessage()"
                @click:append="sendChatMessage()"
                @focus="chatBoxFocused = true"
                @blur="chatBoxFocused = false"
              ></v-text-field>
            </div>
          </v-expansion-panel-content>
        </v-expansion-panel>
      </v-expansion-panels>
    </v-bottom-navigation>
    <CubeTimerOverlay
      :disabled="cubeTimerDisabled"
      :input-method="settingsObject.inputMethod"
      :timer-trigger-duration="settingsObject.timerTriggerDuration"
      :inspection-timer="settingsObject.inspectionTimer"
      @submit-results="updateSolve"
      @timer-state-updated="handleTimerStateUpdate"
      @timer-status-updated="handleTimerStatusUpdate"
      @stopPropagation="stopPropagation = true"
    ></CubeTimerOverlay>
    <ViewRoundDialog
      :round-id="lookupRoundId"
      :status="dialogs.viewRound"
      @close="dialogs.viewRound = false"
    ></ViewRoundDialog>
    <ViewRoundsDialog
      :room="room"
      :status="dialogs.viewRounds"
      @close="dialogs.viewRounds = false"
    ></ViewRoundsDialog>
    <EditRoomSettingsDialog
      :status="dialogs.editRoomSettings"
      :settings-object="settingsObject"
      @update-settings="updateSettings"
      @close="dialogs.editRoomSettings = false"
    ></EditRoomSettingsDialog>
    <ViewAccumulatedResultDialog
      :selected-item="lookupAccumulated"
      :status="dialogs.viewAccumulated"
      @close="dialogs.viewAccumulated = false"
    ></ViewAccumulatedResultDialog>
    <v-tour name="myTour" :steps="steps"></v-tour>
  </v-container>
</template>

<script>
import sharedService from '~/services/shared.js'
import { ROOM_QUERY } from '~/gql/query/room.js'
import { ROUNDS_QUERY } from '~/gql/query/round.js'
import {
  JOIN_ROOM_MUTATION,
  LEAVE_ROOM_MUTATION,
  GRANT_ROOM_MANAGEMENT_TO_CUBER_MUTATION,
  NEXT_ROUND_MUTATION,
  SEND_ROOM_CHAT_MESSAGE_MUTATION,
  UPDATE_SOLVE_MUTATION,
} from '~/gql/mutation/room.js'
import { IDLE_MUTATION } from '~/gql/mutation/cuber.js'
import {
  ROOM_MEMBERS_UPDATED_SUBSCRIPTION,
  ROOM_UPDATED_SUBSCRIPTION,
  ROOM_CHAT_MESSAGE_RECEIVED_SUBSCRIPTION,
  ROUND_STARTED_SUBSCRIPTION,
  ROUND_FINISHED_SUBSCRIPTION,
  SOLVE_UPDATED_SUBSCRIPTION,
} from '~/gql/subscription/room.js'
import CubeTimerOverlay from '~/components/overlay/cubeTimerOverlay'
import PreviewCuberPopover from '~/components/popover/previewCuberPopover'
import ChatHistoryCard from '~/components/card/chatHistoryCard'
import ViewRoundDialog from '~/components/dialog/round/viewRoundDialog'
import ViewRoundsDialog from '~/components/dialog/round/viewRoundsDialog'
import EditRoomSettingsDialog from '~/components/dialog/misc/editRoomSettingsDialog'
import ViewAccumulatedResultDialog from '~/components/dialog/accumulated/viewAccumulatedResultDialog'
import ScrambleDisplay from '~/components/shared/scrambleDisplay.vue'
import Scramble from '~/components/shared/scramble'
import { mapGetters } from 'vuex'

export default {
  components: {
    CubeTimerOverlay,
    PreviewCuberPopover,
    ChatHistoryCard,
    ViewRoundDialog,
    ViewRoundsDialog,
    EditRoomSettingsDialog,
    ViewAccumulatedResultDialog,
    ScrambleDisplay,
    Scramble,
  },

  data() {
    return {
      room: null,
      rounds: [],
      activeRound: null,

      steps: [
        {
          target: '#v-step-startroom',
          header: {
            title: 'Get Started',
          },
          content: `If you are the room creator or manager, click <strong>Start Room</strong> to get started!<br/ />
          Otherwise, you will need to wait until the current round is finished before participating.`,
        },
        {
          target: '#v-step-scramble',
          content: `Scramble the cube according the scramble, which will be shown when the next round has started.<br />
          When the cube has been scrambled, <strong>press and hold space</strong> (or long tap on your mobile device) until the timer interface is <span class="green--text">green</span>.<br />
          Then release and begin solving!<br />
          Hit the space button (or tap the screen) when you are finished, then follow the instructions to finalize and record your time.`,
          params: {
            placement: 'top',
          },
        },
        {
          target: '#v-step-settings',
          header: {
            title: 'Room Settings',
          },
          content: `Click this button to view/edit your settings for this room, including the font size of the scramble and various timer settings.`,
        },
        {
          target: '#v-step-share',
          header: {
            title: 'Share Room',
          },
          content: `Click this button to copy the link to the room to your clipboard. You can then paste it and share with others who would like to join.`,
        },
      ],

      currentUserSolve: null,
      generation: {
        room: 0,
        rounds: 0,
      },

      settingsObject: {
        inputMethod: 'keyboard',
        inspectionTimer: false,
        timerTriggerDuration: 400,
        showScramblePreview: true,
        scrambleFontSize: 32,
        enableSounds: true,
        scramblePreviewVisualization: '2D',
      },

      accumulatedResults: [
        {
          name: 'avg5',
          type_id: '1',
          pivot_n: '5',
        },
        {
          name: 'avg12',
          type_id: '1',
          pivot_n: '12',
        },
      ],

      tableRounds: 6,

      solvesMap: {},

      cuberResults: [],

      loading: {
        joiningRoom: false,
        startingNextRound: false,
        updatingSolve: false,
        idling: false,
        assigningManager: false,
      },

      dialogs: {
        viewRound: false,
        viewRounds: false,
        viewAccumulated: false,
        editRoomSettings: false,
      },

      lookupRoundId: null,
      lookupAccumulated: null,

      errorMessage: null,

      chatMessages: [],
      chatMessage: null,
      chatBoxFocused: false,
      expansionOpenedIndex: undefined,
      chatUnreadMessages: 0,

      pageFocused: true,

      timerState: 0,
      timerStatus: false,
      stopPropagation: false,
    }
  },

  computed: {
    cubeTimerDisabled() {
      return (
        !(this.currentUserSolve && this.currentUserSolve.state != 'FINISHED') ||
        this.chatBoxFocused ||
        this.loading.updatingSolve ||
        this.dialogs.editRoomSettings
      )
    },

    isChatboxOpened() {
      return this.expansionOpenedIndex === 0
    },
    activeRoundNumber() {
      return this.activeRound ? this.activeRound.round_number : 0
    },
    isRoomManager() {
      if (!this.room) return false
      if (this.room.creator && this.room.creator.id == this.user.id) {
        return true
      }

      if (this.room.manager && this.room.manager.id == this.user.id) {
        return true
      }

      return false
    },
    ...mapGetters({
      user: 'auth/user',
    }),
  },

  watch: {
    isChatboxOpened(val) {
      if (val) {
        this.chatUnreadMessages = 0
      }
    },
    user(val) {
      if (val) {
        this.reset()
      }
    },
  },

  mounted() {
    //attempt to join room
    this.reset()

    if (process.client) {
      window.addEventListener('blur', this.onPageBlur)
      window.addEventListener('focus', this.onPageFocus)
      window.addEventListener('keyup', this.handleKeyUp)
    }
  },

  destroyed() {
    if (process.client) {
      window.removeEventListener('blur', this.onPageBlur)
      window.removeEventListener('focus', this.onPageFocus)
      window.removeEventListener('keyup', this.handleKeyUp)
    }
  },

  methods: {
    updateSettings(settingsObject) {
      Object.assign(this.settingsObject, settingsObject)

      //save settings to localStorage
      if (process.client) {
        localStorage.setItem(
          'roomSettings',
          JSON.stringify(this.settingsObject),
        )
      }

      sharedService.generateSnackbar(this.$root, 'Settings Saved', 'success')
    },

    //checks the rounds and deletes any historical rounds and references to solves
    triggerGarbageCollection() {
      while (this.rounds.length > this.tableRounds) {
        const round = this.rounds.pop()
        round.solves.forEach((solve) => {
          delete this.solvesMap[solve.id]
        })
      }
    },

    renderSolveResults(round, cuberId) {
      if (!round) return ''
      let foundSolve = round.solves.find((solve) => solve.cuber.id === cuberId)

      if (!foundSolve) return ''

      return (
        '<span' +
        (foundSolve.is_winner ? " class='winner-text'" : '') +
        '>' +
        sharedService.generateSolveString(
          foundSolve,
          round.round_number === this.activeRoundNumber,
        ) +
        '</span>'
      )
    },

    renderAccumulatedResults(cuberId, accItemObject) {
      if (!this.rounds[1] || !this.rounds[1].accumulated_results) return 'N/A'

      let foundResult = this.findAccumulatedResult(cuberId, accItemObject)

      if (!foundResult) return 'N/A'

      return (
        '<span' +
        (foundResult.is_winner ? " class='winner-text'" : '') +
        '>' +
        sharedService.generateAccumulatedResultString(foundResult) +
        '</span>'
      )
    },

    findAccumulatedResult(cuberId, accItemObject) {
      return this.rounds[1].accumulated_results.find(
        (accumulator) =>
          accumulator.contextualAccumulator.type_id === accItemObject.type_id &&
          accumulator.contextualAccumulator.pivot_n === accItemObject.pivot_n &&
          accumulator.cuber.id == cuberId,
      )
    },

    handleKeyUp(e) {
      if (this.stopPropagation) {
        this.stopPropagation = false
        return
      }
      if (this.timerStatus) return
      if (e.code == 'Enter') {
        if (!this.chatBoxFocused) {
          this.expansionOpenedIndex = 0
          setTimeout(() => {
            this.$refs.chatbox.focus()
          }, 250)
        }
      }
    },

    onPageBlur() {
      this.pageFocused = false
    },

    onPageFocus() {
      this.pageFocused = true
    },

    copyShareLink() {
      sharedService.copyToClipboard(this, window.location.href)
    },

    openViewRoundDialog(round) {
      this.lookupRoundId = round.id
      this.dialogs.viewRound = true
    },

    openEditRoomSettingsDialog() {
      this.dialogs.editRoomSettings = true
    },

    openViewAccumulatedResultDialog(cuberId, accItemObject) {
      let accumulated = this.findAccumulatedResult(cuberId, accItemObject)

      if (!accumulated) return

      this.lookupAccumulated = accumulated
      this.dialogs.viewAccumulated = true
    },

    openViewRoundsDialog() {
      this.dialogs.viewRounds = true
    },

    updateObject(oldObject, newObject) {
      for (let prop in oldObject) {
        if (prop !== 'id' && prop in newObject) {
          //if prop is_winner and the old value was true, skip updating it
          if (prop === 'is_winner' && oldObject[prop] === true) {
            continue
          }
          oldObject[prop] = newObject[prop]
        }
      }
    },

    updateSolvesMap(rounds) {
      rounds.forEach((round) => {
        round.solves.forEach((solve) => {
          this.solvesMap[solve.id] = solve
        })
      })
    },

    updateActiveRound(round) {
      this.activeRound = round || null
    },

    //adds an array of active cubers to the cuberResults, if not already in there
    addActiveCubers(activeCubers, initialLoad) {
      if (!Array.isArray(activeCubers)) return
      activeCubers.forEach((active_cuber) => {
        let foundResult = this.cuberResults.find(
          (cuber) => cuber.id == active_cuber.id,
        )

        if (!foundResult) {
          !initialLoad &&
            this.handleChatMessageReceived({
              user: active_cuber,
              message: 'has just joined the room!',
              system: true,
            })

          this.cuberResults.push({
            id: active_cuber.id,
            cuber: active_cuber,
            updating: false,
            isActive: true,
            recentActiveRound: null,
          })
        } else if (foundResult && !foundResult.isActive) {
          !initialLoad &&
            this.handleChatMessageReceived({
              user: active_cuber,
              message: 'has just joined the room!',
              system: true,
            })

          foundResult.isActive = true
        }
      })
    },

    removeActiveCubers(activeCubers, initialLoad) {
      if (!Array.isArray(activeCubers)) return

      this.cuberResults.forEach((cuber) => {
        let foundResult = activeCubers.find(
          (active_cuber) => active_cuber.id == cuber.id,
        )

        if (!foundResult) {
          if (cuber.isActive && !initialLoad) {
            this.handleChatMessageReceived({
              user: cuber.cuber,
              message: 'has just left the room!',
              system: true,
            })
          }

          //if result not found, remove if cuber didn't participate
          if (!cuber.recentActiveRound) {
            let index = this.cuberResults.indexOf(cuber)
            if (index !== -1) {
              this.cuberResults.splice(index, 1)
            }
          } else {
            cuber.isActive = false
          }
        }
      })
    },

    //compares a list of activeCubers with the cuberResults array, and makes adjustments
    updateActiveCubers(activeCubers, initialLoad = false) {
      this.addActiveCubers(activeCubers, initialLoad)
      this.removeActiveCubers(activeCubers, initialLoad)
    },

    handleChatMessageReceived(chatMessage) {
      this.chatMessages.push(chatMessage)
      if (!this.isChatboxOpened) {
        this.chatUnreadMessages++
        if (this.timerState == 0 && this.settingsObject.enableSounds) {
          sharedService.playSound('/alert2.mp3')
        }
      }
    },

    //builds the cuberResults array from all rounds provided.
    buildCuberResults() {
      this.rounds.forEach((round) => {
        round.solves.forEach((solve) => {
          const foundResult = this.cuberResults.find(
            (cuber) => cuber.id == solve.cuber.id,
          )

          if (!foundResult) {
            this.cuberResults.push({
              id: solve.cuber.id,
              cuber: solve.cuber,
              updating: false,
              isActive: true,
              recentActiveRound: round,
            })
          } else {
            foundResult.recentActiveRound = round
          }
        })
      })
    },

    flashResultUpdating(cuberResult) {
      if (!cuberResult) return
      cuberResult.updating = true
      setTimeout(() => {
        cuberResult.updating = false
      }, 1000)
    },

    //adds the cuber results from a specific new round
    addCuberResults(round) {
      //add any new cubers in the solves to cuberResults
      round.solves.forEach((solve) => {
        let foundResult = this.cuberResults.find(
          (cuber) => cuber.id == solve.cuber.id,
        )

        if (!foundResult) {
          this.cuberResults.push({
            id: solve.cuber.id,
            cuber: solve.cuber,
            updating: false,
            isActive: true,
            recentActiveRound: round,
          })
        } else {
          foundResult.recentActiveRound = round
        }
      })
    },

    //looks in the active round and finds the solve that belongs to the current user, if any.
    findCurrentUserSolve() {
      if (!this.activeRound || !this.$store.getters['auth/user']) {
        return
      }

      const currentUserId = this.$store.getters['auth/user'].id
      const solve = this.activeRound.solves.find(
        (ele) => ele.cuber.id == currentUserId,
      )

      this.currentUserSolve = solve || null
    },

    //checks the cuber results to see if anyone needs to be removed (if past 5 results are all null)
    checkCuberResults() {
      for (let i = this.cuberResults.length - 1; i >= 0; i--) {
        //detect if off the scoreboard
        if (
          !this.cuberResults[i].recentActiveRound ||
          this.cuberResults[i].recentActiveRound.round_number <=
            this.activeRoundNumber - this.tableRounds
        ) {
          this.cuberResults.splice(i, 1)
        }
      }
    },

    async joinAndLoadRoom() {
      this.loading.joiningRoom = true
      try {
        //must be logged in
        if (!this.$store.getters['auth/user']) {
          this.$root.$emit('login-dialog', this.$route.fullPath)
          throw sharedService.generateError('Login required to join room.')
        }

        await this.$apollo.mutate({
          mutation: JOIN_ROOM_MUTATION,
          variables: {
            room_id: this.$route.query.id,
            secret: this.$route.query.secret,
          },
        })
        this.$apollo.queries.room.skip = false

        this.$apollo.subscriptions.roomChatMessageReceived.skip = false
        this.$apollo.subscriptions.roundFinished.skip = false
        this.$apollo.subscriptions.roundStarted.skip = false
        this.$apollo.subscriptions.solveUpdated.skip = false
        this.$apollo.subscriptions.roomUpdated.skip = false
        this.$apollo.subscriptions.roomMembersUpdated.skip = false
      } catch (err) {
        sharedService.handleError(err, this.$root)
        this.errorMessage = sharedService.sanitizeErrorMessage(err.message)
      }
      this.loading.joiningRoom = false
    },

    async startNextRound() {
      this.loading.startingNextRound = true
      try {
        await this.$apollo.mutate({
          mutation: NEXT_ROUND_MUTATION,
          variables: {
            room_id: this.$route.query.id,
          },
        })
      } catch (err) {
        sharedService.handleError(err, this.$root)
        this.loading.startingNextRound = false
      }
      //we will turn loading off when the subscription returns the next round
      //this.loading.startingNextRound = false;
    },

    async sendChatMessage() {
      try {
        if (this.$refs.chatbox) {
          this.$refs.chatbox.blur()
        }

        if (!this.chatMessage) {
          return
        }

        const message = this.chatMessage
        this.chatMessage = null

        await this.$apollo.mutate({
          mutation: SEND_ROOM_CHAT_MESSAGE_MUTATION,
          variables: {
            room_id: this.$route.query.id,
            message: message,
          },
          fetchPolicy: 'no-cache',
        })
      } catch (err) {
        sharedService.handleError(err, this.$root)
      }
    },

    //this is for submitting a completed solve
    async updateSolve(solveResult) {
      this.loading.updatingSolve = true
      try {
        if (!this.activeRound) {
          throw sharedService.generateError('No active round')
        }

        if (!this.currentUserSolve) {
          throw sharedService.generateError(
            'Could not find your solve in this round',
          )
        }

        await this.$apollo.mutate({
          mutation: UPDATE_SOLVE_MUTATION,
          variables: {
            solve_id: this.currentUserSolve.id,
            ...solveResult,
          },
        })
      } catch (err) {
        sharedService.handleError(err, this.$root)
      }
      this.loading.updatingSolve = false
    },

    handleTimerStateUpdate(timerState) {
      //solve must progress
      if (
        timerState > this.timerState &&
        (timerState == 1 || timerState == 4 || timerState == 5)
      ) {
        this.updateSolveState(timerState)
      }

      this.timerState = timerState
    },

    handleTimerStatusUpdate(status) {
      this.timerStatus = status
    },

    async updateSolveState(timerState) {
      //this.loading.updatingSolve = true;
      try {
        if (!this.activeRound) {
          throw sharedService.generateError('No active round')
        }

        if (!this.currentUserSolve) {
          throw sharedService.generateError(
            'Could not find your solve in this round',
          )
        }

        let solveState
        if (timerState == 1) {
          solveState = 'INSPECTING'
        } else if (timerState == 4) {
          solveState = 'SOLVING'
        } else if (timerState == 5) {
          solveState = 'VERIFYING'
        }

        if (!solveState) {
          throw sharedService.generateError('Invalid Solve State')
        }

        await this.$apollo.mutate({
          mutation: UPDATE_SOLVE_MUTATION,
          variables: {
            solve_id: this.currentUserSolve.id,
            state: solveState,
          },
        })
      } catch (err) {
        sharedService.handleError(err, this.$root)
      }
      //this.loading.updatingSolve = false;
    },

    startTutorial() {
      this.$tours['myTour'].start()
    },

    async toggleIdleMode() {
      this.loading.idling = true
      try {
        await this.$apollo.mutate({
          mutation: IDLE_MUTATION,
        })

        sharedService.generateSnackbar(this.$root, 'Idled User', 'success')
      } catch (err) {
        sharedService.handleError(err, this.$root)
      }
      this.loading.idling = false
    },

    async assignNewManager(cuberId) {
      this.loading.assigningManager = true
      try {
        let { data } = await this.$apollo.mutate({
          mutation: GRANT_ROOM_MANAGEMENT_TO_CUBER_MUTATION,
          variables: {
            room_id: this.$route.query.id,
            cuber_id: cuberId,
          },
        })

        //optimistically update the manager
        this.room.manager = data.grantRoomManagementToCuber.manager

        sharedService.generateSnackbar(
          this.$root,
          'Assigned New Manager',
          'success',
        )
      } catch (err) {
        sharedService.handleError(err, this.$root)
      }
      this.loading.assigningManager = false
    },

    reset() {
      this.errorMessage = null

      //load room settings
      if (process.client) {
        const savedSettings = localStorage.getItem('roomSettings')
        if (savedSettings) {
          const parsedSettings = JSON.parse(savedSettings)

          for (const prop in this.settingsObject) {
            if (prop in parsedSettings) {
              this.settingsObject[prop] = parsedSettings[prop]
            }
          }
        }
      }

      this.joinAndLoadRoom()
    },
  },

  head() {
    return {
      title: 'Room' + (this.room ? ' - ' + this.room.name : ''),
      meta: [
        {
          hid: 'description',
          name: 'description',
          content: 'Cube.Zone is a new online platform for social cubing',
        },
      ],
    }
  },

  apollo: {
    room: {
      query: ROOM_QUERY,
      variables() {
        return {
          id: this.$route.query.id,
        }
      },
      manual: true,
      skip: true,
      result({ data }) {
        if (++this.generation.room % 2 == 0) {
          this.room = data.room
          //allow rounds query to start after this is done.
          this.$apollo.queries.rounds.skip = false
        }
      },
    },

    rounds: {
      query: ROUNDS_QUERY,
      variables() {
        return {
          first: this.tableRounds,
          page: 0,
          sorting: ['ROUND_NUMBER_DESC'],
          room_id: this.$route.query.id,
        }
      },
      manual: true,
      skip: true,
      result({ data }) {
        if (++this.generation.rounds % 2 == 0) {
          if (this.cuberResults.length) {
            //subsequent load (due to subscription)
          } else {
            this.rounds = data.rounds.data
            //initial load
            this.updateActiveRound(this.rounds[0])
            this.findCurrentUserSolve()
            this.updateSolvesMap(this.rounds)
            this.buildCuberResults()
            this.updateActiveCubers(this.room.active_cubers.data, true)
          }
        }
      },
    },

    $subscribe: {
      roomChatMessageReceived: {
        query: ROOM_CHAT_MESSAGE_RECEIVED_SUBSCRIPTION,
        variables() {
          return {
            room_id: this.$route.query.id,
          }
        },
        result({ data }) {
          this.handleChatMessageReceived(data.roomChatMessageReceived)
        },
        fetchPolicy: 'no-cache',
        skip: true,
      },

      roundFinished: {
        query: ROUND_FINISHED_SUBSCRIPTION,
        variables() {
          return {
            room_id: this.$route.query.id,
          }
        },
        result({ data }) {
          //set the loading state on the start round to true
          //this.loading.startingNextRound = true;

          //first, ensure that the finished round data auto-syncs with the one on file
          let foundRound = this.rounds.find(
            (round) => round.id == data.roundFinished.id,
          )

          if (foundRound) {
            this.updateObject(foundRound, data.roundFinished)
          }
        },
        fetchPolicy: 'no-cache',
        skip: true,
      },

      roundStarted: {
        query: ROUND_STARTED_SUBSCRIPTION,
        variables() {
          return {
            room_id: this.$route.query.id,
          }
        },
        result({ data }) {
          this.rounds.unshift(data.roundStarted)

          this.updateActiveRound(data.roundStarted)

          this.findCurrentUserSolve()
          this.updateSolvesMap([data.roundStarted])
          this.cuberResults.length
            ? this.addCuberResults(data.roundStarted)
            : this.buildCuberResults()
          this.checkCuberResults()
          if (!this.pageFocused && this.settingsObject.enableSounds) {
            sharedService.playSound('/alert2.mp3')
          }
          this.triggerGarbageCollection()
          this.loading.startingNextRound = false
        },
        fetchPolicy: 'no-cache',
        skip: true,
      },

      solveUpdated: {
        query: SOLVE_UPDATED_SUBSCRIPTION,
        variables() {
          return {
            room_id: this.$route.query.id,
          }
        },
        result({ data }) {
          if (data.solveUpdated.id in this.solvesMap) {
            this.updateObject(
              this.solvesMap[data.solveUpdated.id],
              data.solveUpdated,
            )
          }
          //flash the user time that just got updated
          this.flashResultUpdating(
            this.cuberResults.find(
              (cuber) => cuber.id == data.solveUpdated.cuber.id,
            ),
          )
        },
        fetchPolicy: 'no-cache',
        skip: true,
      },

      roomUpdated: {
        query: ROOM_UPDATED_SUBSCRIPTION,
        variables() {
          return {
            room_id: this.$route.query.id,
          }
        },
        result({ data }) {
          Object.assign(this.room, data.roomUpdated)
        },
        fetchPolicy: 'no-cache',
        skip: true,
      },

      roomMembersUpdated: {
        query: ROOM_MEMBERS_UPDATED_SUBSCRIPTION,
        variables() {
          return {
            room_id: this.$route.query.id,
          }
        },
        result({ data }) {
          this.room.active_cubers.data = data.roomMembersUpdated
          this.updateActiveCubers(this.room.active_cubers.data)
        },
        fetchPolicy: 'no-cache',
        skip: true,
      },
    },
  },

  beforeRouteLeave(to, from, next) {
    const answer = window.confirm('Do you really want to leave this room?')
    if (answer) {
      this.$apollo.mutate({
        mutation: LEAVE_ROOM_MUTATION,
        variables: {
          room_id: this.$route.query.id,
        },
      })
      next()
    } else {
      next(false)
    }
  },
}
</script>

<style scoped>
@keyframes yellowfade {
  from {
    background: yellow;
  }
  to {
    background: transparent;
  }
}

.item-highlight {
  animation: yellowfade 1s;
}

.chat-box {
  width: 100%;
}

.td-border-left {
  border-left: thin solid rgba(255, 255, 255, 0.12);
}

.pointer-cursor {
  cursor: pointer;
}

.container-collapsed {
  max-height: calc(100% - 320px);
}

.container-full {
  max-height: 100%;
}
</style>

<style>
#roomResults > .v-data-table__wrapper {
  max-height: 500px;
}

.winner-text {
  color: green;
  font-weight: bold;
}
</style>
