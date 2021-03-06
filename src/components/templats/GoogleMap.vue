<template>
  <div class="google-map-wrapper">
    <!-- googlemap -->
    <GmapMap
      :center="center"
      :zoom="15"
      map-type-id="roadmap"
      style="width: 100vw; height: 100vh"
      :options="gmapMapOptions"
    >
      <cluster>
        <gmap-custom-marker :key="index" v-for="(m, index) in markers" :marker="m.location">
          <!-- if the bike is rented by another user change maker color -->
          <!-- avaiable bike -->
          <img
            v-if="!m.renting && m.id !== 'user'"
            class="img"
            src="@/assets/img/icons/marker_red.svg"
            @click="toggleInfoWindow(m, index)"
          />
          <!-- another user uses the bike -->
          <img
            v-if="m.renting && m.renitng_user_email !== userInfo.email"
            class="img"
            src="@/assets/img/icons/marker_gray.svg"
            @click="toggleInfoWindow(m, index)"
          />
          <!-- the bike user using -->
          <img
            v-if="m.renting && m.renitng_user_email === userInfo.email"
            class="img"
            src="@/assets/img/icons/marker_white.svg"
            @click="toggleInfoWindow(m, index)"
          />

          <!-- user location -->
          <img
            v-if="m.id === 'user' && !userInfo.rentingBike && !m.renting"
            class="img"
            src="@/assets/img/icons/my_location.svg"
          />
        </gmap-custom-marker>
      </cluster>
      <!-- google map info window -->
      <GmapInfoWindow
        :position="infoWinPos"
        :opened="infoWinIsOpen"
        @closeclick="infoWinIsOpen = false"
        :options="infoWinOptions"
      >
        <!-- info findow content-->
        <!-- another user uses the bike -->
        <div
          class="info-window-style"
          v-if="bikeIsRenting && userInfo.email !== renitng_user_email"
        >
          another user renting bike
        </div>
        <!-- while user is renting a bike show anoher bike -->
        <div
          class="info-window-style"
          v-if="!bikeIsRenting && userInfo.rentingBike && userInfo.email !== renitng_user_email"
        >
          you can not rent another bike before returning the current bike
        </div>
        <!-- avaiable bike -->
        <div class="info-window-style" v-if="!bikeIsRenting && !userInfo.rentingBike">
          <b-icon class="bike-icon" variant="success" icon="bicycle" font-scale="5"></b-icon>
          <h5 v-html="chosenBikeName"></h5>
          <SaveButtons @click="rentBike">RENT BIKE?</SaveButtons>
        </div>
        <!-- while user is renting a bike show anoher bike -->
        <div
          class="info-window-style"
          v-if="bikeIsRenting && userInfo.rentingBike && userInfo.email === renitng_user_email"
        >
          <b-icon class="bike-icon" variant="success" icon="bicycle" font-scale="5"></b-icon>
          <h5 v-html="userInfo.bikeName"></h5>
          you are now renting this bike
          {{ nowData }}
          <SaveButtons :color="outline - success" @click="returnBike">RETURN BIKE</SaveButtons>
        </div>
      </GmapInfoWindow>
    </GmapMap>
    <!-- pop up window by renting and returning -->
    <div
      class="renting-bike-mini-window"
      :class="{ isClosed: isActive }"
      v-if="userInfo.rentingBike || (userInfo.rentingBike && $route.path === '/dashboard/rent')"
    >
      <PopUpWindow>
        <div slot="card-icon">
          <b-icon
            slot="card-icon"
            font-scale="3"
            class="pop-up-window-close-button"
            @click="togglePopUpWindow"
            icon="x"
          ></b-icon>
        </div>
        <div slot="text">
          you are now renting the bike
          <p></p>
          <b-icon class="bike-icon" variant="success" icon="bicycle" font-scale="5"></b-icon>
          <!-- here will come {{ rentingDuration}} to show the duration-->
          <h4 v-html="userInfo.bikeName"></h4>
        </div>
        <div slot="button">
          <SaveButtons @click="returnBike">RETURN BIKE</SaveButtons>
        </div>
      </PopUpWindow>
    </div>

    <!-- pop-up window after returning the bike -->
    <div class="rerutning-bike-mini-window" v-if="$route.path === '/dashboard/returned'">
      <PopUpWindow>
        <div slot="card-icon">
          <router-link to="/dashboard">
            <b-icon class="close-icon" font-scale="3" icon="x"></b-icon>
          </router-link>
        </div>
        <div slot="text">you have returned the bike!</div>
        <!-- here will come {{ rentingDuration}} to show the duration-->
      </PopUpWindow>
    </div>
  </div>
