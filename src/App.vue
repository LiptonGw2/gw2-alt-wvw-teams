<template>
  <div class="topContainer">
    <button @click="showApiKeyManagement = !showApiKeyManagement">
      <template v-if="showApiKeyManagement">Hide</template>
      <template v-else>Show</template> Api Keys
    </button>
    <template v-if="showApiKeyManagement">

      <ImportButton @loaded="onImportDataLoaded" :buttonText="'Import Api Keys'" :accept="'.json,.txt'"></ImportButton>
      <button @click="exportApiKeys">Export Api Keys</button>

      <div class="p-relative">
        <a class="gw2-account-link" href="https://account.arena.net/applications"
          target="_blank">https://account.arena.net/applications</a>
        <div>
          <input ref="addKeyInput" type="text" v-model="newApiKey" v-on:keyup.enter="addApiKey" class="add-input"
            placeholder="Api Key">
          <button @click="addApiKey" :disabled="!newApiKey" class="add-btn">Add</button>
        </div>
      </div>
    </template>
  </div>

  <div>

    <div v-if="showApiKeyManagement">
      <table class="api-keys">
        <tr v-for="(entry, idx) in apiKeys">
          <td>{{ entry.name }}</td>
          <td>{{ entry.apiKey }}</td>
          <td>
            <button @click="removeApiKey(idx)">X</button>
          </td>
        </tr>
      </table>
    </div>

    <div v-if="apiKeys.length">
      <button @click="loadInfo">Load</button>

      <div v-if="matches">
        <h3>WvW Matches</h3>

        <table class="matchesOverview jtm-table">
          <tbody>
            <tr v-for="m in matches.filter(x => x.hasAccountsInMatch)">
              <td>{{ m.region }}</td>
              <td>{{ m.tier }}</td>
              <template v-for="team in m.teams">
                <td>
                  <div v-bind:class="'text-' + team.color">
                    {{ team.name }}
                  </div>
                  <div v-for="acc in team.accounts">- {{ acc.name }}</div>
                </td>
                <td v-bind:class="'text-' + team.color">{{ m.kds[team.color] }}</td>
              </template>
            </tr>
          </tbody>
        </table>
      </div>

      <table class="account-overview jtm-table">
        <tr v-for="(entry, idx) in apiKeys">
          <td>{{ idx + 1 }}.</td>
          <td>{{ entry.region }}</td>
          <td>{{ entry.team }}</td>
          <td>{{ entry.name }}</td>
        </tr>
      </table>
    </div>

  </div>
</template>

<script>
import teamNames from "./teamNames.json";
import ImportButton from "./components/ImportButton.vue";

const LOCAL_STORAGE_SAVE_KEY = "API_KEYS";

