<template>
  <div>
    <!-- pops up a modal when update stats is click -->
    <transition name="modal-fade">
      <div class="modal-overlay" @click="$emit('close-modal')">
        <div class="modal" @click.stop>
          <div style="font-size: 3rem">
            <b style="color: white">Updating Stats for </b>
            <b style="color: #ed1f24">{{ clientName.split(" ")[0] }}</b>
          </div>
          <hr />
          <!-- form with all the information that can be included for updating -->
          <div id="update-form">
            <form action="">
              <label for="">Fat Percentage (%) </label>
              <input
                @input="checkFatsInput(fats)"
                type="number"
                v-model="fats"
                required
                id="fats"
                placeholder="Current Fat %"
              />
              <div
                :key="refreshCount"
                v-if="showFatsError"
                class="error-message"
              >
                fat percentage must be positive and not greater than 100.
              </div>
              <label for="">Weight (kg) </label>
              <input
                @input="checkWeightInput(weight)"
                type="number"
                v-model="weight"
                required
                id="weight"
                placeholder="Current weight"
              />
              <div
                :key="refreshCount"
                v-if="showWeightError"
                class="error-message"
              >
                weight must be positive.
              </div>
              <label for="">Muscle Mass (%) </label>
              <input
                @input="checkMuscleInput(muscle)"
                type="number"
                v-model="muscle"
                required
                id="muscle"
                placeholder="Current Muscle Mass"
              />
              <div
                :key="refreshCount"
                v-if="showMuscleError"
                class="error-message"
              >
                mauscle mass must be positive and not greater than the weight.
              </div>
            </form>
          </div>
          <div>
            <button
              @click="updateDataBase()"
              style="text-align: center"
              type="submit"
            >
              Submit
            </button>
          </div>
        </div>
        <div class="close" @click="$emit('close-modal')">
          <img class="close-img" src="@/assets/images/cross-icon.png" alt="" />
        </div>
      </div>
    </transition>
  </div>
</template>

<script>
import { mapGetters } from "vuex";
import { auth, db } from "../../firebase";
import { doc, getDoc, updateDoc, Timestamp } from "@firebase/firestore";
import Swal from "sweetalert2";

// an object to create the alerts
const Toast = Swal.mixin({
  toast: true,
  position: "top-end",
  showConfirmButton: false,
  timer: 3000,
  timerProgressBar: true,
});

export default {
  name: "UpdateForm",
  data() {
    return {
      fats: null,
      weight: null,
      muscle: null,
      date: "",
      isLoading: false,
      showFatsError: false,
      showWeightError: false,
      showMuscleError: false,
    };
  },
  props: {
    clientName: String,
    clientEmail: String,
  },
  computed: {
    ...mapGetters(["user"]),
  },
  mounted() {
    auth.onAuthStateChanged((user) => {
      this.$store.dispatch("fetchUser", user);
    });
  },
  emit: ["close-modal"],
  methods: {
    async updateDataBase() {
      if (
        this.showFatsError ||
        this.showWeightError ||
        this.showMuscleError ||
        !this.fats ||
        !this.weight ||
        !this.muscle
      ) {
        Swal.fire({
          icon: "error",
          title: "Oops...",
          text: "You have some invalid / incomplete inputs",
        });
      } else {
        this.loading = true;
        const clientRef = doc(db, "client", this.clientEmail);
        const querySnapshot = await getDoc(clientRef);
        // Retrieve the client information
        const clientData = querySnapshot.data();
        let clientFatPercentage = clientData.fatPercentage;
        let clientWeight = clientData.weight;
        let clientMuscleMass = clientData.muscleMass;
        let clientDate = clientData.datetime;
        console.log(this.fats);
        //pushing the form data into the array before updating the database
        clientFatPercentage.push(this.fats);
        clientWeight.push(this.weight);
        clientMuscleMass.push(this.muscle);
        const currentDate = new Date();
        clientDate.push(Timestamp.fromDate(currentDate));
        // Updating the database
        updateDoc(clientRef, {
          fatPercentage: clientFatPercentage,
          weight: clientWeight,
          muscleMass: clientMuscleMass,
          datetime: clientDate,
        })
          .then((response) => {
            // ui feedback
            this.isLoading = false;
            this.$emit("close-modal");
            Toast.fire({
              icon: "success",
              title: "Updated Successfully",
            });
            location.reload();
          })
          .catch((error) => {
            console.log(error);
          });
      }
    },
    // validation for Fat percentage
    checkFatsInput(value) {
      this.showFatsError = value > 100 || value < 0;
    },
    // validation for Weight
    checkWeightInput(value) {
      this.checkMuscleInput(this.muscle);
      this.showWeightError = value < 0;
    },
    // validation for Muscle Mass
    checkMuscleInput(value) {
      this.showMuscleError = this.weight < value || value < 0;
    },
  },
};
</script>

<style scoped>
@import url("https://fonts.googleapis.com/css2?family=Hanken+Grotesk&family=Teko:wght@500;600&display=swap");
.modal-overlay {
  position: fixed;
  z-index: 500;
  top: -10px;
  bottom: -10px;
  left: -10px;
  right: -10px;
  display: flex;
  justify-content: center;
  background-color: rgba(0, 0, 0, 0.7);
}
.modal {
  display: flex;
  flex-direction: column;
  flex-grow: 1;
  overflow: auto;
  position: relative;
  background-color: black;
  font-family: Teko;
  height: 90%;
  width: 90%;
  margin-top: 6%;
  border-radius: 25px;
  border-style: solid;
  border-color: #ed1f24;
  border-width: 2px;
  max-width: 50%;
  max-height: 70%;
  font-size: 28px;
  padding: 30px 50px;
  text-align: center;
}
.modal::-webkit-scrollbar {
  display: none;
}
#update-form {
  background-color: white;
  border-radius: 25px;
  padding: 1rem;
  padding-bottom: 1.2rem;
  text-align: center;
}
input {
  border-radius: 25px;
  padding: 0em 0.5em;
  text-align: center;
}

form {
  display: flex;
  flex-direction: column;
  align-items: center;
}
label {
  color: black;
}

.close {
  margin: 6% 0 0 10px;
  cursor: pointer;
}
.close-img {
  width: 25px;
}
button {
  width: 120px;
  height: 50px;
  border-radius: 25px; /* half of height */
  background-color: rgb(237, 31, 36);
  border: none;
  outline: none;
  cursor: pointer;
  margin: 5px;
  margin-top: 30px;
  font-size: 1.5rem;
  text-align: center;
  box-sizing: border-box;
  color: white;
}
button:hover {
  animation-name: pill-button-highlight;
  animation-duration: 0.15s;
  animation-fill-mode: forwards;
  box-sizing: border-box;
}
select {
  height: 200px;
  border: 0.5px solid black;
  padding: 0px 3px;
  text-align: center;
}
@keyframes pill-button-highlight {
  from {
    border: 0px black solid;
  }
  to {
    border: 2px white solid;
  }
}
.modal-fade-enter,
.modal-fade-leave-to {
  opacity: 0;
}
.modal-fade-enter-active,
.modal-fade-leave-active {
  transition: opacity 0.5s ease;
}
hr {
  border: none;
  height: 4px;
  background-color: white;
  opacity: 1;
}
.error-message {
  color: red;
  font-weight: bold;
}
</style>
