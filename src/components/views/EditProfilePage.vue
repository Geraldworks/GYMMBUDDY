<template>
  <!-- Edit Profile -->
  <div class="edit-page">
    <Navbar />
    <!-- If page still loading -->
    <LoadingSpinner v-if="pageLoading" :pageLoading="pageLoading" />

    <!-- If page done loading -->
    <div v-else class="container">
      <div class="row display-words">
        <div>EDIT PROFILE</div>
      </div>
      <!-- Display Picture -->
      <div class="row picture">
        <label for="file-input">
          <img :src="displayPicture" alt="Image" class="dp" />
          <div class="overlay">Change</div>
        </label>
        <input id="file-input" type="file" @change="onFileSelected" />
      </div>
      <!-- Input fields to update personal particulars -->
      <div class="row justify-content-center">
        <div class="col-md-6">
          <div class="card background-color-change border-0">
            <div class="card-body">
              <form
                action="#"
                @submit.prevent="
                  update();
                  onUpload();
                "
              >
                <!-- Full Name -->
                <div class="form-group row py-2">
                  <label
                    for="fullName"
                    class="col-md-4 col-form-label text-md-right"
                    >FULL NAME:</label
                  >

                  <div class="col-md-6">
                    <input
                      id="fullName"
                      type="text"
                      class="form-control"
                      name="fullName"
                      value
                      required
                      autofocus
                      v-model="fullName"
                    />
                  </div>
                </div>

                <!-- Contact Number -->
                <div class="form-group row py-2">
                  <label
                    for="contactNo"
                    class="col-md-4 col-form-label text-md-right"
                    >CONTACT NUMBER:</label
                  >

                  <div class="col-md-6">
                    <input
                      id="contactNo"
                      type="text"
                      class="form-control"
                      name="contactNo"
                      required
                      v-model="contactNo"
                    />
                  </div>
                </div>

                <!-- Emergency Contact Name -->
                <div class="form-group row py-2">
                  <label
                    for="emergencyContactName"
                    class="col-md-4 col-form-label text-md-right"
                    >EMERGENCY CONTACT:</label
                  >

                  <div class="col-md-6">
                    <input
                      id="emergencyContactName"
                      type="text"
                      class="form-control"
                      name="emergencyContactName"
                      required
                      v-model="emergencyContactName"
                    />
                  </div>
                </div>

                <!-- Emergency Contact Number -->
                <div class="form-group row py-2">
                  <label
                    for="emergencyContactNo"
                    class="col-md-4 col-form-label text-md-right"
                    >EMERGENCY CONTACT NO:</label
                  >

                  <div class="col-md-6">
                    <input
                      id="emergencyContactNo"
                      type="text"
                      class="form-control"
                      name="emergencyContactNo"
                      required
                      v-model="emergencyContactNo"
                    />
                  </div>
                </div>

                <!-- Error Message shown after validation -->
                <p v-if="errorMessage" class="alert alert-danger mt-3">
                  {{ errorMessage }}
                </p>

                <!-- Save Changes Button -->
                <div class="form-group row mb-0">
                  <div class="button-div">
                    <button type="submit">
                      <div
                        v-if="isLoading"
                        class="spinner-border spinner-border-sm"
                      ></div>
                      Save
                    </button>
                  </div>
                </div>
              </form>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { mapGetters } from "vuex";
