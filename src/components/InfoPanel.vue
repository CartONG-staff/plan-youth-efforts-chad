<template>
  <div
    class="InfoPanel"
    v-if="$store.state.submissions != null">
    <InfoPanelPlace v-show="$store.state.selectedPlaceData != null" />
    <div class="InfoPanel__filtersActionsCtn" v-if="$store.state.selectedPlaceData === null">
      <input type="text" v-model="search" class="InfoPanel__search" :placeholder="words[lang].infoPanel.label.SearchPlaceholder"/>
      <button class="InfoPanel__toggleFiltersBtn Button" @click="filtersShown = true" v-if="!filtersShown">
        <img :src="require('@/assets/img/icons/slider.svg')"/>
        {{ words[lang].infoPanel.label.ShowFilters }}
      </button>
      <button class="InfoPanel__toggleFiltersBtn Button" @click="filtersShown = false" v-if="filtersShown">
        <img :src="require('@/assets/img/icons/close.svg')"/>
        {{ words[lang].infoPanel.label.HideFilters }}
      </button>
    </div>
    <div class="InfoPanel__filtersCtn" v-if="filtersShown && $store.state.selectedPlaceData === null">
      <div class="InfoPanel__filters InfoPanel__filters--step1">
        <h3 v-html="words[lang].infoPanel.label.Step1Label"></h3>
        <CategoryPicker/>
      </div>
      <div class="InfoPanel__filters InfoPanel__filters--step2" v-if="$store.state.selectedCategory !== null" :data-category="$store.state.selectedCategory">
        <h3 v-html="words[lang].infoPanel.label.Step2Label + ' (' + activeFilterCount + ' ' + words[lang].infoPanel.label.Step2LabelActiveFilters + ')'"></h3>
        <div :class="'InfoPanel__filters--step2--list InfoPanel__filters--step2--' + $store.state.selectedCategory">
          <div class="Filter" :data-type="filters[$store.state.selectedCategory][key].type" v-for="key in Object.keys(filters[$store.state.selectedCategory])" :key="key" :data-enabled="filters[$store.state.selectedCategory][key].enabled ? 'true' : 'false'">
            <button
              @click="filters[$store.state.selectedCategory][key].enabled = !filters[$store.state.selectedCategory][key].enabled"
              class="Filter__enableFilterBtn">
              {{ filters[$store.state.selectedCategory][key].enabled ? "-" : "+" }}
            </button>
            <input
              type="checkbox"
              :id="key"
              :name="key"
              v-model="filters[$store.state.selectedCategory][key].data"
              :disabled="!filters[$store.state.selectedCategory][key].enabled"
              v-if="filters[$store.state.selectedCategory][key].type === 'switch'"/>
            <label :for="key">{{ words[lang].infoPanel.filters[$store.state.selectedCategory][key] }}</label>
            <select v-if="filters[$store.state.selectedCategory][key].type === 'list'" v-model="filters[$store.state.selectedCategory][key].data">
              <option :value="null">{{ words[lang].infoPanel.label.SelectAnOption }}</option>
              <option v-for="(id) in $store.state.lists[filters[$store.state.selectedCategory][key].listItems]" :value="id" :key="id">
                {{ words[lang].infoPanel.lists[filters[$store.state.selectedCategory][key].listItems][id] }}
              </option>
            </select>
          </div>
        </div>
      </div>
    </div>
    <div class="InfoPanel__resultsCtn" v-if="$store.state.selectedPlaceData === null">
      <div class="InfoPanel__resultsHeader">
        <h2>{{ words[lang].infoPanel.label.StructuresList }}</h2>
        <button class="InfoPanel__exportBtn Button" @click="exportResults()">
          <img :src="require('@/assets/img/icons/download.svg')"/>
          {{ words[lang].infoPanel.label.ExportData }}
        </button>
      </div>
      <div
        v-for="(category, key) in ['health', 'education', 'youth-organizations'].filter(cat => cat === $store.state.selectedCategory || $store.state.selectedCategory === null)"
        :key="key"
        class="InfoPanel__categCtn">
        <div class="InfoPanel__categHeader" :style="'border-color:'+$store.state.categories[category].color">
          <div class="InfoPanel__categHeaderLeft">
            <img
              :src="require('@/assets/img/icons/' + category +'.svg')"
              :style="'background-color:'+$store.state.categories[category].color"/>
            <h2>{{ words[lang].category[category] }}</h2>
          </div>
          <div class="InfoPanel__categHeaderRight">
            <span>
              <b>{{ placesByCategoryFiltered(category).length }}</b>
              {{ words[lang].infoPanel.label.PlacesCountCategOutOff }} {{ placesByCategory(category).length  }} {{ words[lang].infoPanel.label.PlacesCountCategResults }}
            </span>
          </div>
        </div>
        <div
          class="InfoPanel__place"
          v-for="(submission) in limitResults(category)"
          :key="submission.id" :data-id="submission.id"
          @click="goToPlace(submission.id)"
          >{{ submission.label }}
          <img :src="require('@/assets/img/icons/arrow.svg')"/></div>
        <div
          class="InfoPanel__showMore"
          @click="categoryListShow[category] = !categoryListShow[category]"
          v-if="placesByCategoryFiltered(category).length > 5"
          v-html="showMore(category)"
          />
      </div>
    </div>
  </div>
