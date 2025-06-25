<template>
  <div class="container">
    <h1>IEEE 754 Floating Point Visualizer</h1>
    
    <div class="input-section">
      <div class="input-group">
        <label for="numberInput">Enter Binary:</label>
        <input 
          type="text" 
          id="numberInput" 
          v-model="inputValue"
          placeholder="e.g., 100.001 or 1100011000000"
          pattern="[01.]*"
          @input="updateVisualization"
        >
      </div>
    </div>

    <div class="visualization" v-html="visualizationHtml"></div>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue'

export default {
  name: 'App',
  setup() {
    const inputValue = ref('100.001')
    const visualizationHtml = ref('')
    const currentFormat = ref(32)

    const parseSpecialInput = (str) => {
      const s = str.trim().toLowerCase()
      if (s === 'inf' || s === 'infinity') return Infinity
      if (s === '-inf' || s === '-infinity') return -Infinity
      if (s === 'nan') return NaN
      return null
    }

    const isValidBinary = (str) => {
      const dotCount = (str.match(/\./g) || []).length
      return dotCount <= 1 && /^[01.]+$/.test(str) && str.length > 0
    }

    const binaryToDecimal = (binaryStr) => {
      if (binaryStr.includes('.')) {
        const [integerPart, fractionalPart] = binaryStr.split('.')
        
        let decimal = 0
        if (integerPart) {
          decimal = parseInt(integerPart, 2)
        }
        
        if (fractionalPart) {
          for (let i = 0; i < fractionalPart.length; i++) {
            if (fractionalPart[i] === '1') {
              decimal += Math.pow(2, -(i + 1))
            }
          }
        }
        
        return decimal
      } else {
        return parseInt(binaryStr, 2)
      }
    }

    const generateBinaryConversionSteps = (binaryStr) => {
      if (!binaryStr.includes('.')) {
        const integerPart = binaryStr
        let steps = '<div class="calc-step">Integer part: ' + integerPart + '</div>'
        let calculation = ''
        let total = 0
        
        for (let i = 0; i < integerPart.length; i++) {
          const bit = integerPart[i]
          const power = integerPart.length - 1 - i
          const value = parseInt(bit) * Math.pow(2, power)
          total += value
          
          if (bit === '1') {
            if (calculation) calculation += ' + '
            calculation += `${bit}×2^${power}`
          }
        }
        
        if (calculation) {
          steps += `<div class="calc-step">${calculation} = ${total}</div>`
        }
        
        return steps
      } else {
        const [integerPart, fractionalPart] = binaryStr.split('.')
        let steps = ''
        let totalInteger = 0
        let totalFractional = 0
        
        if (integerPart && integerPart !== '0') {
          steps += '<div class="calc-step">Integer part: ' + integerPart + '</div>'
          let intCalculation = ''
          
          for (let i = 0; i < integerPart.length; i++) {
            const bit = integerPart[i]
            const power = integerPart.length - 1 - i
            const value = parseInt(bit) * Math.pow(2, power)
            totalInteger += value
            
            if (bit === '1') {
              if (intCalculation) intCalculation += ' + '
              intCalculation += `${bit}×2^${power}`
            }
          }
          
          if (intCalculation) {
            steps += `<div class="calc-step">${intCalculation} = ${totalInteger}</div>`
          }
        }
        
        if (fractionalPart) {
          steps += '<div class="calc-step">Fractional part: .' + fractionalPart + '</div>'
          let fracCalculation = ''
          
          for (let i = 0; i < fractionalPart.length; i++) {
            const bit = fractionalPart[i]
            const power = -(i + 1)
            const value = parseInt(bit) * Math.pow(2, power)
            totalFractional += value
            
            if (bit === '1') {
              if (fracCalculation) fracCalculation += ' + '
              fracCalculation += `${bit}×2^${power}`
            }
          }
          
          if (fracCalculation) {
            steps += `<div class="calc-step">${fracCalculation} = ${totalFractional}</div>`
          }
        }
        
        const total = totalInteger + totalFractional
        steps += `<div class="calc-step">Total: ${totalInteger} + ${totalFractional} = ${total}</div>`
        
        return steps
      }
    }

    const generateFloat32Visualization = (number, binaryInput) => {
      const buffer = new ArrayBuffer(4)
      const view = new DataView(buffer)
      view.setFloat32(0, number)
      const bits = view.getUint32(0)
      
      const sign = (bits >>> 31) & 1
      const exponent = (bits >>> 23) & 0xFF
      const mantissa = bits & 0x7FFFFF
      
      const signBit = sign.toString()
      const exponentBits = exponent.toString(2).padStart(8, '0')
      const mantissaBits = mantissa.toString(2).padStart(23, '0')
      
      const biasedExponent = exponent
      const actualExponent = exponent - 127
      const mantissaValue = 1 + mantissa / Math.pow(2, 23)
      
      const bitLabels = Array.from({length: 32}, (_, i) => 31 - i)
      
      return `
        <div class="ieee-representation">
          <h3 style="text-align: center; margin-bottom: 20px;">32-bit IEEE 754 Representation</h3>
          
          <div style="margin-bottom: 20px;">
            <div style="text-align: center; margin-bottom: 10px;">
              <strong>Input Binary Integer:</strong> ${binaryInput} = ${number} (decimal)
            </div>
          </div>
          
          <div class="bit-labels">
            ${bitLabels.map(label => `<div class="bit-label">${label}</div>`).join('')}
          </div>
          
          <div class="bit-display">
            <div class="bit sign-bit">${signBit}</div>
            ${exponentBits.split('').map(bit => `<div class="bit exponent-bit">${bit}</div>`).join('')}
            ${mantissaBits.split('').map(bit => `<div class="bit mantissa-bit">${bit}</div>`).join('')}
          </div>
          
          <div class="legend">
            <div class="legend-item">
              <div class="legend-color sign-bit"></div>
              <span>Sign (1 bit)</span>
            </div>
            <div class="legend-item">
              <div class="legend-color exponent-bit"></div>
              <span>Exponent (8 bits)</span>
            </div>
            <div class="legend-item">
              <div class="legend-color mantissa-bit"></div>
              <span>Mantissa (23 bits)</span>
            </div>
          </div>
        </div>
        
        <div class="calculations">
          <div class="calc-section">
            <div class="calc-title">Binary to Decimal Conversion</div>
            <div class="calc-step">Binary input: ${binaryInput}</div>
            ${generateBinaryConversionSteps(binaryInput)}
            <div class="calc-step"><strong>Decimal value: ${number}</strong></div>
          </div>
        
          <div class="calc-section">
            <div class="calc-title">Sign Calculation</div>
            <div class="calc-step">Sign bit: ${sign}</div>
            <div class="calc-step">Sign: ${sign === 0 ? 'Positive (+)' : 'Negative (-)'}</div>
          </div>
          
          <div class="calc-section">
            <div class="calc-title">Exponent Calculation</div>
            <div class="calc-step">Exponent bits: ${exponentBits}</div>
            <div class="calc-step">Biased exponent: ${biasedExponent}</div>
            <div class="calc-step">Bias: 127</div>
            <div class="calc-step">Actual exponent: ${biasedExponent} - 127 = ${actualExponent}</div>
          </div>
          
          <div class="calc-section">
            <div class="calc-title">Mantissa Calculation</div>
            <div class="calc-step">Mantissa bits: ${mantissaBits}</div>
            <div class="calc-step">Mantissa decimal: ${mantissa}</div>
            <div class="calc-step">Normalized: 1 + ${mantissa}/2²³</div>
            <div class="calc-step">Value: ${mantissaValue.toFixed(10)}</div>
          </div>
        </div>
        
        <div class="result">
          Final IEEE 754 Value: ${sign === 0 ? '+' : '-'} ${mantissaValue.toFixed(10)} × 2^${actualExponent} = ${view.getFloat32(0)}
        </div>
      `
    }

    const updateVisualization = () => {
      const special = parseSpecialInput(inputValue.value)

      if (special !== null) {
        visualizationHtml.value = generateFloat32Visualization(special, inputValue.value)
        return
      }

      const binaryString = inputValue.value.replace(/[^01.]/g, '')
      if (!binaryString || !isValidBinary(binaryString)) {
        visualizationHtml.value = '<div class="calc-section"><div class="calc-title">Please enter a valid binary number (0s, 1s, decimal point), or type "inf", "-inf", or "nan"</div></div>'
        return
      }
      
      const number = binaryToDecimal(binaryString)
      visualizationHtml.value = generateFloat32Visualization(number, binaryString)
    }

    onMounted(() => {
      updateVisualization()
    })

    return {
      inputValue,
      visualizationHtml,
      updateVisualization
    }
  }
}
</script>
