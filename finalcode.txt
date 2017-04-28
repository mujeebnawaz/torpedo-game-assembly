;Stack and Stack Pointer Addresses 
.equ     SPH    =$3E                  ;High Byte Stack Pointer Address 
.equ     SPL    =$3D                  ;Low Byte Stack Pointer Address 
.equ     RAMEND =$25F                 ;Stack Address 

;Ship Lights Hex
.equ    RDESTROYER    =$1             ;initiates 00000001 
.equ    RBATTLESHIP   =$3             ;initiates 00000011
.equ    RAIRCRAFT     =$7             ;initiates 00000111

.equ    LDESTROYER    =$80            ;initiates 10000000
.equ    LBATTLESHIP   =$C0            ;initiates 11000000
.equ    LAIRCRAFT     =$E0            ;initiates 11100000

;Score Definitions
.equ    DESTROYER    =$0C             ;initiates 12 points
.equ    BATTLESHIP   =$08             ;initiates 08 points
.equ    AIRCRAFT     =$04             ;initiates 04 points


;Input and Output Port Addresses
.equ    PORTA   =$1B                    ;Port A Output Address
.equ    DDRA    =$1A                    ;Port A Data Direction Address
.equ    PINA    =$19                    ;Port A Input Address

.equ    PORTB   =$18			;Port B Output Address
.equ    DDRB    =$17			;Port B Data Direction Address
.equ    PINB    =$16			;Port B Input Address

.equ    PORTC   =$15			;Port C Output Address
.equ    DDRC    =$14			;Port C Data Direction Address
.equ    PINC	=$13			;Port C Input Address

.equ    PORTD   =$12		  	;Port D Output Address
.equ    DDRD    =$11	                ;Port D Data Direction Address
.equ    PIND    =$10			;Port D Input Address

.def     ZL     =r30                    ;Define low byte of Z 
.def     ZH     =r31                    ;Define high byte of Z 


;General Purpose Registers
.def    temp    =r16

 
;Random Number Registers
.def    seed    =r17

.def    lives   =r9

;LEDs
.def    led    =r19


.def    count  =r27



;Score
.def    totalscore =r11
.def    tempscore  =r28
.def    scoredisplay   =r0


.def    count1 = r29

;Speed register, stores speed of the delay
.def    speed = r18



;Set stack pointer to end of memory 
       ldi      temp,high(RAMEND) 
       out      SPH,temp          ;Load high byte of end of memory address 
       ldi      temp,low(RAMEND) 
       out      SPL,temp          ;Load low byte of end of memory address 


      



;Main Program

       rcall    newgame
starta:
       rcall    reada

startb:
       rcall    start

startc: 
      dec       r9 
      rcall     random
      rcall     assign
      rcall     getship
      rcall     walk    
      rjmp      startb

; End Main Program


resetgame: 
      ldi       temp,$FF
      out       PORTC,temp
      rjmp      resetgame


newgame:
       ldi      temp,$03
       add      lives,temp
       ldi      speed,$FF
	 
;Initialise Input Port
       ldi      temp,$00
       out      DDRA,temp
   
;Initialise Output Port
       ldi      temp,$FF
       out      DDRC,temp
       out      DDRD,temp
	   out      DDRB,temp
       ret



/**
Following code generates a random number ranged (0 - 255)

using PseudoRandom algorithm.

Random number generator requires a see (initial number) in order
to generate random numbers. Seed is given by user from 0 - 7 push buttons 
with random probablity of 1:8. 
*/

reada:
       in       seed,PINA    ;Stores input on r17
       com      seed         ;Flips the bit
       cpi      seed,00      ;Compares seed from 00 or null
       brne     start        
       rjmp     reada        
       ret	

start: 
       ldi      r20,$5
       ldi      count,$00  
       ret

random:
       ldi      r21, $01     ;initiates r21 to 00000001 for bit extracting purpose
       ldi      r22, $02     ;initiates r22 and r23 to 00000010 for bit extracting purpose
       ldi      r23, $02
       and      r21,seed     ;Extract the bits and stores the results on r21,r22 and r23
       and      r22,seed
       lsl      r21
       eor      r21,r22      ;Exclusive ORs
       and      r21,r23
       lsr      seed     
       lsl      r21 
       lsl      r21
       lsl      r21
       lsl      r21
       lsl      r21
       lsl      r21
       or       seed,r21     ;Joins the bits
       dec      r20          ;Loops over 5 times at start using r20
       brne     random
       ldi      r26,$00
       add      r26,seed     ;Transfers seed to r26
       ret
       
	   
