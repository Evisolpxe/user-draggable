<template>
  <b-container>
    <form>
      <b-form-group
        label="list"
        label-for="exampleFormControlTextarea1"
        :description="`list of user id (current: ${idList.length} players.)`"
      >
        <b-form-textarea id="exampleFormControlTextarea1" rows="3" v-model.trim.lazy="idInput"></b-form-textarea>
        <b-form-invalid-feedback :state="idInputState">{{appendNotice}}</b-form-invalid-feedback>
        <b-form-valid-feedback :state="idInputState">{{validNotice}}</b-form-valid-feedback>
      </b-form-group>
      <b-form-group id="input-group-1" label="group size:" label-for="input-group">
        <b-form-input
          id="input-group"
          v-model="groupSize"
          type="number"
          required
          placeholder="0"
          :state="groupSizeState"
        ></b-form-input>
        <b-form-invalid-feedback :state="groupSizeState">需要大于1！</b-form-invalid-feedback>
      </b-form-group>
    </form>
    <span v-if="loading">
      <h4>
        <b-spinner></b-spinner>
        Loading...({{this.forShow_Listed.length}} / {{this.idList.length}})
      </h4>
    </span>
    <router-link
      v-if="ready"
      :to="{
            name:'Select',
            params:{
                list: players,
                groupSize,
            },
        }"
    >
      <b-button :disabled="loading">去分组11111</b-button>
    </router-link>
    <b-list-group>
      <Player v-for="player in forShow_Listed" :key="player.id" :player="player" />
    </b-list-group>
  </b-container>
</template>

<script>
import Player from "../components/player.vue";
const node_osu = require("node-osu");
const i = new node_osu.Api("A3tGREAemXk213gfJJUewH9675g", {
  // baseUrl: sets the base api url (default: https://osu.ppy.sh/api)
  notFoundAsError: true, // Throw an error on not found instead of returning nothing. (default: true)
  completeScores: false, // When fetching scores also fetch the beatmap they are for (Allows getting accuracy) (default: false)
  parseNumeric: true // Parse numeric values into numbers/floats, excluding ids
});
const User = node_osu.User;

export default {
  name: "inputList",
  data: () => ({
    idInput: "",
    idInputState: true,
    groupSize: 8,
    groupSizeState: true,
    cacheUsers: [],
    idList: [],
    players: [],
    ready: false,
    loading: false,
    appendNotice: "",
    validNotice: "",
      userElo: {},
  }),
  computed: {
    forShow_Listed: function() {
      return this.idList
        .map(id => {
          const user = this.cacheUsers.find(user => user.id == id);
          return user;
        })
        .filter(user => user);
    }
  },
  asyncComputed: {},
  methods: {
    async getUser({ id }) {
      if (id) {
        let user = undefined;
        try {
          user = new User(
            i.config,
            await fetch(`http://47.101.168.165:5005/api/users/${id}`)
              .then(res => res.json())
              .then(res => res[0])
              .catch(() => undefined)
          );
        } catch (e) {
          return undefined;
        }
        let elo = await fetch(`http://47.101.168.165:5005/api/users/elo/${id}`)
            .then(res=> res.json())
            .then(res=>user.elo = res.elo).catch(()=>0);
        console.log({ name: user.name, id: user.id, elo: elo});
        this.cacheUsers.push(user);
        return user;
      }
    },
    async fetchPlayers() {
      this.ready = false;
      this.loading = true;
      let rtn = await Promise.all(
        this.idList.map(async id => {
          const user = this.cacheUsers.find(user => user.id == id);
          if (user) {
            return user;
          } else {
            return await this.getUser({ id });
          }
        })
      );
      rtn = rtn.filter(user => user);
      if (rtn.length > 0) this.ready = true;
      this.loading = false;
      this.players = rtn;
      return rtn;
    },
    onlyUnique(value, index, self) {
      return self.indexOf(value) === index;
    },
    isNumeric(n) {
      return !isNaN(parseFloat(n)) && isFinite(n);
    }
  },
  watch: {
    idInput() {
      let idList = this.idInput.split("\n");
      this.idList = idList.filter(this.onlyUnique);
      if (idList.length > this.idList.length) {
        this.idInputState = true;
        this.validNotice = `列表疵了。有重复的id！！不过我给你删了。`;
      }
    },
    idList() {
      if (this.idInputState) this.fetchPlayers();
    },
    groupSize(val) {
      if (val > 1) this.groupSizeState = true;
      else this.groupSizeState = false;
    }
  },
  components: {
    Player
  }
};
</script>
