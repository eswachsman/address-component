<template>
  <div class="address-form">
    <h2>Add an address</h2>

    <div class="form-row">
      <div class="form-group">
        <label>Country: <span class="required">*</span></label>
        <div class="select-wrapper">
          <select 
            v-model="selectedCountry"
            @change="handleCountryChange"
          >
            <option 
              v-for="country in countries" 
              :key="country.code" 
              :value="country"
            >
              {{ country.name }}
            </option>
          </select>
        </div>
      </div>
    </div>
    
    <div class="form-row">
      <div class="form-group">
        <label>Address 1: <span class="required">*</span></label>
        <input 
          type="text" 
          v-model="searchText"
          @input="handleAddressInput"
          placeholder="Start typing to search..."
        >
        <div v-if="isLoading" class="loading-indicator"></div>
        <div v-if="predictions.length > 0" class="predictions-list">
          <div 
            v-for="prediction in predictions" 
            :key="prediction.id"
            class="prediction-item"
            @click="selectPrediction(prediction)"
          >
            {{ prediction.description }}
          </div>
        </div>
      </div>
      <div class="form-group">
        <label>Address 2:</label>
        <input 
          type="text" 
          v-model="address.line2"
          placeholder="Apt, Suite, Unit number"
        >
      </div>
    </div>

    <div class="form-row">
      <div class="form-group">
        <label>{{ stateLabel }}: <span class="required">*</span></label>
        <input 
          type="text" 
          v-model="stateSearchText"
          @input="handleStateInput"
          @focus="showStatesList = true"
          :placeholder="statePlaceholder"
          :readonly="!['US', 'CA'].includes(selectedCountry.code)"
        >
        <div v-if="showStatesList && filteredRegions.length > 0" class="predictions-list">
          <div 
            v-for="region in filteredRegions" 
            :key="region.abbr"
            class="prediction-item"
            @click="selectRegion(region)"
          >
            {{ region.name }}
          </div>
        </div>
      </div>
      <div class="form-group">
        <label>City: <span class="required">*</span></label>
        <input 
          type="text" 
          v-model="address.city"
          placeholder="City"
        >
      </div>
    </div>

    <div class="form-row">
      <div class="form-group">
        <label>ZIP: <span class="required">*</span></label>
        <input 
          type="text" 
          v-model="address.zip"
          placeholder="ZIP code"
        >
      </div>
      <div class="form-group">
        <!-- Empty group for layout balance -->
      </div>
    </div>

    <div class="button-group">
      <button class="btn btn-cancel" @click="cancel">Cancel</button>
      <button class="btn btn-save" @click="save">Save</button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import { streetNames, cities, countries, usStates, canadianProvinces } from './mockData'

const searchText = ref('')
const address = ref({
  line1: '',
  line2: '',
  state: '',
  city: '',
  zip: '',
  country: 'United States'
})

const predictions = ref([])
const isLoading = ref(false)
const selectedCountry = ref(countries[0]) // Default to US
const stateSearchText = ref('')
const showStatesList = ref(false)

const getRandomItems = (arr, count) => {
  const shuffled = [...arr].sort(() => 0.5 - Math.random())
  return shuffled.slice(0, count)
}

const generateZip = () => {
  return String(Math.floor(Math.random() * 90000) + 10000)
}

const handleAddressInput = () => {
  if (!searchText.value) {
    predictions.value = []
    return
  }

  isLoading.value = true
  
  setTimeout(() => {
    const input = searchText.value.toLowerCase()
    let streetNumber = ''
    
    // Extract number from input if it exists
    const numberMatch = input.match(/^\d+/)
    if (numberMatch) {
      streetNumber = numberMatch[0]
    }

    // Get filtered streets that match input (excluding number)
    let matchingStreets = streetNames.filter(street => 
      street.toLowerCase().includes(input.replace(/^\d+\s*/, ''))
    )

    // If no matches found, use random streets instead
    if (matchingStreets.length === 0) {
      matchingStreets = getRandomItems(streetNames, 5)
    }

    // Generate 5 predictions
    predictions.value = matchingStreets.slice(0, 5).map((street, index) => {
      const randomCity = getRandomItems(cities, 1)[0]
      const randomState = getRandomItems(usStates, 1)[0]
      const number = streetNumber || Math.floor(Math.random() * 9000) + 1000

      return {
        id: index,
        description: `${number} ${street}, ${randomCity}, ${randomState.abbr}`,
        details: {
          number,
          street,
          city: randomCity,
          state: randomState.abbr,
          zip: generateZip()
        }
      }
    })

    isLoading.value = false
  }, 300)
}

