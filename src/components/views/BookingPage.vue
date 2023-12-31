<template>
  <div class="booking-page">
    <Navbar />
    <div class="booking-page-header">
      <div>current bookings</div>
      <!-- the buttons to be displayed at the top for booking management -->
      <div class="button-container">
        <button class="pill-button" @click="showCancelModal()">Cancel Booking</button>
        <button class="pill-button" @click="showBookModal()">Make Booking</button>
        <button class="pill-button" @click="showPreviousBookingsModal()">Previous Bookings</button>
      </div>
    </div>
    <!-- Cancel / Book / Previous bookings popups to be displayed when clicked accordingly -->
    <div class="popup">
      <CancelModal
        v-show="cancelModal"
        @removeBookings="updateBookModal"
        @setNewClientBookings="updateClientBookings"
        @close-modal="cancelModal = false"
        :clientBookings="clientBookings"
        :newBookings="newBookingsFromBookModal"
      />
    </div>
    <div class="popup">
      <BookModal
        v-show="bookModal"
        @addBookings="updateCancelModal"
        @close-modal="bookModal = false"
        :allBookedSessions="dateToBookMappings"
        :cancelledBookings="cancelledBookingsFromCancelModal"
      />
    </div>
    <div class="popup">
      <PreviousBookingsModal
        v-show="previousBookingsModal"
        @close-modal="previousBookingsModal = false"
        :previousBookings="previousClientBookings"
      />
    </div>
    <!-- calendar component to be rendered for the specific client -->
    <div
      style="
        height: 800px;
        width: 85%;
        margin: auto;
        padding: 50px;
        text-align: center;
      "
    >
      <vue-cal
        :xsmall="this.viewPortWidth <= 1200"
        :key="key"
        class="calendar"
        active-view="month"
        :disable-views="['years', 'year', 'day']"
        :time-from="9 * 60"
        :time-to="20 * 60"
        :time-step="60"
        :events="events"
        :on-event-click="onEventClick"
        :editable-events="{
          title: false,
          drag: false,
          resize: false,
          delete: false,
          create: false,
        }"
        :snap-to-time="60"
        :timeCellHeight="70"
        :min-date="minDate"
        :max-date="maxDate"
        events-count-on-year-view
      />
      <!-- details of the booking event when the event is clicked on the calendar -->
      <div class="popup">
        <CalendarDetailModal
          v-show="showModal"
          :eventTitle="eventTitle"
          :eventDate="eventDate"
          :eventStart="eventStart"
          :eventEnd="eventEnd"
          :eventExerciseType="eventExerciseType"
          :eventTrainer="eventTrainer"
          @close-modal="showModal = false"
        />
      </div>
    </div>
  </div>
</template>

<script>
import VueCal from "vue-cal";
import BookModal from "../client/BookModal.vue";
import CancelModal from "../client/CancelModal.vue";
import CalendarDetailModal from "../client/CalendarDetailModal.vue";
import PreviousBookingsModal from "../client/PreviousBookingsModal.vue";
import { collection, getDocs, query, where } from "firebase/firestore";
import { db, auth } from "../../firebase.js";
import { mapGetters } from "vuex";
import "vue-cal/dist/vuecal.css";

// creating initial dates for date management when doing bookings
let minDate = new Date();
let maxDate = new Date();
minDate.setDate(minDate.getDate() + 1);
maxDate.setMonth(minDate.getMonth() + 3);

