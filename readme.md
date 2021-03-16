# From DAC-DMA to ADC to DAC out

Different strategies for DAC-DMA to ADC to DAC out 

Almost same as (https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6_HRTIM)[https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6_HRTIM] for the 2 Freq generator by DAC-DMA but will output the ADC data to DAC by ADC IT  

## Setup

### GPIO  

* GPIOC6/8 is used for troubleshooting  

### TIM6 as 2MHz to trigger DAC3  

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

### DAC3 for 2 Freq generator using DMA  

* Set DAC High Freq. = 160MHz 
* Trigger TIM6 Out Event when using TIM6 as trigger
* Trigger EXT line 9 when using HRTIM TimE output as trigger	
	* Connect PC8 to PC9 
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

### OpAmp6  

* Mode = Follower DAC3 output1, input P
* Power Mode = High Speed
* Setup OpAmp6 in main.c  

	/*##- Start OPAMP    #####################################################*/
	/* Enable OPAMP */
	if(HAL_OK != HAL_OPAMP_Start(&hopamp6))
	{
		Error_Handler();
	}

### Master HRTIM is used to trigger ADC by Regular and Injected mode 

* Setup Master HRTIM with 50% duty using CMP1  
* ADC trigger1 on Master Period for Regular Conversion mode 
* ADC trigger2 on Master CMP1 for Injected Conversion mode

### HRTIM TimE used to trigger DAC3 in EXT line 9  

* Setup HRTIM TimE with 50% duty using CMP1  
* Set Interleaved mode = Half
* Set Active source = Period
* Set Reset source = CMP1
* Setup HRTIM Master and TimE in main.c 

	if(HAL_OK != HAL_HRTIM_WaveformOutputStart(&hhrtim1, HRTIM_OUTPUT_TE1))
	{
		Error_Handler();
	}

	if(HAL_OK != HAL_HRTIM_WaveformCounterStart(&hhrtim1, HRTIM_TIMERID_MASTER | HRTIM_TIMERID_TIMER_E))
	{
		Error_Handler();
	}
	
### ADC triggered by HRTIM Master Period and CMP1    

* Setup 1 Regular Conversion mode   
	* External trigger by HRTIM trig 1 event
* Setup 1 Injected Conversion mode   
	* External trigger by HRTIM trig 2 event
* Enable NVIC Global IT with Call HAL handler
* Add callback for Regular Conversion mode

	HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
	{
	  /* Prevent unused argument(s) compilation warning */
	  UNUSED(hadc);

	  /* NOTE : This function should not be modified. When the callback is needed,
				function HAL_ADC_ConvCpltCallback must be implemented in the user file.
	   */

	//  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, SET);
	  HAL_GPIO_WritePin(GPIOC, GPIO_PIN_6, SET);
	  adc_data = HAL_ADC_GetValue(hadc);
	  HAL_DAC_SetValue(&hdac4, DAC_CHANNEL_1, DAC_ALIGN_12B_R, adc_data);
	}

* Add callback for Injected Conversion mode  

	HAL_ADCEx_InjectedConvCpltCallback(ADC_HandleTypeDef *hadc)
	{
	  /* Prevent unused argument(s) compilation warning */
	  UNUSED(hadc);

	  /* NOTE : This function should not be modified. When the callback is needed,
				function HAL_ADCEx_InjectedConvCpltCallback must be implemented in the user file.
	  */
	//  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, SET);
	  HAL_GPIO_WritePin(GPIOC, GPIO_PIN_6, RESET);
	  adc_data = HAL_ADCEx_InjectedGetValue(hadc, ADC_INJECTED_RANK_1);
	  HAL_DAC_SetValue(&hdac4, DAC_CHANNEL_1, DAC_ALIGN_12B_R, adc_data);
	}
	
* Enable ADC and IT in main.c 

	/*##- Enable Injected ADC Channel IT ##############################*/
	if(HAL_OK != HAL_ADCEx_InjectedStart_IT(&hadc1))
	{
		/* Start Error */
		Error_Handler();
	}

	/*##- Enable Regular Injection ADC Channel ##############################*/
	if(HAL_OK != HAL_ADC_Start_IT(&hadc1))
	{
		/* Start Error */
		Error_Handler();
	}
	
### DAC4  

* Enable DAC4  
* Set DAC High Freq. = 160MHz 
* Trigger by Software  
* setup DAC4 in main.c  

	/*##- Enable DAC Channel ##############################*/
	if(HAL_OK != HAL_DAC_Start(&hdac4, DAC_CHANNEL_1))
	{
		/* Start Error */
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
	
### Troubleshooting

* Case 1 = When TIM6 and DAC3 are enabled
	* ADC not triggered, no Output on PC8
* Case 2 = When TIM6 is disabled and DAC3 is enabled
	* ADC triggered but no DAC3 output
* Case 3 = When TIM6 is enabled and DAC3 is disabled
	* ADC triggered but no DAC3 output
	
### Now OK!

* Need to enable NVIC of DAC3 DMA with HAL Call handler

### Waveforms

