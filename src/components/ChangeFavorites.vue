<template>
  <div>
    <div id = "favMaster" >
      <h1 id = "header">ÄNDRA PÅ FAVORITERNA</h1>
      <div id = "burger" >

        <label>Namn på ny burgare: </label>
          <input
          type="text"
           v-model = 'rows.name'>

          <label>URL till den nya bilden: </label>
            <input
            type="text"
            v-model = 'rows.url'>

            <label>Beskrivning av den nya burgaren (sv): </label>
              <textarea
              rows="4"
              v-model = 'rows.description_sv'></textarea>

              <label>Beskrivning av den nya burgaren (en): </label>
              <textarea
              rows="4"
              v-model = 'rows.description_en'></textarea>

                <div v-for="(row, index) in rows"
                class="new-ing"
                :key = "`row-${index}`">

                  <p>{{row.title}} {{index+1}}</p>
                  <input v-model="row.value"><button id="cancelBtn" @click = "remove(index)">Ta bort</button>
                </div>
                <button class = 'addRow' @click = "addRow()">Lägg till ingrediens</button>
                <br>
                <br>
                <h2>VILKEN BURGARE VILL DU ERSÄTTA?</h2>
                <div id="radio-done">

                  <div id="radio">
                <div
                v-for = "(item, index) in favBurgers"
                class = "existingBurger"
                :key = "`item-${index}`">

                <input
                type="radio"
                class = "radio"
                v-model = 'rows.checked'
                :value = "index">

                <label>{{item.name}}</label>

              </div>

              <div class ="existingBurger">
                <input
                type="radio"
                class = "radio"
                v-model='rows.checked'
                value = 'add' >
                <label>Lägg till som en ny hamburgare</label>
              </div>
            </div>
              <button class = 'confirm' @click = "updateInfo">BEKRÄFTA</button>
            </div>
          </div>

            <div role="tablist" id="fullaccordion">

              <b-card
              no-body
              class="mb-1"
              v-for="(tab, index) in tabs"
              :key="index">

              <b-card-header
              header-tag="header"
              class="p-1"
              role="tab">

              <b-btn
              block
              href="#"
              v-b-toggle="'accordion'+index"
              variant="info">{{tab.label}}</b-btn>

            </b-card-header>

            <b-collapse
            :id="'accordion'+index"
            accordion="my-accordion"
            role="tabpanel">

            <b-card-body>
              <div class="ing-container">
                <p>ID:</p>
                <p>Namn (sv):</p>
                <p>Namn (en):</p>
                <p>Lager:</p>
              </div>
              <div
              v-for="ingredient in ingredients"
              :key="ingredient.ingredient_id">
              <div
              v-if="ingredient.category===tab.category"
              class="ing-container">
              <div>
                {{ingredient.ingredient_id}}
              </div>
              <div>
                {{ingredient.ingredient_sv}}
              </div>
              <div>
                {{ingredient.ingredient_en}}
              </div>
              <div>
                {{ingredient.stock}}
              </div>
            </div>
          </div>
        </b-card-body>
      </b-collapse>
    </b-card>
  </div>

  <SlotModal
  v-if="this.showSlotModal">
  <div slot="body"><h3>Fel inparametrar, testa igen!</h3></div>
  <div slot="footer"><button id="goBack" @click = 'favError'>Gå tillbaka</button></div>
</SlotModal>
</div>
</div>
</template>

<script>

import SlotModal from '@/components/SlotModal.vue'

