<template>
  <v-row justify="center">
    <v-dialog v-model="dialog" persistent max-width="600px">
      <template v-slot:activator="{ on }">
        <v-btn color="#03DAC5"
            outlined dark v-on="on">Member Requests</v-btn>
      </template>
      <v-card>
        <v-card-title>
          <span class="headline">Member Requests</span>
          <v-spacer></v-spacer>
          <v-icon class="clickable" ripple @click="dialog = false">mdi-close-box-outline</v-icon>
        </v-card-title>
        <v-data-table :headers="headers" :items="requests" class="elevation-10">
          <template v-slot:item="row">
            <tr>
              <td>{{row.item.senderName}}</td>
              <td>{{row.item.message}}</td>
              <td>
                <v-row class="justify-end">
                <v-btn
                  class="mx-2"
                  fab
                  dark
                  small
                  color="#03DAC5"
                  @click="dialog = false;acceptRequest(row.item)"
                >
                  <v-icon dark>mdi-check-circle-outline</v-icon>
                </v-btn>
                <v-btn
                  class="mx-2"
                  fab
                  dark
                  small
                  color="red lighten-2"
                  @click="dialog = false;deleteRequest(row.item)"
                >
                  <v-icon dark>mdi-minus-circle-outline</v-icon>
                </v-btn>
                </v-row>
              </td>
            </tr>
          </template>
        </v-data-table>
      </v-card>
    </v-dialog>
  </v-row>
</template>

<script>
import auth from "@/auth.js";
export default {
  props: {
    currentTeam: Object
  },
  data() {
    return {
      team: null,
      dialog: false,
      requests: [
      ]
    };
  },
  methods: {
    setTeam() {
      this.team = Object.assign({}, this.currentTeam);
    },
    getTeamRequests() {
      fetch(
        `${process.env.VUE_APP_REMOTE_API}/api/team/request?teamId=${this.team.tournamentId}`,
        {
          headers: {
            Authorization: "Bearer " + auth.getToken(),
            Accept: "application/json",
            "content-type": "application/json"
          }
        }
      )
        .then(response => {
          if (response.ok) {
            return response.json();
          }
        })
        .then(data => {
          this.requests = data;
        })
        .catch(err => console.error(err));
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
            this.$emit("request-update");
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
          this.$emit("request-update");
        }
      });
    }
  },
  created() {
    this.setTeam();
    this.getTeamRequests();
  }
};
</script>

<style>
</style>