</template>

<script>
import firebase from "firebase";
import moment from "moment";
import db from "../firebaseInit";
import "firebase/database";
import "firebase/storage";
import PopUpWindow from "../atoms/PopUpWindow";
import GmapCustomMarker from "vue2-gmap-custom-marker";
import SaveButtons from "../atoms/SaveButtons.vue";
export default {
  name: "GoogleMap",
  components: {
    SaveButtons,
    GmapCustomMarker,
    PopUpWindow
  },

  data() {
    return {
      userInfo: { email: "", rentingBike: false, bikeId: "", startTime: "", bikeName: "" },
      markers: [],
      // torriger to close the renting-pop-up-window
      isActive: false,
      //TODO demo marker list. next the center will be user's location point
      center: { lat: 52.5298, lng: 13.40145 },
      gmapMapOptions: {
        mapTypeControl: false,
        streetViewControl: false,
        rotateControl: false,
        fullscreenControl: false,
        disableDefaultUi: false,
        clickableIcons: false
      },
      rentingDuration: "",
      infoWinIsOpen: false,
      infoWinPos: {
        lat: 0,
        lng: 0
      },
      chosenBikeId: "",
      chosenBikeName: "",
      bikeIsRenting: false,
      currentMarkerIdx: null,
      infoWinOptions: {
        pixelOffset: {
          width: 0,
          height: -35
        }
      }
    };
  },

  created() {
    // initialize marker's array
    this.markers.splice(0);
    //getting bicycle data from firebase
    db.collection("bicycle")
      .get()
      .then(querySnapshot => {
        querySnapshot.forEach(doc => {
          const data = {
            id: doc.id,
            bikeName: doc.data().bike_namne,
            location: { lat: doc.data().lat, lng: doc.data().lng },
            renting: doc.data().rented,
            renitng_user_email: doc.data().renitng_user_email
          };
          this.markers.push(data);
        });
      });
    // getting user and renting_list data
    const user = firebase.auth().currentUser;
    const RentingList = db
      .collection("renting_list")
      .where("user_email", "==", user.email)
      .get()
      .then(querySnapshot => {
        querySnapshot.forEach(doc => {
          if (doc.data().user_email === user.email) {
            const data = {
              email: user.email,
              rentingBike: true,
              bikeName: doc.data().bike_name,
              bikeId: doc.data().bicycle_id,
              startTime: doc.data().start_time,
              bikeLocationLat: doc.data().lat,
              bikeLocationLng: doc.data().lng
            };
            this.userInfo = data;
            // change the center to the renting bike marker
            this.center = { lat: doc.data().lat, lng: doc.data().lng };
            //if user email is not in the renting_list
            // making user info wwith rentingbike:false, user email
          } else if (!doc) {
            const data = {
              email: user.email,
              rentingBike: false
            };
            this.userInfo = data;
            console.log(this.userInfo);
          } else {
            console.log("no user info");
          }
        });
      })
      .catch(function(error) {
        console.log("Error getting renting_list: ", error);
      });
    const centerLocation = {
      id: "user",
      location: this.center
    };
    this.markers.push(centerLocation);
  },

  methods: {
    async logOut() {
      try {
        const user = await firebase.auth().signOut();
        alert(`you are logged out`);
        this.$router.replace({ name: "HomePage" });
      } catch (err) {
        console.log(err);
      }
    },
    // there is no reload in Vue-router
    reload(pathName) {
      window.location.reload({ path: pathName });
    },
    //FIXME to get a duration of renting time
    getDuration() {
      // let startTime = moment(this.userInfo.startTime).format("DD MMM YYYY hh:mm a");
      console.log(moment(this.userInfo.startTime).toNow());
      return (this.rentingDuration = moment(this.userInfo.startTime).toNow());
    },
    togglePopUpWindow() {
      this.isActive = !this.isActive;
    },
    // looking for identified id and not renting and change InfoWindowIsOpen to open infowindow
    toggleInfoWindow(clickedMarker, index) {
      this.infoWinPos = clickedMarker.location;
      this.bikeIsRenting = clickedMarker.renting;
      this.chosenBikeName = clickedMarker.bikeName;
      this.renitng_user_email = clickedMarker.renitng_user_email;
      this.chosenBikeId = clickedMarker.id;
      if (this.currentMarkerIdx === index) {
        this.infoWinIsOpen = !this.infoWinIsOpen;
      } else {
        this.currentMarkerIdx = index;
        this.infoWinIsOpen = true;
      }
    },
    //update the bike's rent infos rented:true retuned:false (defaut, renred:flase, returned:true)
    async updateRentedBikeInfo() {
      try {
        //getting current user
        const user = await firebase.auth().currentUser;
        const data = db.collection("bicycle").doc(this.chosenBikeId);
        await data
          .update({
            renitng_user_email: user.email,
            rented: true
          })
          .then(() => {
            this.reload("/dashboard");
            const changeRoute = this.$router.replace({ path: "/dashboard/rent" });
          });
      } catch (error) {
        console.log("error by ubdationg bike info:", error);
      }
    },
    // after returning the bike email will be removed and rented will be false in DB.
    async updateReturnedBikeInfo() {
      try {
        const data = db.collection("bicycle").doc(this.userInfo.bikeId);
        await data
          .update({
            rented: false,
            renitng_user_email: ""
          })
          .then(() => {
            this.initializeUserInfo();
            this.reload("/dashboard");
            this.$router.replace({ path: "/dashboard/returned" });
          });
      } catch (error) {
        console.log("error by ubdationg bike info:", error);
      }
    },
    // after returning bike all bike info of userinfo will be removed (initialized)
    initializeUserInfo() {
      const user = firebase.auth().currentUser;
      const data = {
        email: user.email,
        rentingBike: false,
        bikeName: "",
        bikeId: "",
        startTime: ""
      };
      this.userInfo = data;
    },
    async addRentingBikeList() {
      try {
        //getting current user
        const user = await firebase.auth().currentUser;
        //chosen bike's path
        const bike = db.doc(`id/${this.chosenBikeId}`);
        // creating a data to add 'renting_list'
        const data = {
          user_email: user.email,
          bicycle_id: bike.id,
          bike_name: this.chosenBikeName,
          lat: this.infoWinPos.lat,
          lng: this.infoWinPos.lng,
          start_time: firebase.firestore.Timestamp.now()
        };
        // add a new data to the 'renting_list' in DB
        const list = await db.collection("renting_list").add(data);
      } catch (error) {
        console.error("Error by adding renting_bike list: ", error);
      }
    },
    async deleteRentingBikeList() {
      const user = await firebase.auth().currentUser;
      const RentingList = await db
        .collection("renting_list")
        .where("user_email", "==", user.email)
        .get()
        .then(querySnapshot => {
          querySnapshot.forEach(doc => {
            doc.ref.delete();
          });
        })
        .catch(function(error) {
          console.log("Error delete bilinlist: ", error);
        });
    },
    // to get rent duration and create history page keep the rent-return data.
    async addHistory() {
      try {
        //getting current user, bike, and rentinglist
        const user = firebase.auth().currentUser;
        const bike = db.doc(`id/${this.userInfo.bikeId}`);
        //getting renting_list
        const rentingList = db
          .collection("renting_list")
          .where("bicycle_id", "==", this.userInfo.bikeId)
          .get()
          .then(querySnapshot => {
            querySnapshot.forEach(rentingList => {
              // creating a data for adding 'history'
              const data = {
                user_email: user.email,
                bicycle_id: bike.id,
                bike_name: this.userInfo.bikeName,
                lat: this.userInfo.bikeLocationLat,
                lng: this.userInfo.bikeLocationLng,
                start_time: this.userInfo.startTime,
                end_time: firebase.firestore.Timestamp.now()
              };
              // add a new data to the 'history' in DB
              const list = db.collection("history").add(data);
            });
          })
          .catch(err => {
            console.log(err);
          });
      } catch (error) {
        console.error("Error by adding history: ", error);
      }
    },
    rentBike() {
      this.addRentingBikeList();
      this.updateRentedBikeInfo();
    },
    returnBike() {
      this.addHistory();
      this.deleteRentingBikeList();
      this.updateReturnedBikeInfo();
    }
  }
};
</script>
<style lang="scss" scoped>
@import "../../assets/scss/components/organisms/_google-map";
</style>