</template>

<script>

import { words } from '@/constants/lang'
// import MapboxGeocoder from '@mapbox/mapbox-gl-geocoder';
import '@mapbox/mapbox-gl-geocoder/lib/mapbox-gl-geocoder.css';
import CategoryPicker from './CategoryPicker.vue'
import InfoPanelPlace from './InfoPanelPlace.vue'

export default {
  name: 'InfoPanel',
  components: {
    CategoryPicker,
    InfoPanelPlace
  },
  created () {
    this.$root.$refs.InfoPanel = this;
  },
  computed: {
    lang: function () {
      return this.$store.state.lang
    },
    filterId: function (filterKey) {
      return filterKey
    },
    activeFilterCount: function () {
      return Object.keys(this.filters[this.$store.state.selectedCategory]).filter(key =>
        this.filters[this.$store.state.selectedCategory][key].enabled === true
      ).length
    }
  },
  data: () => ({
    search: '',
    words,
    filtersShown: false,
    limit: 5,
    categoryListShow: {
      'health': false,
      'education': false,
      'youth-organizations': false
    },
    filters: {
      'education': {
        "groupConsent/groupEducation/groupEducStatus/educOwnership": { type: "list", listItems: "ownership", data: null, enabled: false },
        "groupConsent/groupEducation/groupEducStatus/operational": { type: "switch", data: true, enabled: false },
        "groupConsent/groupEducation/groupEducStatus/educDisability": { type: "switch", data: true, enabled: false },
        "groupConsent/groupEducation/groupEducFunctionning/price": { type: "switch", data: true, enabled: false },
        "groupConsent/groupEducation/groupEducCurriculum/peacebuilding": { type: "switch", data: true, enabled: false },
        "groupConsent/groupEducation/groupEducReporting/reportingSystem": { type: "switch", data: true, enabled: false },
        "groupConsent/groupEducation/groupEducSecurity/shelter": { type: "switch", data: true, enabled: false },
      },
      'health': {
        "groupConsent/groupHealth/groupHltFunctinonning/hltDisability": { type: "switch", data: true, enabled: false },
        "groupConsent/groupHealth/groupHltService/youthService": { type: "switch", data: true, enabled: false },
        "groupConsent/groupHealth/groupHltService/serviceWomen": { type: "switch", data: true, enabled: false },
        "groupConsent/groupHealth/groupHltSrh/sexualViolenceResponse": { type: "switch", data: true, enabled: false },
      },
      'youth-organizations': {
        "groupConsent/groupParticipation/groupParticipationRegistration/registration": { type: "switch", data: true, enabled: false },
        "groupConsent/groupParticipation/groupParticipationActivity/organizationActivitiy": { type: "list", listItems: "adminLevels", data: null, enabled: false },
        "groupConsent/groupParticipation/groupParticipationActivity/activities": { type: "list", listItems: "activities", data: null, enabled: false }
      }
    }
  }),
  watch: {
    '$store.state.selectedPlaceData': function() {
      if (this.$store.state.selectedPlaceData != null)
        this.$root.$refs.InfoPanelPlace.showImage()
    },
    '$store.state.selectedCategory': function() {
      let showFullLists = !(this.$store.state.selectedCategory === null)
      Object.keys(this.categoryListShow).forEach((key) => {
        this.categoryListShow[key] = showFullLists
      });
    },
    // Check if filters data has changed based on a deep watch, if yes update icons of the map
    filters: {
      handler() { this.updateIcons() },
      deep: true
    },
    search: function () {
      this.updateIcons()
    }
  },
  methods: {
    updateIcons: function () {
        if (this.$store.state.map != null) this.$root.$refs.Map.updateIcons()
    },
    showMore: function(category) {
      return this.categoryListShow[category] ? words[this.lang].infoPanel.label.ShowLessResults : words[this.lang].infoPanel.label.Show + " <b>" + (this.placesByCategoryFiltered(category).length - this.limit) + "</b> " +  words[this.lang].infoPanel.label.MoreResults;
    },
    exportResults: function() {
      let categories = Object.keys(this.$store.state.categories)
      if (this.$store.state.selectedCategory !== null) categories = [this.$store.state.selectedCategory]
      categories.forEach((category) => {
        let flattenResults = [];
        let results = this.placesByCategoryFiltered(category)
        results.forEach((result) => {
          let flattenResult = this.flattenObject(result)
          flattenResult['category'] = category
          flattenResults.push(flattenResult)
        })

        let link = document.createElement('a')
        link.id = 'download-csv'
        link.setAttribute('href', 'data:text/plain;charset=utf-8,%EF%BB%BF' + encodeURIComponent(this.convertToCSV(flattenResults)));
        link.setAttribute('download', this.getFilename(category));
        document.body.appendChild(link)
        document.querySelector('#' + link.id).click()
        link.remove()
      })
    },
    getFilename: function (category) {
      return  category + "_" + new Date().toISOString().slice(0, 10) + '.csv'
    },
    convertToCSV: function (results) {
      const labels = Object.keys(results[0])

      let bigArray = [labels.join(';')]
      results.forEach(result => { 
        let array = []
        labels.forEach((label) => {
          let res = typeof result[label] === 'string' ? result[label].replace(/;/g, ',') : result[label]
          array.push(res || "-")
        })
        bigArray.push(array.join(';').replace(/\n/g, ''))
      })
      return bigArray.join('\n')
    },
    flattenObject: function (obj) {
        let flattenKeys = {};
        for (let i in obj) {
            if (!obj.hasOwnProperty(i)) continue;
            if ((typeof obj[i]) == 'object') {
                // flattenKeys[i] = obj[i];
                let flatObject = this.flattenObject(obj[i]);
                for (let j in flatObject) {
                    if (!flatObject.hasOwnProperty(j)) continue;
                    flattenKeys[i + '.' + j] = flatObject[j];
                }
            } else {
                flattenKeys[i] = obj[i];
            }
        }
        return flattenKeys;
    },
    placesByCategoryFiltered: function(category) {
      let results = this.$store.state.submissions.filter(submission => submission.icon == category && submission.label.normalize('NFD').replace(/[\u0300-\u036f]/g, "").toLowerCase().includes(this.search.toLowerCase().normalize('NFD').replace(/[\u0300-\u036f]/g, "")))
      console.log(results)
      let resultsAfterFilters = results.filter(submission => {
        let activeFilters = Object.keys(this.filters[category]).filter(filter => this.filters[category][filter].enabled === true)
        return activeFilters.every(key => {
          if (this.filters[category][key].type === "switch") return (this.filters[category][key].data ? "yes" : "no") === submission.data[key]
          if (this.filters[category][key].type === "list") {
            if (this.filters[category][key].data === null) return true;
            return submission.data[key].split(' ').includes(this.filters[category][key].data)
          }
          return this.filters[category][key].data === submission.data[key]
        });
      })
      return resultsAfterFilters;
    },
    placesIdByCategoryFiltered: function () {
      let idArray = []
      Object.keys(this.$store.state.categories).forEach((category) => {
        this.placesByCategoryFiltered(category).map((place) => idArray.push(place.id))
      })
      return idArray
    },
    placesByCategory: function(category) {
      return this.$store.state.submissions.filter(submission => submission.icon == category)
    },
    limitResults: function(category) {
      let results = this.placesByCategoryFiltered(category)
      return this.categoryListShow[category] ? results : results.slice(0, this.limit);
    },

    getSubmissionById: function(id) {
      return this.$store.state.submissions.filter(submission => submission.id == id)
    },
    goToPlace: function(id) {
      let result = this.getSubmissionById(id)
      this.$store.commit("updateSelectedPlaceData", result[0]);
    }
  }
}
</script>