export default {
  components: { ImportButton },
  data() {
    return {
      showApiKeyManagement: true,
      newApiKey: null,
      teamNames,
      apiKeys: [],
      matches: null
    }
  },
  methods: {

    addApiKey() {
      let newApiKey = (this.newApiKey || "").trim();

      if (!newApiKey || this.apiKeys.find(x => x.apiKey == newApiKey))
        return;

      this.apiKeys.push({
        apiKey: newApiKey,
        name: null,
        team_id: null,
        team: null,
        region: null
      });

      this.saveApiKeys();

      this.newApiKey = null;

      if (this.$refs.addKeyInput)
        this.$refs.addKeyInput.focus();
    },

    saveApiKeys() {
      localStorage.setItem(LOCAL_STORAGE_SAVE_KEY, JSON.stringify(this.apiKeys));
    },

    getSavedApiKeys() {
      let apiKeys = localStorage.getItem(LOCAL_STORAGE_SAVE_KEY);
      if (apiKeys) {
        apiKeys = JSON.parse(apiKeys);
        if (!(apiKeys instanceof Array))
          apiKeys = [];
      }
      this.apiKeys = apiKeys || [];
      this.showApiKeyManagement = this.apiKeys.length == 0;
    },

    removeApiKey(idx) {
      this.apiKeys.splice(idx, 1);
      this.saveApiKeys();
    },

    loadInfo() {
      if (this.loading)
        return;

      this.loading = true;

      let promises = [];

      for (let entry of this.apiKeys) {
        let url = "https://api.guildwars2.com/v2/account?v=2025-02-12T00:00:00Z&access_token=" + entry.apiKey;

        promises.push(
          fetch(url)
            .then(response => {
              if (!response.ok) {
                throw new Error(`HTTP error! Status: ${response.status}`);
              }
              return response.json();
            })
            .then(data => {
              entry.name = data.name;

              if (data.wvw)
                entry.team_id = data.wvw.team_id;
              if (entry.team_id) {
                let team = teamNames.find(x => x.id == entry.team_id);
                if (team) {
                  entry.team = team.name;
                  entry.region = team.region;
                }
                else {
                  entry.team = entry.region = null;
                }
              }
            })
            .catch(error => {
              console.error('Error fetching data:', error);
            }));
      }

      Promise.all(promises).then(() => {
        this.loading = false
        this.saveApiKeys();
        this.checkMatchAccounts();
      });
    },

    checkMatchAccounts() {
      if (!this.matches)
        return;

      for (let match of this.matches) {

        for (let team of match.teams) {
          team.accounts = this.apiKeys.filter(x => x.team_id == team.id);
        }

        match.hasAccountsInMatch = !!match.teams.find(x => x.accounts.length);
      }
    },

    loadMatchInfo() {

      let ids = "2-1,2-2,2-3,2-4,2-5,1-1,1-2,1-3,1-4";

      fetch("https://api.guildwars2.com/v2/wvw/matches?ids=" + ids)
        .then(response => {
          if (!response.ok)
            throw new Error(`HTTP error! Status: ${response.status}`);
          return response.json();
        })
        .then(data => {
          let matches = [];

          let allTeamIds = teamNames.map(x => x.id);

          for (let entry of data) {

            let id = entry.id.split("-");

            let teams = [];
            let kds = {};
            for (let color in entry.all_worlds) {
              let teamId = entry.all_worlds[color].find(x => allTeamIds.includes(x));
              let team = teamNames.find(x => x.id == teamId);
              if (team) {
                teams.push({
                  color,
                  name: team.name,
                  id: teamId,
                  accounts: []
                })
              }
              kds[color] = this.roundNumber(entry.kills[color] / entry.deaths[color]);
            }

            matches.push({
              region: id[0] == "2" ? "EU" : "NA",
              tier: id[1],
              kills: entry.kills,
              deaths: entry.deaths,
              kds,
              teams,
              hasAccountsInMatch: false
            });
          }

          this.matches = matches;
          this.checkMatchAccounts();

          console.log(this.matches);
        });
    },

    exportApiKeys() {
      if (!this.apiKeys.length)
        return;

      const filename = "apiKeys.json";
      const jsonStr = JSON.stringify(this.apiKeys, null, 2);
      const blob = new Blob([jsonStr], { type: "application/json" });
      const url = URL.createObjectURL(blob);

      const a = document.createElement("a");
      a.href = url;
      a.download = filename;
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
      URL.revokeObjectURL(url);
    },

    onImportDataLoaded(data) {

      try {
        const jsonData = JSON.parse(data);

        if (jsonData instanceof Array)
          this.apiKeys = jsonData;

      } catch (error) {

        let lines = data.split("\n");

        for (let line of lines) {
          if (!line)
            continue;

          let apiKey = line.trim();

          if (!this.apiKeys.find(x => x.apiKey == apiKey)) {
            this.apiKeys.push({
              apiKey: newApiKey,
              name: null,
              team_id: null,
              team: null,
              region: null
            });
          }
        }
      }

      this.loadInfo();
    },

    roundNumber(number) {
      return Math.round(number * 100) / 100;
    }
  },
  mounted() {
    this.getSavedApiKeys();
    this.loadMatchInfo();

    document.addEventListener("visibilitychange", () => {
      if (document.visibilityState == "visible") {
        document.title = "jtm";
      } else {
        document.title = "mjt";
      }
    });

    window.app = this;
  }
}
</script>

<style>
.name {
  text-align: center;
  font-weight: bolder;
  font-size: 20px;
}

.long-name {
  font-size: 16px !important;
}

.v-long-name {
  font-size: 15px !important;
}
</style>