assign:      
       ldi      r24,$2A 
       ldi      r25,$2A      ;Initiates r24,r25 with 42 since there are 6 ships (255/6)

/*
Assigns the ships as following:
1. Randomly generated number < 42
   Assigns Destroyer with right direction

2. Randomly generated number < 84  AND > 42
   Assigns Battleship with left direction

3. Randomly generated number < 126 AND > 84
   Assigns Aircraft with right direction

4. Randomly generated number < 168 AND > 126
   Assigns Destroyer from left directions

5. Randomly generated number < 210 AND > 168
   Assigns Battleship with right direction

6. Randomly generated number < 255 AND > 210
   Assigns Aircraft with left direction

   Uses a counter for assesing the random number
   Program automatically stores the relevant ship score to a register

*/


subassign:
       inc      count            
       cp       r26,r24
       brlo     getship
       add      r24,r25
       cpi      count,06
       brne     subassign
       ret

getship:
       cpi      count,01
       brne     b
       ldi      led,RDESTROYER          ;Loads the ship to led (r19)
       ldi      tempscore,DESTROYER     ;Load the score to tempscore (r28)
       ret

b:	
       cpi      count,02
       brne     c
       ldi      led,LBATTLESHIP
       ldi      tempscore,BATTLESHIP
       ret

c:
       cpi      count,03
       brne     d
       ldi      led,RAIRCRAFT
       ldi      tempscore,AIRCRAFT
       ret

d:
       cpi      count,04
       brne     e
       ldi      led,LDESTROYER
       ldi      tempscore,DESTROYER
       ret  
    
e:	
       cpi      count,05
       brne     f
       ldi      led,RBATTLESHIP
       ldi      tempscore,BATTLESHIP
       ret
       

f:	
       cpi       count,06
       ldi       led,LAIRCRAFT
       ldi       tempscore,AIRCRAFT
       ret
	   

/*
Walk is a subroutine which is responsible for moving the 
ship from left to right based on its current position.

If Ship is on right, it will move it to left and vise versa.
Each walks calls a polling (torpedo) to check if user have hit a ship.

Walk also calls a speed delay, which is dynamic to the level user is on.
I.e. if user is on the first level delay will be long, but if user is on 
last level the delay will be extremely short.

*/  
walk:  
       ldi        r22,$08
       ldi        r21,$01
       and        r21,led
       cpi        r21,$01
       brne       r
       rjmp       leftwalk
   
r: 
       ldi        r21,$80
       and        r21,led
       cpi        r21,$80
       breq       rightwalk
       rjmp       walk

rightwalk:  
       ldi        temp,$00
       cp         lives,temp
       breq       walkreset
       out        PORTC,led      ;Outputs ship to PORTC
       rcall      speeddelay     ;Calls speed delay
       rcall      torpedo        ;Polls switch
       rcall      speeddelay
       lsr        led            ;Moves ship to right
       dec        r22
       cpi        r22,00         ;Moves ships to right unless it disappears
       brne       rightwalk
       ret
	  

leftwalk:
       ldi        temp,$00
       cp         lives,temp
       breq       walkreset
       out        PORTC,led
       rcall      speeddelay
       rcall      torpedo
       rcall      speeddelay
       lsl        led
       dec        r22
       cpi        r22,00
       brne       leftwalk
       ret

walkreset: 

       rjmp       resetgame

torpedo:
       in         r24,PINA      ;Takes input from PINA, stores it to r24
       com        r24           ;Flip the bits
       and        r24,led       ;Extracts the bits for button and led matching
       cpi        r24,$00       ;If extraction does not results in zero, scores
       brne       score
       ret

/*
Game keeps track of ships, if user mises three ships in a row, game resets itself
otherwise if user scores game maintain the lives/chances to the same value.

I.e. if user misses a ship lives will become 2 from 3 but then if user keeps scoring
lives will remain 2.

Furthermore, Score also increases the speed of the game by decreasing the delay.
Finally it sounds the buzzer and display the score on seven segment display.
*/

score: 
       inc        lives         ;Increases the lives (if user does not)
       subi       speed,$17     ;Decreases the speed loop.
       ldi        count,$0A

