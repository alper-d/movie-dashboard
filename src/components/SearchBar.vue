<template lang="pug">
  v-card(dark)
    v-card-text
        v-combobox(  
            ref="combobox"  
            v-model="model"
            :items="items"
            :loading="isLoading"
            :search-input.sync="search"
            color="white"
            clearable
            hide-no-data
            item-text="Description"
            item-value="imdbID"
            label="Search Movie"
            placeholder="Start typing to Search"
            prepend-icon="mdi-database-search"
            return-object
            menu-props="{disableKeys: false}"
            )
    
    v-divider
    v-container(v-if="paginationLoading || retrieveError" fluid)
        v-row(align="center" justify="center" class="my-1")
            v-progress-circular(v-if="paginationLoading" indeterminate color="red") 
            v-alert(v-if="retrieveError" color="red") "Couldn't retrieve data"

    v-expand-transition
        v-container(v-if="detailBlock && details" fluid)
            v-row(align="center" justify="center")
                v-col(cols="12" sm="4" )
                    v-img( :src="details.Poster" v-show="details.Poster")
                v-col(cols="12" sm="8")
                    v-list( class="blue-grey darken-2")
                        v-list-item(v-for="(field, i) in fields" v-if="typeof(field.value) === 'string'" :key="i")
                            v-list-item-content
                                v-list-item-title(v-text="field.value")
                                v-list-item-subtitle(v-text="field.key")

    v-expand-transition
        v-container(v-if="paginatedData" fluid)
            v-pagination(v-if="paginatedData.length" v-model="page" :length="paginationLength" circle)
            v-row
                v-col(v-for="(item, i) in paginatedData" :key="i" cols="12" sm="2" md="3")
                    
                    v-card(class="m-1" height="100%" @click="() => cardClicked(item.imdbID, i)")
                        v-card-title(class="justify-center text-wrap bottom" bottom="0%") {{item.Title}}
                        v-expand-transition
                            div(v-if="item.showDetail")
                                v-img(:src="item.Poster" width="100%")
                                v-card-subtitle(class="text-center text-wrap" ) {{item.Year}}
                                
</template>

<script>

import Vue from 'vue'
//import goTo from 'vuetify/es5/services/goto'

  export default {
    name: "SearchBar",

    data: () => ({
        descriptionLimit: 60,
        page:1,
        paginationLength:1,
        maxPerPage:10,
        entries: [],
        isLoading: false,
        paginationLoading:false,
        model: null,
        search: null,
        paginatedData:null,
        focusedMovie:null,
        details:null,
        detailBlock: false,
        retrieveError:false,
    }),

    computed: {
      fields () {
        if (!this.details) return []

        return Object.keys(this.details).map(key => {
          return {
            key,
            value: this.details[key] || 'n/a',
          }
        })
      },
        items () {
        if (!this.entries) return []
        return this.entries.map(entry => {
            const Description = entry.Title.length > this.descriptionLimit
            ? entry.Title.slice(0, this.descriptionLimit) + '...'
            : entry.Title
            return Object.assign({}, entry, { Description })
        })
        },
    },
    methods: {
        searchQuery (changePagination="",query="") {
            this.retrieveError = false,
            this.paginationLoading = true
            const val = query==="" ? this.search : query
            const page = changePagination==="pagination" ? this.page : 1 
            console.log("searchQuery " + val + " "+page)

            fetch(`http://www.omdbapi.com/?apikey=be1732ea&s=${val}&page=${page}&y=2020`)
                .then(res => res.json())
                .then(res => {
                    const { totalResults:count, Search:paginatedData } = res
                    switch(changePagination){
                        case "autocomplete":{
                            this.entries = paginatedData
                            break
                        }
                        case "pagination" :{
                            this.paginatedData = paginatedData.map(e => {
                                return Object.assign({}, e, {showDetail: false})
                            })
                            
                            this.paginationLength = Math.ceil((+count)/this.maxPerPage)
                            break
                        }
                        case "full_search":{
                            this.page = 1
                            this.paginatedData = paginatedData.map(e => {
                                return Object.assign({}, e, {showDetail: false})
                            })
                            this.paginationLength = Math.ceil((+count)/this.maxPerPage)
                        }
                    }
                    console.log(count)
                })
                .catch(err => {
                    console.log(err)
                    this.paginatedData= []
                    this.paginationLength = 1
                    this.retrieveError = true
                }).finally(() => {this.paginationLoading = false})
        },
        setData(movie=null){
            console.log("setData")
            const imdbID = movie
            this.retrieveError=false
            this.paginationLoading = true
            fetch(`http://www.omdbapi.com/?apikey=be1732ea&i=${imdbID}` )
                .then(res => res.json())
                .then(res => {
                    Object.keys(res).map((key) => {
                        if (res[key] === 'N/A' && key !== "Poster") delete res[key]})
                    Object.assign(res,{})
                    this.details = Object.assign({}, res)
                })
                .catch(err => {
                    console.log(err)
                    this.retrieveError=true
                }).finally(() => {this.paginationLoading = false}) 
        },
        cardClicked(imdbID, cardIndex) {
            this.focusedMovie = imdbID
            this.paginatedData.map((data, i) => {
                let newEntry = null
                if(i===cardIndex){
                    this.detailBlock=!this.paginatedData[cardIndex].showDetail
                    newEntry = Object.assign({}, data, {showDetail:!this.paginatedData[cardIndex].showDetail})
                }else{
                    newEntry = Object.assign({}, data, {showDetail:false})
                }
                Vue.set(this.paginatedData, i, newEntry)    
            })
        }
    },

    watch: {
        model(){
            console.log(this.model)
            this.$refs.combobox.isMenuActive = false
            this.isLoading = true
            this.detailBlock = false
            if(typeof(this.model) === 'object' && this.model !== null){
                this.detailBlock = true
                this.searchQuery("full_search",this.model.Title)
            }else{
                this.searchQuery("full_search",this.model.Title)
            }
            this.isLoading = false
        },
        search () {
            console.log("search watch")
            if (this.isLoading) return
            this.isLoading = true
            this.searchQuery("autocomplete")
            this.isLoading = false
        },
        page (){
            this.searchQuery("pagination")
        },
        focusedMovie(){
            this.setData(this.focusedMovie)
    }
  }
  }
</script>