export default {
  name: "BookingPage",
  data() {
    return {
      // date related to calendar
      viewPortWidth: null,
      key: 0,
      clientTrainer: "",
      clientBookings: [],
      previousClientBookings: [],
      newBookingsFromBookModal: [],
      cancelledBookingsFromCancelModal: [],
      allBookings: [],
      dateToBookMappings: {},
      cancelModal: false,
      bookModal: false,
      previousBookingsModal: false,
      minDate: minDate,
      maxDate: maxDate,
      events: [],
      // data related to popup
      showModal: false,
      eventTitle: null,
      eventDate: null,
      eventStart: null,
      eventEnd: null,
      eventExerciseType: null,
      eventTrainer: null,
    };
  },
  components: {
    VueCal,
    BookModal,
    CancelModal,
    CalendarDetailModal,
    PreviousBookingsModal,
  },
  computed: {
    ...mapGetters(["user"]),
  },
  mounted() {
    auth.onAuthStateChanged((user) => {
      this.$store.dispatch("fetchUser", user);
    });
    this.viewPortWidth = window.innerWidth;
  },
  // first fetch of information
  async created() {
    // a container to store all bookings of this client
    let clientBookings = [];
    // a container to store all bookings of this client's trainer
    let allClientBookings = [];
    // track query bookings from firebase
    let bookingsFromFirebase = [];
    // track all day -> session slots
    let dateToBookMappings = {};
    // track previous bookings
    let previousClientBookings = [];
    // track the date today
    let today = new Date();

    // retrieving this client's bookings
    const clientRef = collection(db, "client");
    const thisClientQuery = query(
      clientRef,
      where("email", "==", this.user.data.email)
    );
    const clientQuerySnapshot = await getDocs(thisClientQuery);
    clientQuerySnapshot.forEach((doc) => {
      // get the bookings field from this client
      bookingsFromFirebase = doc.data().bookings;
      // get the trainer email to get all client emails
      this.clientTrainer = doc.data().trainerEmail;
    });
    // after getting all bookings of this client, modify it to be used by cancel modal
    bookingsFromFirebase.forEach((booking) => {
      let title = booking.title;
      let focus = booking.focus;
      let from = booking.from.toDate();
      let to = booking.from.toDate();

      // only bookings that are made after today will be displayed on the calendar
      if (today < from) {
        clientBookings.push({ title, focus, from, to });
      } else {
        // otherwise, it will be shown in the "Previous Bookings" modal
        previousClientBookings.push({ title, focus, from, to });
      }
    });
    this.clientBookings = clientBookings;
    this.previousClientBookings = previousClientBookings;

    // getting all client emails associated with the current trainer
    const trainerRef = collection(db, "trainer");
    const trainerQuery = query(
      trainerRef,
      where("email", "==", this.clientTrainer)
    );

    // a storage for all client emails
    let allClients = [];
    const trainerQuerySnapshot = await getDocs(trainerQuery);
    trainerQuerySnapshot.forEach((doc) => {
      allClients = doc.data().ClientsId;
      this.eventTrainer = doc.data().fullName;
    });

    // getting all bookings for all clients
    const allClientQuery = query(clientRef, where("email", "in", allClients));
    const allClientQuerySnapshot = await getDocs(allClientQuery);
    allClientQuerySnapshot.forEach((client) => {
      client.data().bookings.forEach((doc) => {
        // each iteration gets one bookings from all the other clients
        let start = doc.from.toDate();
        let end = doc.to.toDate();
        let title = doc.title;
        let focus = doc.focus;
        let obj = { start, end, title, focus };

        // checking for date
        if (today < start) {
          // setting the class attribute for each booking
          if (client.data().email != this.user.data.email) {
            obj["class"] = "otherClient unclickable";
            obj["title"] = "Unavailable";
            allClientBookings.push(obj);
          } else {
            obj["class"] = "thisClient unclickable";
            allClientBookings.push(obj);
          }

          // data pivoting on each booking
          let month = start.getMonth();
          let day = start.getDate();
          let startHour = start.getHours();

          if (dateToBookMappings.hasOwnProperty(`${day}, ${month}`)) {
            dateToBookMappings[`${day}, ${month}`].push(startHour);
          } else {
            dateToBookMappings[`${day}, ${month}`] = [startHour];
          }
        }
      });
      // this is the initial set of bookings when the booking component is rendered
      this.dateToBookMappings = dateToBookMappings;
      this.events = allClientBookings;
    });
  },
  watch: {
    // if new bookings are made, display new information on the calendar
    newBookingsFromBookModal(newBookings) {
      newBookings.forEach((x) => {
        this.events.push({
          start: x.from,
          end: x.to,
          title: x.title,
          focus: x.focus,
          class: "thisClient unclickable",
        });
      });
      this.key++;
    },
    // if bookings are cancelled, remove these events from the calendar
    cancelledBookingsFromCancelModal(cancelledBookings) {
      cancelledBookings.forEach((x) => {
        this.events = this.events.filter((y) => {
          return x["from"].getTime() !== y["start"].getTime();
        });
      });
      this.key++;
    },
  },
  methods: {
    // displays the cancel pop up
    showCancelModal() {
      this.cancelModal = true;
    },
    // displays the booking pop up
    showBookModal() {
      this.bookModal = true;
    },
    // displayed the previous bookings pop up
    showPreviousBookingsModal() {
      this.previousBookingsModal = true;
    },
    // pass the new bookings over to cancel modal to be able to cancel more bookings reactively
    updateCancelModal(newBookings) {
      this.newBookingsFromBookModal = newBookings;
    },
    // pass the cancelled bookings over to book modal to be able to make more bookings reactively
    updateBookModal(cancelledBookings) {
      this.cancelledBookingsFromCancelModal = cancelledBookings;
    },
    // pass the client bookings over to cancel modal to be able to make more bookings reactively
    updateClientBookings(newClientBookings) {
      this.clientBookings = newClientBookings;
    },
    // function required for display the pop up
    onEventClick(event, e) {
      this.eventTitle = event.title;
      // Convert to Date object for now because not connected to FS yet
      this.eventDate = new Date(event.start).toLocaleDateString();
      this.eventStart = new Date(event.start).toLocaleTimeString();
      this.eventEnd = new Date(event.end).toLocaleTimeString();
      this.eventExerciseType = event.focus;
      this.showModal = true;

      e.stopPropagation();
    },
  },
};
</script>