beep:
       ldi        temp,$0F
       out        PORTD,temp
       rcall      beepdelay
       ldi        temp,$00
       out        PORTD,temp 
       rcall      beepdelay
       dec        count
       cpi        count,$00
       brne       beep
       rjmp       displayscore
	    


displayscore:
      ldi     r24,$00
      ldi     r25,$00
      ldi     count,$00  
      add     totalscore,tempscore
      mov     tempscore,totalscore
      ldi     temp,$01
      add     tempscore,temp

reset1:    
         clr     count             ;Set table position counter to zero 

next1:             
         ldi     ZL,low(table*2)   ;Set Z pointer to start of table 
         ldi     ZH,high(table*2)
                                    
	                             
         add    ZL,r25               
         lpm
         ldi    r23,$00
         add    r23,scoredisplay 
         inc    count
         inc    r25             ;Increment table position counter 
         cpi    count,16        ;and test if end of table has been reached 
         brne   reset2          ;if not the end of the table, get next data value in table 
         rjmp   reset2  
		         

reset2:  
 
         ldi    ZL,low(table*2)    
         ldi    ZH,high(table*2) 
         clr    count1 
         ldi    count1,16
         adiw   ZL,16            
next2:  
         cpi    tempscore,00
         breq   displaydelay
         dec    tempscore               
         lpm       
         ldi    r22,$00          
         add    r22,scoredisplay   
         ldi    r24,$01      
         add    ZL,r24 
         dec    count1             
         cpi    count1,00
		                 
         brne   next2 
         rjmp   next1
 
displaydelay: 
	ldi     count,$07 

sia:
	ldi     r24,$FF       
	ldi     r25,$FF  

showit:
        rcall   scoredelay   
    	out      PORTB,r23      ;Outputs the Most significant character
    	rcall    scoredelay
     	out      PORTB,r22      ;Outputs the Least significant character
		

		dec      r25
		brne     showit
		dec      r24 
		brne     s
	s:	
        dec      count
	    cpi      count,00
		breq     yu
		brne     sia
	yu: 
        ldi      r23,$00
        ldi      r22,$00
        rcall    scoredelay
    	out      PORTB,r23 
    	rcall    scoredelay
     	out      PORTB,r22  
        rjmp     startb
   


	     
table:   .DB $3F,$06,$5B,$4F,$66,$6D,$7D,$07,$7F,$6F,$77,$7C,$39,$5E,$79,$71,$BF,$86,$DB,$CF,$E6,$ED,$FD,$87,$FF,$EF,$F7,$FC,$B9,$DE,$F9,$F1


delay:
         ldi    r24,$FF        ;Initialise 2nd loop counter 
delayloop2:   
         ldi    r25,$FF        ;Initialise 1st loop counter 
delayloop1: 
         dec    r25            ;Decrement the 1st loop counter  
		 brne   delayloop1     ;and continue to decrement until 1st loop counter = 0 
         dec    r24            ;Decrement the 2nd loop counter 
         brne   delayloop2     ;If the 2nd loop counter is not equal to zero repeat the 1st loop, else continue 
         ret
beepdelay:   

       ldi        r24,$0F      ;Initialise 2nd loop counter 
beeploop2:
     
       ldi        r25,$0F      ;Initialise 1st loop counter 

beeploop1:   
       dec        r25          ;Decrement the 1st loop counter 
       brne       beeploop1    ;and continue to decrement until 1st loop counter = 0 
       dec        r24          ;Decrement the 2nd loop counter 
       brne       beeploop2    ;If the 2nd loop counter is not equal to zero repeat the 1st loop, else continue 
       ret                     ;Return




scoredelay:  
       ldi        r24,$FF    ;Initialise decrement counter 
scoreloop1:  
       dec        r24        ;Decrement counter 
       brne       scoreloop1 ;and continue to decrement until counter = 0 
       ret                   ;Return





speeddelay:   
       cpi        speed,$30
	   breq       reset
       mov        r24,speed       ;Initialise 2nd loop counter 

speedloop2:     
       mov        r25,speed       ;Initialise 1st loop counter 

speedloop1:   
       dec        r25              ;Decrement the 1st loop counter 
       brne       speedloop1       ;and continue to decrement until 1st loop counter = 0 
       dec        r24              ;Decrement the 2nd loop counter 
       brne       speedloop2       ;If the 2nd loop counter is not equal to zero repeat the 1st loop, else continue 
       ret                         ;Return



reset: rjmp resetgame


