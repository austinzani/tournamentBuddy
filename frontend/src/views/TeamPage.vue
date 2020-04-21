<template>
  <v-container>
    <v-row>
      <v-col cols="6" sm="3">
        <v-img :src="getImage()" class="my-3" contain height="150" />
      </v-col>
      <v-col cols="5" sm="3">
        <h1>{{team.teamName}}</h1>
        <h2>{{team.game}}</h2>
      </v-col>
      <v-col cols="12" sm="6">
        <v-row>
          <v-col>
            <h3>Team Bio:</h3>
          </v-col>
        </v-row>
        <p>{{team.teamBio}}</p>
      </v-col>
    </v-row>
    <v-row>
      <v-spacer></v-spacer>
      <v-col cols="12" sm="4">
        <edit-team :current-team="team" @update-team="getTeam(); teamEvent()" />
      </v-col>
      <v-spacer></v-spacer>
      <v-col cols="12" sm="4">
        <accept-applicants :current-team="team" @request-update="getRoster()"/>
      </v-col>
      <v-spacer></v-spacer>
    </v-row>
    <v-row>
      <v-col>
        <v-card>
          <v-card-title>
            Team Roster
            <v-spacer></v-spacer>
            <v-text-field
              v-model="searchRoster"
              append-icon="mdi-magnify"
              label="Search Team Roster"
              single-line
              hide-details
            ></v-text-field>
          </v-card-title>
          <v-data-table color="#03DAC5" :headers="rosterHeaders" :items="roster" :search="searchRoster">
            <template v-slot:item="row">
              <tr>
                <td>{{row.item.username}}</td>
                <td>{{row.item.firstName}}</td>
                <td>{{row.item.lastName}}</td>
                <td>{{row.item.email}}</td>
                <td>
                  <v-icon v-if="row.item.role === 'captain'">mdi-check-bold</v-icon>
                </td>
                <td>
                  <v-icon
                    color="red lighten-2"
                    v-if="captainedTeams.includes(team.teamId)"
                    small
                    @click="deleteMember(row.item.id)"
                  >mdi-delete</v-icon>
                </td>
              </tr>
            </template>
          </v-data-table>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>

<script>
import EditTeam from "@/components/EditTeam.vue";
import auth from "@/auth";
import api from "@/api";
import img from "@/image.js";
import AcceptApplicants from "@/components/AcceptApplicants.vue";

export default {
  components: {
    EditTeam,
    AcceptApplicants
  },
  props: {
    currentTeam: Object
  },
  data() {
    return {
      startingTeam: null,
      searchRequest: "",
      searchRoster: "",
      pendingHeaders: [
        { text: "Username", value: "senderName" },
        { text: "Message", value: "message" },
        { text: "accept/decline", value: "accept/decline" }
      ],
      teamNameRules: [
        v => !!v || "Team name is required.",
        v => v.length >= 8 || "Team name must be at least 4 characters."
      ],

      rosterHeaders: [
        { text: "Username", value: "username" },
        { text: "First Name", value: "firstName" },
        { text: "Last Name", value: "lastName" },
        { text: "Email Address", value: "email" },
        { text: "Captain", value: "captain", sortable: false },
        { text: "", value: "delete" }
      ],

      requests: [],

      roster: [],

      captainedTeams: []
    };
  },

  created() {
    this.getTeam();

    api.getCaptainedTeams().then(result => (this.captainedTeams = result));
  },

  methods: {
    getTeam() {
      const teamId = this.$route.params.id;
      fetch(`${process.env.VUE_APP_REMOTE_API}/api/team/${teamId}`, {
        method: "GET",
        headers: {
          Authorization: "Bearer " + auth.getToken()
        }
      })
        .then(response => {
          return response.json();
        })
        .then(data => {
          this.startingTeam = data;
          this.getRequests();
          this.getRoster();
        });
    },

    getRequests() {
      fetch(
        `${process.env.VUE_APP_REMOTE_API}/api/team/request?teamId=${this.$route.params.id}`,
        {
          method: "GET",
          headers: {
            Authorization: "Bearer " + auth.getToken()
          },
          credentials: "same-origin"
        }
      )
        .then(response => {
          return response.json();
        })
        .then(data => (this.requests = data));
    },

    getRoster() {
      fetch(
        `${process.env.VUE_APP_REMOTE_API}/api/user?teamId=${this.$route.params.id}`,
        {
          method: "GET",
          headers: {
            Authorization: "Bearer " + auth.getToken()
          },
          credentials: "same-origin"
        }
      )
        .then(response => {
          return response.json();
        })
        .then(members => (this.roster = members));
    },

    deleteMember(userId) {
      const teamId = this.team.teamId;

      confirm("Are you sure you want to delete this team member?") &&
        fetch(
          `${process.env.VUE_APP_REMOTE_API}/api/team/roster?userId=${userId}&teamId=${teamId}`,
          {
            method: "DELETE",
            headers: {
              Authorization: "Bearer " + auth.getToken()
            },
            credentials: "same-origin"
          }
        ).then(response => {
          if (response.ok) {
            this.getRoster(teamId);
          }
        });
    },

    acceptRequest(request) {
      const teamId = this.team.teamId;

      confirm("Are you sure you want to accept this request?") &&
        fetch(
          `${process.env.VUE_APP_REMOTE_API}/api/team/roster?captainStatus=false`,
          {
            method: "POST",
            headers: {
              Authorization: "Bearer " + auth.getToken(),
              Accept: "application/json",
              "content-type": "application/json"
            },
            body: JSON.stringify(request)
          }
        ).then(response => {
          if (response.ok) {
            this.deleteRequest(request);
            this.getRoster(teamId);
          }
        });
    },

    deleteRequest(request) {
      const teamId = this.team.teamId;

      fetch(`${process.env.VUE_APP_REMOTE_API}/api/team/request`, {
        method: "DELETE",
        headers: {
          Authorization: "Bearer " + auth.getToken(),
          Accept: "application/json",
          "content-type": "application/json"
        },
        body: JSON.stringify(request)
      }).then(response => {
        if (response.ok) {
          this.getRequests(teamId);
        }
      });
    },
    teamEvent(){
      this.$emit('update-team', this.team.teamId)
    },
    getImage(){
      return img.getImage(this.team.teamId);
    }
  },
  computed: {
    team() {
      if (this.currentTeam !== null) {
        this.getRequests();
        this.getRoster();
        return this.currentTeam;
      } else {
        return this.startingTeam;
      }
    }
  }
};
</script>