<style scoped>
@import url("https://fonts.googleapis.com/css2?family=Hanken+Grotesk&family=Teko:wght@500;600&display=swap");
.nav {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  background-color: black;
  color: #fff;
  height: 200px;
  font-family: "Source Sans Pro", "sans-serif";
  font-size: larger;
  border-bottom: 5px solid #ed1f24;
  box-sizing: border-box;
  min-width: 100vw;
}

.booking-page-header {
  color: white;
  border-bottom: 5px white solid;
  font-size: 3rem;
  text-transform: uppercase;
  width: 80%;
  margin: auto;
  padding-top: 20px;
  display: flex;
  justify-content: space-between;
  flex-wrap: wrap;
}
.booking-page {
  background-color: black;
  min-height: 100vh;
  font-family: Teko;
  min-width: 100vh;
}

.calendar {
  background-color: white;
  font-family: "Source Sans Pro", "sans-serif";
  font-size: 1.2rem;
}

.pill-button:hover {
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
.pill-button {
  width: 200px;
  height: 50px;
  border-radius: 25px; /* half of height */
  background-color: rgb(237, 31, 36);
  border: none;
  outline: none;
  cursor: pointer;
  margin: 5px;
  font-size: 1.5rem;
  text-align: center;
  box-sizing: border-box;
  color: white;
}

.button-container {
  text-align: center;
  display: flex;
  justify-content: left;
  flex-wrap: wrap;
}

@media screen and (max-width: 800px) {
  .booking-page {
    padding-bottom: calc(600px - 10vh);
  }
  .content-container {
    max-height: 80vh;
  }
}

@media screen and (max-width: 550px) {
  .booking-page {
    padding-bottom: calc(850px - 10vh);
  }
}

@media screen and (max-width: 400px) {
  .booking-page {
    padding-bottom: calc(1200px - 10vh);
  }
}
</style>

<style>
.vuecal__event {
  background-color: rgba(169, 169, 169, 0.7);
  border: solid rgba(0, 0, 0, 0.3);
  border-width: 0 0 2px 0;
  box-sizing: border-box;
  padding: 5px;
  cursor: pointer;
  display: flex;
  justify-content: center;
  flex-wrap: wrap;
  align-items: center;
}
/* .unclickable {
  cursor: pointer;
} */

.vuecal__event-title {
  font-size: 0.9em;
  font-weight: bold;
}

.vuecal__event-time {
  font-size: 0.7em;
  display: none;
}

@media screen and (max-width: 900px) {
  .vuecal__event-title {
    font-size: 50%;
    font-weight: bold;
  }
}

.vuecal__cell--has-events {
  /* Add background color to cells with events */
  background-color: #fffacd;
}

/* Styles for Min and Max Range vue-cal */
.vuecal__cell--disabled {
  text-decoration: line-through;
}

.vuecal__cell--after-max {
  color: #a8a2a2;
}

.otherClient {
  background: #ed1f24;
  color: #fff;
  pointer-events: none;
}

.thisClient {
  background: #000;
  color: #fff;
  border: 1px solid transparent;
}

.thisClient:hover {
  border: 1px white solid;
}
/* Other styles */
.vuecal__menu,
.vuecal__cell-events-count {
  background-color: #ed1f24;
}

.vuecal__title-bar {
  background-color: #ff8082;
}

.vuecal__cell--today,
.vuecal__cell--current {
  background-color: #ffd6d7;
}

.vuecal:not(.vuecal--day-view) .vuecal__cell--selected {
  background-color: #ff8082;
}

.vuecal__cell--selected:before {
  border-color: black;
}

/* Cells and buttons get highlighted when an event is dragged over it. */
.vuecal__cell--highlighted:not(.vuecal__cell--has-splits),
.vuecal__cell-split--highlighted {
  background-color: rgba(195, 255, 225, 0.5);
}

.vuecal__arrow.vuecal__arrow--highlighted,
.vuecal__view-btn.vuecal__view-btn--highlighted {
  background-color: rgba(136, 236, 191, 0.25);
}
</style>
