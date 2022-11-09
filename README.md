# LAB_Timer_InputCaputre_Ultrasonic

### LAB : Timer InputCaputre Ultrasonic

**Date:** 2022.11.10

**Author/Partner:** Heo Dabin/ Hwang Seungeon 

**Github:** [Github code](https://github.com/DABINHEO/LAB_Timer_InputCaputre_Ultrasonic.git)

**Demo Video:** [Youtube Link](https://youtu.be/x0IUM98kR-M)

##            



### Introduction

In this LAB, an ultrasonic distance sensor will be used in input capture mode to create a program to measure the distance between the sensor and an object from which ultrasonic waves collide. First, we will write a header code for input capture, which is to set the start, edge type, and period of input capture mode. We will use a pwm signal with a pulse width of 10 [usec], with a period of 50 [msec] as a trigger for the ultrasonic distance sensor, and take an echo pulse from the input capture pin to calculate the distance to the object.



### Requirement

#### Hardware

* MCU
  
  * NUCLEO-F401RE
* Sensor
   * HC-SR04
   

   

#### Software

* Keil uVision, CMSIS, EC_HAL library

##          



### Problem1: Create HAL library

We wrote a header code for input capture, which is to set the start, edge type, and period of input capture mode. And also wrote a code to perform a specific action when capturing a specific signal input.

#### Description with Code

* ecTIM.c  description

```c++
// Input Capture configuration ---------------------------------------------------------------------			
// 1. Select Timer channel(TIx) for Input Capture channel(ICx)
// Default Setting1
TIMx->CCMR1 &= 	~TIM_CCMR1_CC1S;
TIMx->CCMR1 &=	~TIM_CCMR1_CC2S;
TIMx->CCMR2 &=	~TIM_CCMR2_CC3S;
TIMx->CCMR2 &=	~TIM_CCMR2_CC4S;
TIMx->CCMR1 |= 	TIM_CCMR1_CC1S_0;      		//01<<0   CC1S    TI1=IC1
TIMx->CCMR1 |= 	TIM_CCMR1_CC2S_0;     		//01<<8   CC2s    TI2=IC2
TIMx->CCMR2 |= 	TIM_CCMR2_CC3S_0;			//01<<0   CC3s    TI3=IC3
TIMx->CCMR2 |= 	TIM_CCMR2_CC4S_0;	  		//01<<8   CC4s    TI4=IC4

// 2. Filter Duration (use default) do not write yet

// 3. IC Prescaler (use default) do not write yet

// 4. Activation Edge: CCyNP/CCyP	
TIMx->CCER &= ~(7 << (4 * (ICn-1) + 1));					// CCy(Rising) for ICn

// 5.	Enable CCy Capture, Capture/Compare interrupt
TIMx->CCER |= 1 << 4*(ICn - 1);								// CCn(ICn) Capture Enable	

// 6.	Enable Interrupt of CC(CCyIE), Update (UIE)
TIMx->DIER |= 1 << ICn;										// Capture/Compare Interrupt Enable	for ICn
TIMx->DIER |= TIM_DIER_UIE;									// Update Interrupt enable	

// 7.	Enable Counter 
TIMx->CR1	 |= TIM_CR1_CEN;								// Counter enable	
}


// Configure Selecting TIx-ICy and Edge Type
void ICAP_setup(IC_t *ICx, int ICn, int edge_type){
    TIM_TypeDef *TIMx = ICx->timer;	// TIMx
    int 				CHn 	= ICx->ch;		// Timer Channel CHn
    ICx->ICnum = ICn;

    // Disable  CC. Disable CCInterrupt for ICn. 
    TIMx->CCER &= ~(1 << 4*(ICn - 1));							// Capture Disable
    TIMx->DIER &= ~(1 << ICn);									// CCn Interrupt Disable	

    // Configure  IC number(user selected) with given IC pin(TIMx_CHn)
    switch(ICn){
        case 1:
            TIMx->CCMR1 &= ~TIM_CCMR1_CC1S;						//reset   CC1S
            if (ICn==CHn) TIMx->CCMR1 |= 	TIM_CCMR1_CC1S_0;   //01<<0   CC1S    Tx_Ch1=IC1
            else TIMx->CCMR1 |= 	TIM_CCMR1_CC1S_1;      		//10<<0   CC1S    Tx_Ch2=IC1
            break;
        case 2:
            TIMx->CCMR1 &= ~TIM_CCMR1_CC2S;						//reset   CC2S
            if (ICn==CHn) TIMx->CCMR1 |= 	TIM_CCMR1_CC2S_0;   //01<<0   CC2S    Tx_Ch2=IC2
            else TIMx->CCMR1 |= 	TIM_CCMR1_CC1S_1;     		//10<<0   CC2S    Tx_Ch1=IC2
            break;
        case 3:
            TIMx->CCMR2 &= ~TIM_CCMR2_CC3S;						//reset   CC3S
            if (ICn==CHn) TIMx->CCMR2 |= 	TIM_CCMR2_CC3S_0;	//01<<0   CC3S    Tx_Ch3=IC3
            else TIMx->CCMR2 |= TIM_CCMR1_CC1S_1;		     	//10<<0   CC3S    Tx_Ch4=IC3
            break;
        case 4:
            TIMx->CCMR2 &= ~TIM_CCMR2_CC4S;						//reset   CC4S
            if (ICn==CHn) TIMx->CCMR2 |= 	TIM_CCMR2_CC4S_0;	//01<<0   CC4S    Tx_Ch4=IC4
            else TIMx->CCMR2 |= 	TIM_CCMR1_CC1S_1;	     	//10<<0   CC4S    Tx_Ch3=IC4
            break;
        default: break;
    }


    // Configure Activation Edge direction
    TIMx->CCER &= ~(7 << (4 * (ICn-1) + 1));						// Clear CCnNP/CCnP bits for ICn
    switch(edge_type){
        case IC_RISE: TIMx->CCER |= 0 << (4 * (ICn-1) + 1);	 break;							//rising:  00
        case IC_FALL: TIMx->CCER |= 1 << (4 * (ICn-1) + 1);	 break;							//falling: 01
        case IC_BOTH: TIMx->CCER |= 4 << (4 * (ICn-1) + 1) | 1 << (4 * (ICn-1) + 1); break; //both:    11
    }

    // Enable CC. Enable CC Interrupt. 
    TIMx->CCER |= 1 << (4*(ICn - 1)); 								// Capture Enable
    TIMx->DIER |= 1 << ICn; 										// CCn Interrupt enabled	
}

// Time span for one counter step
void ICAP_counter_us(IC_t *ICx, int usec){	
    TIM_TypeDef *TIMx = ICx->timer;	
    TIMx->PSC = 84*usec - 1;					// Timer counter clock: 1us * usec
    TIMx->ARR = 0xFFFF;							// Set auto reload register to maximum (count up to 65535)
}

uint32_t is_CCIF(TIM_TypeDef *TIMx, uint32_t ccNum){
    return (TIMx->SR & 1 << ccNum);	
}

void clear_CCIF(TIM_TypeDef *TIMx, uint32_t ccNum){
    TIMx->SR &= ~(1 << ccNum);	
}
```

##          



### Problem2: Ultrasonic Distance Sensor (HC-SR04)

The HC-SR04 ultrasonic distance sensor. This economical sensor provides 2cm to 400cm of non-contact measurement functionality with a ranging accuracy that can reach up to 3mm. Each HC-SR04 module includes an ultrasonic transmitter, a receiver and a control circuit.

![HC-SR04](https://user-images.githubusercontent.com/91526930/198864049-3dba8f8d-aec8-4f9a-8da3-9adc0fe0e4b9.png)

**The HC-SR04 Ultrasonic Range Sensor Features:**

- Input Voltage: 5V
- Current Draw: 20mA (Max)
- Digital Output: 5V
- Digital Output: 0V (Low)
- Sensing Angle: 30° Cone
- Angle of Effect: 15° Cone
- Ultrasonic Frequency: 40kHz
- Range: 2cm - 400cm

#### Procedure



#### Configuration

![image](https://user-images.githubusercontent.com/113574284/200164417-731c623b-343c-44ff-84b7-30d10a87ffec.png)



#### Circuit Diagram

![image](https://user-images.githubusercontent.com/113574284/200164596-ca685f8d-cbe6-48fb-a89a-c5021c11de14.png)



#### Discussion

1. There can be an over-capture case, when a new capture interrupt occurs before reading the CCR value. When does it occur and how can you calculate the time span accurately between two captures?

   If a capture occurs while the CCxIF flag was  already high, then the over-capture flag CCxOF (TIMx_SR register) is set. CCxIF can be  cleared by software by writing it to ‘0’ or by reading the captured data stored in the  TIMx_CCRx register. CCxOF is cleared when written to ‘0’. 

   Create a new variable, store one value, and calculate the difference with the other one.

   

2. In the tutorial, what is the accuracy when measuring the period of 1Hz square wave? Show your result.

   ![image](https://user-images.githubusercontent.com/113574284/200943630-12ede0a5-e836-428b-9bfa-1435676892ff.png)




#### Description with Code

* LAB_Timer_InputCaputre_Ultrasonic.c  description

```c++
#include "stm32f411xe.h"
#include "math.h"
#include "ecGPIO.h"
#include "ecRCC.h"
#include "ecTIM.h"
#include "ecPWM.h"
#include "ecUART.h"
#include "ecSysTIck.h"

uint32_t ovf_cnt = 0;
float distance = 0;
float timeInterval = 0;
float time1 = 0;
float time2 = 0;

void setup(void);

int main(void){

    setup();

    while(1){
        distance = (float) timeInterval * 340.0 / 2.0 / 10.0; 	// [mm] -> [cm]
        printf("%f [cm]\r\n", timeInterval);
        delay_ms(500);
    }
}

void TIM2_IRQHandler(void){
    if(is_UIF(TIM2)){                     	// Update interrupt
        ovf_cnt++;							// overflow count
        clear_UIF(TIM2);  					// clear update interrupt flag
    }
    if(is_CCIF(TIM2, 3)){ 					// TIM2_Ch3 (IC3) Capture Flag. Rising Edge Detect
        time1 = TIM2->CCR1;					// Capture TimeStart
        clear_CCIF(TIM2, 3);                // clear capture/compare interrupt flag 
    }								                      
    else if(is_CCIF(TIM2, 4)){ 				// TIM2_Ch3 (IC4) Capture Flag. Falling Edge Detect
        time2 = TIM2->CCR1;					// Capture TimeEnd
        // Total time of echo pulse (10us * counter pulses -> [msec] unit)
        timeInterval = ((time2 - time1) + ovf_cnt * (TIM2->ARR +1)) / 1000; 					
        ovf_cnt = 0;                        // overflow reset
        clear_CCIF(TIM2, 4);				// clear capture/compare interrupt flag 
    }
}

void setup(){

    RCC_PLL_init(); 
    SysTick_init();
    UART2_init();

    // PWM configuration ---------------------------------------------------------------------	
    PWM_t trig;							// PWM1 for trig
    PWM_init(&trig, GPIOA, 6);			// PA_6: Ultrasonic trig pulse
    PWM_period_us(&trig, 50000);    	// PWM of 50ms period. Use period_us()
    PWM_pulsewidth_us(&trig, 10);   	// PWM pulse width of 10us

    // Input Capture configuration -----------------------------------------------------------------------	
    IC_t echo;							// Input Capture for echo
    ICAP_init(&echo, GPIOB, 10);    	// PB10 as input caputre
    ICAP_counter_us(&echo, 10);   		// ICAP counter step time as 10us
    ICAP_setup(&echo, 3, IC_RISE);   	// TIM2_CH3 as IC3 , rising edge detect
    ICAP_setup(&echo, 4, IC_FALL);   	// TIM2_CH3 as IC4 , falling edge detect
}
```



#### Results

![KakaoTalk_20221110_062024272](https://user-images.githubusercontent.com/113574284/200944176-b9eab40b-adc6-4975-b19c-3388fe20eb20.jpg)

[Youtube Link](https://youtu.be/x0IUM98kR-M)



##          



### Reference

[LAB: Input Capture - Ultrasonic](https://ykkim.gitbook.io/ec/course/lab/lab-input-capture-ultrasonic)

Class materials in Embedded Controller by Prof. Kim

RM0383 Reference manual

##          



### Troubleshooting

The distance was not measured properly in LAB. The first thing I checked was whether the rising and falling edges took the correct action.
In the tutorial code, it was confirmed that input capture works well in rising and falling. GPIOA's No. 5 pin was used as a capture, and A's No. 6 was set as a PWM signal with a period of 500 ms and confirmed by connecting it.
In addition, when pin 5 of the tutorial GPIOA is used as a capture and number 6 of A is used as a trigger of the sensor, the brightness of the LED connected to number 5 changes depending on the distance, so it is judged that the sensor is also not a problem.
I thought it might be the reason why the code in the conditional sentence could not be executed, so I checked it through LED toggle, and there was no problem with this. 
As the result value did not change depending on the distance, but a constant value came out, so I thought there might be a problem in accessing Timer's CCR, and after checking it through the code, it was confirmed that the CCR value stored to calculate the time difference in the conditional sentence was not properly stored.
Therefore, it was confirmed through a PWM signal, not a sensor, and this time, it was confirmed that the CCR value was properly loaded. However, when the output of the value was checked through the term, it was possible to observe the stutter even though a short delay was given. The reason seems to be the lack of computation in the calculation process inside the interrupt.
The calculation itself is strange because when I added time2-time1 only in the calculation of timeInterval, its result was 0, and the result value of the expression including time2-time1 and the excluded expression is different.
I don't know exactly where it went wrong.