const selectPrediction = (prediction) => {
  searchText.value = `${prediction.details.number} ${prediction.details.street}`
  address.value.line1 = searchText.value
  address.value.city = prediction.details.city
  address.value.state = prediction.details.state
  address.value.zip = prediction.details.zip
  predictions.value = []
}

const stateLabel = computed(() => {
  if (selectedCountry.value.code === 'US') return 'State'
  if (selectedCountry.value.code === 'CA') return 'Province'
  return 'State / Province / Region'
})

const statePlaceholder = computed(() => {
  if (selectedCountry.value.code === 'US') return 'Select state'
  if (selectedCountry.value.code === 'CA') return 'Select province'
  return 'Enter state/province/region'
})

const availableRegions = computed(() => {
  if (selectedCountry.value.code === 'US') return usStates
  if (selectedCountry.value.code === 'CA') return canadianProvinces
  return []
})

const filteredRegions = computed(() => {
  if (!stateSearchText.value) return availableRegions.value
  const search = stateSearchText.value.toLowerCase()
  return availableRegions.value.filter(region => 
    region.name.toLowerCase().includes(search) || 
    region.abbr.toLowerCase().includes(search)
  )
})

const handleCountryChange = () => {
  address.value.state = ''
  stateSearchText.value = ''
  showStatesList.value = false
  
  // Clear address if switching countries
  address.value.line1 = ''
  address.value.line2 = ''
  address.value.city = ''
  address.value.zip = ''
  searchText.value = ''
  predictions.value = []
}

const handleStateInput = () => {
  if (['US', 'CA'].includes(selectedCountry.value.code)) {
    showStatesList.value = true
  }
}

const selectRegion = (region) => {
  address.value.state = region.abbr
  stateSearchText.value = region.name
  showStatesList.value = false
}

const save = () => {
  const required = ['line1', 'city', 'zip', 'state']
  const isValid = required.every(field => address.value[field])
  
  if (!isValid) {
    alert('Please fill in all required fields')
    return
  }
  
  console.log('Saving address:', {
    ...address.value,
    country: selectedCountry.value.name
  })
}

const cancel = () => {
  searchText.value = ''
  stateSearchText.value = ''
  showStatesList.value = false
  selectedCountry.value = countries[0] // Reset to US
  Object.keys(address.value).forEach(key => {
    address.value[key] = key === 'country' ? 'United States' : ''
  })
}

// Close predictions when clicking outside
document.addEventListener('click', (e) => {
  if (!e.target.closest('.form-group')) {
    predictions.value = []
    showStatesList.value = false
  }
})
</script>

<style scoped>
.select-wrapper {
  position: relative;
}

.select-wrapper select {
  width: 100%;
  padding: 10px 14px;
  border: 1.5px solid #e0e0e0;
  border-radius: 8px;
  font-size: 14px;
  background-color: white;
  appearance: none;
  cursor: pointer;
}

.select-wrapper::after {
  content: '';
  position: absolute;
  right: 14px;
  top: 50%;
  transform: translateY(-50%);
  width: 0;
  height: 0;
  border-left: 5px solid transparent;
  border-right: 5px solid transparent;
  border-top: 5px solid #4a5568;
  pointer-events: none;
}

.select-wrapper select:focus {
  outline: none;
  border-color: #2196f3;
  box-shadow: 0 0 0 3px rgba(33, 150, 243, 0.1);
}
</style>