export default {
  name: 'ChangeFavorites',
  components:{
    SlotModal
  },
  props:{
    ingredients: Array,
    favBurgers: Array
  },
  data: function(){
    return{
      rows: [],
      showSlotModal: false,
      tabs:[
        {'label':'Bröd',
        'category':4},
        {'label':'Protein',
        'category':1},
        {'label':'Pålägg',
        'category':2},
        {'label':'Sås',
        'category':3},
      ]
    }
  },
  methods: {
    addRow: function(){ //lägger till en ingrediens
      this.rows.push({
        title: 'Ingrediens'
      })
    },
    //funktionen hittar ingredienserna baserat på ingrediensernas svenska
    //och engelska namn och skickar informationen till servern
    //funktionen har en inbyggd errrorhandler som hanterar fel som kan uppstå
    //och medddelar användaren
    updateInfo: function(){
      let favoriteIngredients = [];
      let favoritePrice = 0;
      try{
        for (var i = 0; i< this.rows.length; i++){
          let ingredient = this.findIngredient(i);
          favoriteIngredients.push(ingredient); /*lägger favoritingredienser i en array*/
          favoritePrice += ingredient.selling_price; /*räknar ut priset för de ingredienserna*/
        }
      }
      catch(err){
        this.favError(); //Kallar på funktionen som hanterar felet
      }
      if(this.checkIfChecked() && !this.showSlotModal){

        this.URLValidation()

        let allergies = this.checkAllergies(favoriteIngredients);

        let burger = {
          "name": this.capitalize(this.rows.name),
          "url": this.rows.url,
          "ingredients": favoriteIngredients,
          "price": favoritePrice,
          "index": this.rows.checked,
          "selected": false,
          "description_sv": this.rows.description_sv,
          "description_en": this.rows.description_en,
          "gluten_free": allergies.gluten_free,
          "lactose_free":allergies.lactose_free,
          "vegan":allergies.vegan
        }
        this.$store.state.socket.emit('updateinfo', burger);
        this.rows = [];
      }
    },
    /*Tittar i rows och ser om den angivna ingrediensen finns, både
    på id, sv och en*/
    findIngredient:function(index){
      let ingredient = this.ingredients.find(ingredient=>ingredient.ingredient_sv.toUpperCase() === this.rows[index].value.toUpperCase());
      if(ingredient == undefined){
        ingredient = this.ingredients.find(ingredient=>ingredient.ingredient_en.toUpperCase() === this.rows[index].value.toUpperCase());
        if(ingredient == undefined){
          ingredient = this.ingredients.find(ingredient=>ingredient.ingredient_id == this.rows[index].value);
        }
    }
    return ingredient;
  },
    /*Tar en array av ingredienser och ser vilka allergener den innehåller*/
    checkAllergies:function(ingredients){
      let allergies = {
        'gluten_free': true,
      'lactose_free': true,
      'vegan': true };

      for(let i = 0;i<ingredients.length;i++){
        if(ingredients[i].milk_free === 0){
          allergies.lactose_free = false;
        }
        if(ingredients[i].gluten_free === 0){
          allergies.gluten_free = false;
        }
        if(ingredients[i].vegan === 0){
          allergies.vegan = false;
        }
      }
      return allergies;
    },
    /*Kollar om längden på urlen är 0 efter att vi
    trimmat den (dvs kollar om urlen bara är massa mellanslag)*/
    URLValidation:function(){
      if (this.rows.url === undefined||this.rows.url.trim().length === 0){
        this.rows.url='https://toppng.com/public/uploads/preview/fast-food-burger-11528345395r3cdlrs6sr.png';
      }
    },
    favError: function(){ //funktion som visar eller döljer felmodalen
      this.showSlotModal = !this.showSlotModal;
    },
    /*Gör första bokstaven i namnet på burgaren till stor bokstav*/
    capitalize:function(string){
      let firstLetter = string.charAt(0).toUpperCase();
      let restOfString = string.slice(1);
      return firstLetter + restOfString;
    },
    checkIfChecked: function(){ //kollar att någon radiobutton är checked, kallar på felhanteraren annars
      if(this.rows.checked != null){
        return true;
      }
      else{
        this.showSlotModal = true;
        return false;
      }
    },
    remove: function(index){ //tar bort en ingrediens
      this.rows.splice(index,1);
    }
  }
}
</script>

<style scoped>

#favMaster{
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  padding-top:15px;
  grid-column-gap: 3em;
}
#header{
  margin-left:20px;
}
#burger{
  display:grid;
  grid-column: 1/4;
  grid-row:2/3;
  margin-left:20px;
}
#burger > label{
  margin-bottom:0;
}

#radio-done{
  display:grid;
  grid-template-columns: 2fr 1fr;
  margin-bottom:25px;
}

#fullaccordion{
  grid-column:4/8;
  grid-row: 2;
  margin-right:20px;
  margin-top: calc(1em + 2.4px);
}

#header{
  grid-column: 1/8;
  grid-row:1/2;
}

#cancelBtn{
  background-color: red;
  color: white;
  border: 2px solid white;
  border-radius: 7px;
  margin-left: 5px;
  height: 5vh;
}
#cancelBtn:hover{
  background-color: #b30000;
}

.confirm, #goBack{
  background-color: #4CAF50; /* Green */
  border: 2px solid white;
  border-radius: 10px;
  color: white;
  padding: 15px 32px;
  text-align: center;
  text-decoration: none;
  display: inline-block;
}

.confirm{
  font-size: 16px;
}
#goBack{
  height: 10vh;
  width: 10vw;
  font-size: 18px;
}
input[type="text"],label,textarea{
  display:block;
}
input[type="text"],textarea{
  margin-bottom:10px;
}

.confirm:hover{background-color: #367c39}
#goBack:hover{background-color: #367c39}

.radio{
  -webkit-appearance:button;
  -moz-appearance:button;
  appearance:button;
  border:4px solid #ccc;
  border-top-color:#bbb;
  border-left-color:#bbb;
  background:#fff;
  width:50px;
  height:50px;
  border-radius:50%;
}
.radio:checked{
  border:20px solid #17a2b8;
}

.addRow{
  margin-top: 1vh;
  background-color: #17a2b8;
  color: white;
  border-radius: 7px;
  border: 1px solid white;
  height: 7vh;
}
.addRow:hover{background-color: #138496}

.existingBurger{
  display:grid;
  grid-template-columns: 50px auto;
  grid-gap: 10px;
  margin-bottom: 10px;
}
.existingBurger > *{
  margin: auto auto auto 0;
}

p, label, h1,h2,h4,h5,h6{
  color:white;
}

h2{
  margin-top: 0.5em;
}

.ing-container{
  display: grid;
  grid-template-columns: 10% 40% 40% 10%;
  border-bottom: 1px solid rgba(0,0,0,0.1);
}

.ing-container > p{
  color:black;
  margin:0;
  font-weight:bold;
}

</style>