<style lang="less">
@import "../assets/less/var";
.InfoPanel {
  position: fixed;
  width: @dim-info-panel;
  max-width: 100%;
  background: rgba(240,240,240,0.9);
  height: 100vh;
  top: 0;
  right: 0;
  z-index: 10;
  overflow-y: auto;
  overflow-x: hidden;
  box-shadow: -5px 0 20px -5px rgba(0,0,0,0.3);
  display: flex;
  flex-flow: column nowrap;

  > * {
    padding-left: 1rem;
    padding-right: 1rem;
  }

  .InfoPanel__filters--step2--list {
    display: flex;
    flex-flow: column nowrap;
  }

  .Filter {
    display: flex;
    flex-flow: row nowrap;
    justify-content: space-between;
    position: relative;
    margin-bottom: 1rem;

    &[data-enabled="false"] > *:not(.Filter__enableFilterBtn) { filter: grayscale(1) !important; opacity: 0.45; }
    &[data-enabled="true"] > *:not(.Filter__enableFilterBtn) { filter: grayscale(0);}

    &[data-type="switch"][data-enabled="false"] label {
      &::after, &::before {
        content: initial;
      }
    }

    &[data-type="list"][data-enabled="false"] select {
      display: none;
    }

    &[data-type="list"] select {
      padding: 0.5rem;
      border-radius: 4px;
      border: solid 1.5px #aaa;
      margin-top: 0.5rem;
      cursor: pointer;
      appearance: none;
      -moz-appearance: none;
      -webkit-appearance: none;
      background: url("../assets/img/icons/expand.svg") no-repeat calc(100% - .5rem) #fff;

      /* For IE10 */
      &::-ms-expand {
        display: none;
      }

      option {
        padding: 0.5rem;
      }
    }

    &[data-type="list"][data-enabled="false"] select {
      display: none;
    }

    // To show filters at the end
    // &[data-enabled="true"]:last-child {
    //   padding-bottom: 1rem;
    // }
    // &[data-enabled="true"] {
    //   order: -1;
    // }

    .Filter__enableFilterBtn {
      border: none;
      width: 15px;
      margin-right: 0.5rem;
      position: absolute;
      background: grey;
      display: inline-flex;
      justify-content: center;
      align-items: center;
      height: 14px;
      max-width: 14px;
      border-radius: 2px;
      color: white;
      cursor: pointer;
      font-size: 0.65rem;
    }

    label {
      font-size: 0.85rem;
      cursor: pointer;
      margin-left: 1.5rem;
      width: 100%;
    }

    &[data-type="list"] {
      display: flex;
      flex-flow: column nowrap;
    }

    &[data-type="switch"] {

      label {
        @space-switch:  3rem;
        margin-right: @space-switch;

        &::after {
          content: "";
          width: 31px;
          height: 12px;
          border-radius: 7px;
          background: #c4c4c4;
          position: absolute;
          right: calc(5px - @space-switch);
          cursor: pointer;
          top: 0;
        }

        &::before {
          content: "";
          width: 20px;
          height: 20px;
          position: absolute;
          border-radius: 50%;
          background: #8F8F8F;
          box-shadow: 0 5px 10px -2px rgba(0,0,0,0.2);
          transition: all 0.2s ease-in;
          right: calc(8px - @space-switch);
          top: -4px;
          transform: translateX(-14px);
          z-index: 1;
          cursor: pointer;
        }
      }
      input[type='checkbox'] {
        opacity: 0;
        position: absolute;
        top: -100vh;

        &:checked ~ label {
          &::before {
            transform: translateX(8px);
          }
        }
      }
    }

  }
  .InfoPanel__filters {
    &[data-category='education'] {
      input[type='checkbox']:checked ~ label::before, .Filter[data-enabled="true"] .Filter__enableFilterBtn { background: @color-education; }
    }
    &[data-category='health'] {
      input[type='checkbox']:checked ~ label::before, .Filter[data-enabled="true"] .Filter__enableFilterBtn  { background: @color-health; }
    }
    &[data-category='youth-organizations'] {
      input[type='checkbox']:checked ~ label::before, .Filter[data-enabled="true"] .Filter__enableFilterBtn { background: @color-youth-organizations; }
    }
  }

  .InfoPanel__filtersCtn {
    h3 {
      font-size: .9rem;
      font-weight: 100;
    }
  }

  .InfoPanel__filtersActionsCtn {

    display: flex;
    flex-flow: row nowrap;
    justify-content: space-between;
    align-items: center;
    padding: 1rem;

    .InfoPanel__search {
      padding: .7rem 1rem .7rem 2.75rem;
      width: 150px;
      font-size: .9rem;
      border: 1.5px solid #aaa;
      box-sizing: border-box;
      border-radius: 4px;
      display: flex;
      flex: 1 0 auto;
      justify-content: center;
      align-items: center;
      background: #fff url(../assets/img/icons/search.svg) no-repeat 1rem center;
      background-size: auto 40%;

      &::placeholder { color: 727272; }

      img {
        width: 23px;
        margin-right: 8px;
      }
    }
  }

  .Button {
    background: #FFFFFF;
    border: 1.5px solid #000000;
    box-sizing: border-box;
    border-radius: 4px;
    font-size: .9rem;
    margin-left: 1rem;
    font-weight: bold;
    max-height: 44px;
    padding: .7rem 1rem .7rem 1rem;
    cursor: pointer;
    display: flex;
    flex-flow: row nowrap;
    justify-content: space-between;
    align-items: center;
    transition: all .25s linear;

    img {
      width: 18px;
      margin-right: 5px;
    }

    &:hover {
      background: #333;
      color: white;
      img {
        filter: invert(1);
      }
    }
  }
  .InfoPanel__categCtn {
    margin-bottom: 1rem;

    .InfoPanel__categHeader {
      display: inline-flex;
      flex-flow: row nowrap;
      justify-content: space-between;
      align-items: center;
      border: solid 2px;
      box-sizing: border-box;
      border-radius: 0 4px 4px 0;
      width: 100%;
      background: #F9F9F9;

      .InfoPanel__categHeaderLeft {
        display: inline-flex;
        flex-flow: row nowrap;
        align-items: center;

        img {
          width: 20px;
          height: 24px;
          padding: 6px;
          margin-right: 10px;
        }

        h2 {
          font-family: @font-primary;
          font-weight: 100;
          margin: 0;
          font-size: 1.2em;
        }
      }

      .InfoPanel__categHeaderRight {
        display: inline-flex;
        flex-flow: row nowrap;
        align-items: center;

        span {
          font-family: @font-secondary;
          font-size: 0.7rem;
          color: #808080;
          margin-right: .5rem;
        }
      }
    }
  }
  .InfoPanel__resultsCtn {
    border-top: solid 1px #9E9E9E;
    padding-top: 1rem;
    background: white;
    flex: 1 0 auto;

    .InfoPanel__resultsHeader {
      display: flex;
      flex-flow: row nowrap;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 0.5rem;

      h2 {
        font-family: @font-primary;
        font-size: 1.2rem;
        font-weight: 100;
        margin: 0;
      }

      .InfoPanel__exportBtn {
        img { width: 24px; }
      }
    }

    .InfoPanel__showMore {
      font-size: 0.7rem;
      text-align: center;
      padding: 0.75rem 1.5rem 0.5rem 0.5rem;
      color: #9E9E9E;
      cursor: pointer;
      position: sticky;
      background: linear-gradient(0deg, rgba(255,255,255,0.9) 80%, transparent);
      bottom: 0;
    }

    .InfoPanel__place {
      cursor: pointer;
      font-size: 0.8em;
      padding: .5rem 1rem .5rem .5rem;
      margin-right: 1.5rem;
      display: flex;
      flex-flow: row nowrap;
      justify-content: space-between;
      align-items: center;

      img {
        width: 15px;
        opacity: 0;
      }

      &:not(:last-child) {
        border-bottom: solid 1px #DEDEDE;
      }

      &:hover{
        font-weight: bold;

        img {
          transform: translateX(5px);
          opacity: 1;
          transition: all .2s ease-in;
        }
      }
    }
  }
}
</style>
