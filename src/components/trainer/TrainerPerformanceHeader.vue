<template>
  <div class="profile-container">
    <!--Profile Picture-->
    <img class="profile-picture" :src="profilePic" alt="dp" />
    <div class="your-profile">
      <div class="trainer-header">
        <div>
          Viewing
          <span style="color: #ed1f24">{{ clientName.split(" ")[0] }}</span>
        </div>
        <!-- buttons that the trainer can click to navigate and understand the client better -->
        <div class="trainer-options">
          <div class="right-side-items" @click="routeToPerformancePage()">
            Performance
          </div>
          <div class="right-side-items" @click="routeToRoutinePage()">
            Routines
          </div>
          <div class="right-side-items" @click="showUpdateForm()">
            Update Stats
          </div>
          <div class="pop-up">
            <UpdateForm2
              v-show="updateForm"
              :clientEmail="email"
              :clientName="clientName"
              @close-modal="updateForm = false"
            />
          </div>
          <!-- If we detect that the trainer wants to return, we will emit "refresh" to 
          go back to the page with all the client cards-->
          <div class="right-side-items" @click="returnBackToTrainerHomePage()">
            Return
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { doc, collection, getDocs, query, where } from "firebase/firestore";
import { db } from "../../firebase.js";
import UpdateForm2 from "./UpdateForm2.vue";
import RoutineContent from "../client/RoutineContent.vue";

export default {
  name: "TrainerPerformanceHeader",
  components: {
    UpdateForm2,
    RoutineContent,
  },
  data() {
    return {
      nameToDisplay: "",
      clientName: "",
      updateForm: false,
    };
  },
  props: {
    email: String,
    profilePic: String,
  },
  methods: {
    // returns to the trainer menu selection page
    returnBackToTrainerHomePage() {
      this.$emit("returnToHome");
    },
    // popups up the update form when this method is run
    showUpdateForm() {
      this.updateForm = true;
    },
    // displays the routine page when this method is run
    routeToRoutinePage() {
      this.$emit("routeToRoutine");
    },
    // displays the client performance page when this method is run
    routeToPerformancePage() {
      this.$emit("routeToPerformance");
    },
  },
  emits: ["returnToHome", "routeToRoutine", "routeToPerformance"],
  async created() {
    try {
      // data base querying for the correct client information to render
      const clientRef = collection(db, "client");
      const q = query(clientRef, where("email", "==", this.email));
      const querySnapshot = await getDocs(q);

      // retrieving all the needed client information
      querySnapshot.forEach((doc) => {
        let documentData = doc.data();
        let clientName = documentData.fullName;
        this.clientName = clientName;
      });
    } catch (error) {
      console.log(error);
      console.log("No email observed in database");
    }
  },
};
</script>

<style scoped>
@import url("https://fonts.googleapis.com/css2?family=Hanken+Grotesk&family=Teko:wght@500;600&display=swap");

.profile-container {
  padding: 20px 10vw;
  display: flex;
  justify-content: flex-start;
  font-family: Teko;
  background-color: black;
  color: white;
  background-image: url("@/assets/images/profile_bg.jpg");
  background-size: contain;
}

.profile-picture {
  border-radius: 50%;
  width: 100px;
  height: 100px;
  border: 2px solid white;
  object-fit: cover;
}

.your-profile {
  font-size: 3rem;
  padding-left: 30px;
  width: 1000px;
  flex: 2;
  text-transform: uppercase;
}

.your-profile > div {
  border-bottom: 5px solid white;
}

.trainer-header {
  display: flex;
  flex-wrap: wrap;
  flex-direction: row;
  justify-content: space-between;
}

.trainer-options {
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;
}
.right-side-items {
  margin-left: auto;
  font-size: 0.6em;
  margin-top: 16px;
  padding: 0px 10px;
}

.right-side-items:hover {
  display: inline;
  cursor: pointer;
  animation: turn-red;
  animation-duration: 0.3s;
  animation-fill-mode: forwards;
}
@keyframes turn-red {
  from {
    color: white;
  }
  to {
    color: #ed1f24;
  }
}
</style>
