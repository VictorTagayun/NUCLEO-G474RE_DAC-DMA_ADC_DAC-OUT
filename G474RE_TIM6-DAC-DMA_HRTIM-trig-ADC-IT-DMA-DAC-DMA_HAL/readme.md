# 2 Freq generator by DAC-DMA with TIM6. HRTIM will trigger ADC by Regular conversion "ONLY". ADC will Interrupt after ADC conversion and output and also transfer data by DMA. DAC4 will show ADC by DMA. DAC1 will show data by ADC IT.

Almost same as (https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6_HRTIM)[https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6_HRTIM] for the 2 Freq generator by DAC-DMA but will output the ADC data to DAC by IT  

## Setup

### GPIO  

* GPIOC11 is used for troubleshooting  
* GPIOA5 is used for LED/Troubleshooting    

### TIM6 as 2MHz

* Activate TIM6
* Prescaler = 17-1  
* ARR = 5-1
* TRGO Event = Update event
* setup in main.c 

	/*##- Enable TIM peripheral counter ######################################*/
	if(HAL_OK != HAL_TIM_Base_Start(&htim6))
	{
		Error_Handler();
	}

### DAC3 for 2 Freq generator using DMA by TIM6   

* Set DAC High Freq. = 160MHz 
* Trigger TIM6 Out Event
* DMA Settings
	* Mode = Circular
	* Increment Adress = Memory
	* Data width = Word
* Disable DMA IT
* add #include "waveforms.h"
* add High freq to become 2 freq

	for (uint16_t cntr = 0; cntr < MySine2000_SIZE; cntr++)
	{
		MySine2000[cntr] += 682;
		MySine2000[cntr] += MySine200[cntr % MySine200_SIZE];
	}
	
* setup DAC3 in main.c  

	/*##- Enable DAC Channel and associated DMA ##############################*/
	if(HAL_OK != HAL_DAC_Start_DMA(&hdac3, DAC_CHANNEL_1,
				   (uint32_t*)MySine2000, MySine2000_SIZE, DAC_ALIGN_12B_R))
	{
		/* Start DMA Error */
		Error_Handler();
	}

### OpAmp6 Output from DAC3    

* Mode = Follower DAC3 output1, input P
* Power Mode = High Speed
* Setup OpAmp6 in main.c  

	/*##- Start OPAMP    #####################################################*/
	/* Enable OPAMP */
	if(HAL_OK != HAL_OPAMP_Start(&hopamp6))
	{
		Error_Handler();
	}

### HRTIM TimE1, trigger ADC internally and DAC4 externally   

* Setup HRTIM TimE with 50% duty using CMP1  
* ADC trigger1 on TimE Period  
* Set Interleaved mode = Half
* Set Active source = Period
* Set Reset source = CMP1
* Setup HRTIM Master and TimE in main.c 

	if(HAL_OK != HAL_HRTIM_WaveformOutputStart(&hhrtim1, HRTIM_OUTPUT_TE1))
	{
		Error_Handler();
	}

	if(HAL_OK != HAL_HRTIM_WaveformCounterStart(&hhrtim1, HRTIM_TIMERID_TIMER_E))
	{
		Error_Handler();
	}
		
### ADC DMA to DAC4 DMA    

* Setup 1 Regular Conversion mode   
	* External trigger by HRTIM trig 1 event (TimE)  
* Add DMA
	* Mode Circular @ Memory, data width Word
* ADC_Settings
	* DMA Continous Request = Enable
* NVIC, 
	* Enable DMA global IT with Call handler
	* Disable ADC1-2 Global IT
* Add callback for Regular Conversion mode, later will be used for DAC1 output

	HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
	{
	  /* Prevent unused argument(s) compilation warning */
	  UNUSED(hadc);

	  /* NOTE : This function should not be modified. When the callback is needed,
				function HAL_ADC_ConvCpltCallback must be implemented in the user file.
	   */

		HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_11);
		adc_data = HAL_ADC_GetValue(hadc);
		HAL_DAC_SetValue(&hdac4, DAC_CHANNEL_1, DAC_ALIGN_12B_R, adc_data);
	}

* Enable ADC and DMA    

	/*##- Enable ADC Channel and associated DMA ##############################*/
	if(HAL_OK != HAL_ADC_Start_DMA(&hadc1, &adc_dac_value, 1))
	{
		/* Start DMA Error */
		Error_Handler();
	}


### DAC4 for DMA by EXTI   

* Enable DAC4  
* Click External Trigger
* Set DAC High Freq. = 160MHz 
* Trigger by External Line 9   
* NVIC 
	* enable DMA global IT with Call handler 
	* Disable EXT line IT
* Add DMA
	* Mode Circular @ Memory, data width Word
* GPIO Setting (EXTI trigger)
	* External Event/IT with Falling edge trigger detection
* setup DAC4 in main.c  

	/*##- Enable DAC Channel and associated DMA ##############################*/
	if(HAL_OK != HAL_DAC_Start_DMA(&hdac4, DAC_CHANNEL_1,
								&adc_dac_value, 1, DAC_ALIGN_12B_R))
	{
		/* Start DMA Error */
		Error_Handler();
	}
	
### OpAmp4  

* Mode = Follower DAC3 output1, input P
* Power Mode = High Speed
* Setup OpAmp4 in main.c  

	/*##- Start OPAMP    #####################################################*/
	/* Enable OPAMP */
	if(HAL_OK != HAL_OPAMP_Start(&hopamp4))
	{
		Error_Handler();
	}
	
### DAC1 for ADC Data _HAL_ADC_ConvCpltCallback_ 

* Enable DAC1  
* Output buffer = Enable
* DAC High Freq = Above 160MHz
* No Trigger by anything even Software
* setup DAC1 in main.c  

	/*##- Enable DAC Channel ##############################*/
	if(HAL_OK != HAL_DAC_Start(&hdac1, DAC_CHANNEL_1))
	{
		/* Start Error */
		Error_Handler();
	}

### Troubleshooting

* DAC1-2 cannot output ADC data if triggered by software, etc.