import { auth, db } from "../../firebase";
import { doc, getDoc, updateDoc } from "@firebase/firestore";
import {
  getStorage,
  ref,
  uploadBytesResumable,
  list,
  getDownloadURL,
} from "firebase/storage";
import defaultPic from "../../assets/images/default_dp.svg";
import LoadingSpinner from "../LoadingSpinner.vue";
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
  name: "EditProfilePage",
  data() {
    return {
      fullName: "",
      contactNo: "",
      emergencyContactName: "",
      emergencyContactNo: "",
      isLoading: false,
      errorMessage: "",
      displayPicture: null,
      profilePicture: null,
      pageLoading: false,
    };
  },
  components: { LoadingSpinner },
  computed: {
    ...mapGetters(["user"]),
  },
  mounted() {
    this.pageLoading = true;
    auth.onAuthStateChanged((user) => {
      this.$store.dispatch("fetchUser", user);
    });
    // Gets the current information
    const clientRef = doc(db, "client", this.user.data.email);
    getDoc(clientRef)
      .then((docSnap) => {
        const data = docSnap.data();
        // Sets the current information to data variables to be shown to user
        this.fullName = data.fullName;
        this.contactNo = data.contactNo;
        this.emergencyContactName = data.emergencyContactName;
        this.emergencyContactNo = data.emergencyContactNo;
      })
      .catch((err) => {
        console.log(err);
      });
    const storage = getStorage();
    const listRef = ref(storage);
    list(listRef)
      .then((res) => {
        // Gets current display picture and displays it
        res.items.forEach((imageRef) => {
          const email = imageRef._location.path.slice(0, -4);
          if (email == this.user.data.email) {
            getDownloadURL(imageRef).then((url) => {
              this.displayPicture = url;
            });
          }
        });
      })
      .finally(() => {
        // Set default picture if no display picture
        if (!this.displayPicture) {
          this.displayPicture = defaultPic;
        }
        this.pageLoading = false;
      });
  },
  methods: {
    // Function to push the updated information into firestore and firebase storage
    async update() {
      const clientRef = doc(db, "client", this.user.data.email);
      this.isLoading = true;
      updateDoc(clientRef, {
        fullName: this.fullName,
        contactNo: this.contactNo,
        emergencyContactName: this.emergencyContactName,
        emergencyContactNo: this.emergencyContactNo,
      })
        .then((response) => {
          this.isLoading = false;
          // Feedback when profile is updated
          Toast.fire({
            icon: "success",
            title: "Profile Saved",
          });
        })
        .catch((err) => {
          console.log(err);
        });
    },
    // Read the file and display the picture immediately when uploaded
    onFileSelected(event) {
      let reader = new FileReader();
      reader.onload = (e) => {
        this.displayPicture = e.target.result;
      };
      reader.readAsDataURL(event.target.files[0]);
      this.profilePicture = event.target.files[0];
    },
    // Function to upload image to firebase storage
    onUpload() {
      // Register three observers:
      // 1. 'state_changed' observer, called any time the state changes
      // 2. Error observer, called on failure
      // 3. Completion observer, called on successful completion
      // Ensure that it only uploads the pic when a pic is selected
      if (this.profilePicture) {
        const storage = getStorage();
        const storageRef = ref(storage, `${this.user.data.email}.jpg`);
        const uploadTask = uploadBytesResumable(
          storageRef,
          this.profilePicture
        ); // will replace old pic if already there
        uploadTask.on(
          "state_changed",
          (snapshot) => {
            // Observe state change events such as progress, pause, and resume
            // Get task progress, including the number of bytes uploaded and the total number of bytes to be uploaded
            const progress =
              (snapshot.bytesTransferred / snapshot.totalBytes) * 100;
            console.log("Upload is " + progress + "% done");
            switch (snapshot.state) {
              case "paused":
                console.log("Upload is paused");
                break;
              case "running":
                console.log("Upload is running");
                break;
            }
          },
          (error) => {
            // Handle unsuccessful uploads
            console.log("Unsuccessful upload!");
          }
        );
      }
    },
  },
};
</script>

<style scoped>
.container {
  margin: auto;
}
.dp {
  width: auto;
  height: 200px;
  border-radius: 50%;
}

.edit-page {
  background-color: black;
  overflow-y: hidden;
  min-width: 100vw;
  min-height: 100vh;
}

.display-words {
  padding: 20px 10vw;
  margin: 0;
  justify-content: flex-start;
  font-family: Teko;
  background-color: black;
  color: white;
  font-size: 3rem;
  width: auto;
  flex: 2;
}

.display-words > div {
  border-bottom: 5px solid white;
}

.picture {
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden;
  position: relative;
}

.picture img {
  height: 200px;
  width: 200px;
  object-fit: cover;
  border-radius: 50%;
  border: 2px white solid;
}
.picture > label {
  width: auto;
  /* so that it can only be clicked in the circle */
  border-radius: 50%;
}

.picture .overlay {
  position: absolute;
  bottom: 0;
  background: rgb(0, 0, 0);
  background: rgba(0, 0, 0, 0.5); /* Black see-through */
  color: #f1f1f1;
  color: white;
  width: 200px;
  font-size: 20px;
  padding: 20px;
  text-align: center;
  font-family: "Source Sans Pro", sans-serif;
  cursor: pointer;
}

.picture .overlay:hover {
  color: #ed1f24;
}

.button-div {
  text-align: center;
}

.card-body {
  background-color: black;
  color: white;
}

button {
  width: 120px;
  height: 50px;
  border-radius: 25px; /* half of height */
  background-color: rgb(237, 12, 16);
  border: none;
  outline: none;
  cursor: pointer;
  margin: 5px;
  margin-top: 30px;
  font-size: 1.5rem;
  text-align: center;
  box-sizing: border;
  color: white;
  font-family: Teko;
}

button:hover {
  animation-name: pill-button-highlight;
  animation-duration: 0.15s;
  animation-fill-mode: forwards;
  box-sizing: border-box;
}

@keyframes pill-button-highlight {
  from {
    border: 0px white solid;
  }
  to {
    border: 2px white solid;
  }
}

#file-input {
  display: none;
}
</style>
