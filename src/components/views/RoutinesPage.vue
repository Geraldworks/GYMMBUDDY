<template>
  <div class="routines-page">
    <Navbar />
    <div class="routine-page-header">Your Routines</div>

    <div class="content-container">
      <RoutineContent :email="email" :userFullName="userFullName" />
    </div>
  </div>
</template>

<script>
import { collection, getDocs, query, where } from "firebase/firestore";
import { mapGetters, useStore } from "vuex";
import { db, auth } from "../../firebase.js";
import RoutineContent from "../client/RoutineContent.vue";

export default {
  name: "RoutinesPage",
  computed: {
    ...mapGetters(["user"]),
  },
  data() {
    return {
      email: "",
      userFullName: "",
    };
  },
  mounted() {
    const store = useStore();
    auth.onAuthStateChanged((user) => {
      store.dispatch("fetchUser", user);
    });
  },
  async created() {
    // firestore querying for the correct information
    const clientRef = collection(db, "client");
    const thisClientQuery = query(
      clientRef,
      where("email", "==", this.user.data.email)
    );
    const clientQuerySnapshot = await getDocs(thisClientQuery);

    let clientName;
    clientQuerySnapshot.forEach((doc) => {
      clientName = doc.data().fullName;
    });

    this.email = this.user.data.email;
    this.userFullName = clientName;
    // console.log("--- Routines Page ---");
    // console.log(this.email);
    // console.log(this.userFullName);
    // console.log(this.fullName);
  },
  methods: {
    // if using this method, ensure RoutineContent $emit() & listen to it
    // reloadRoutinesPage() {
    //   // window.location.reload();
    //   this.$router.go(0);
    // },
  },
  components: {
    RoutineContent,
  },
  // watch: {
  //   email(newEmail) {
  //     console.log("watch:", newEmail);
  //   },
  //   fullName(newFullName) {
  //     console.log("watch:", newFullName);
  //   },
  // },
};
</script>

<style scoped>
@import url("https://fonts.googleapis.com/css2?family=Hanken+Grotesk&family=Teko:wght@500;600&display=swap");

.routine-page-header {
  color: white;
  border-bottom: 5px white solid;
  font-size: 3rem;
  text-transform: uppercase;
  width: 80%;
  margin: auto;
  padding-top: 20px;
}

.routines-page {
  background-color: black;
  overflow-y: hidden;
  min-width: 800px; /* Or else the PerformanceBottom component will overflow into the side margin */
  padding-bottom: 5%;
  min-height: 100vh;
  font-family: Teko;
}

.content-container {
  background-color: #d9d9d9;
  margin: 0vw 10vw;
  border-radius: 25px;
  padding: 30px 60px;
  margin-top: 22px;
  max-height: 59vh;
  overflow-y: auto;
}

@media screen and (max-width: 800px) {
  .routines-page {
    padding-bottom: calc(600px - 10vh);
  }
  .content-container {
    max-height: 80vh;
  }
}

@media screen and (max-width: 550px) {
  .routines-page {
    padding-bottom: calc(850px - 10vh);
  }
}

@media screen and (max-width: 400px) {
  .routines-page {
    padding-bottom: calc(1200px - 10vh);
  }
}

body::-webkit-scrollbar {
  display: none;
}
</style>