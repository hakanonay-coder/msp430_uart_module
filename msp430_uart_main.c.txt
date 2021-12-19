#include <msp430.h>
#include <stdio.h>
#include <string.h>

//******************************************************************************
// UART Initialization *********************************************************
//******************************************************************************

#define LED_OUT     P1OUT
#define LED_DIR     P1DIR
#define LED_PIN     BIT0

#define SMCLK_115200    0
#define SMCLK_9600      1

#define UART_MODE      SMCLK_9600  //SMCLK_115200///

unsigned char RXDataBuffer[64];
unsigned char TXDataBuffer[64] = "Hello world\r\n\0";
unsigned char* pTXBuffer = &TXDataBuffer[0];
unsigned char* pRXBuffer = &RXDataBuffer[0];
unsigned int bytesToSend;
unsigned int bytesReceived;

/*
 * UART timing based on 8MHz SMCLK
 */
void initUART()
{
    // Configure USCI_A0 for UART mode
    UCA0CTLW0 |= UCSWRST;                      // Put eUSCI in reset
#if UART_MODE == SMCLK_115200

    UCA0CTLW0 |= UCSSEL__SMCLK;               // CLK = SMCLK
    // Baud Rate Setting
    // Use Table 21-5
    UCA0BRW = 4;
    UCA0MCTL |= UCOS16 | UCBRF_3 | UCBRS_5 ;

#elif UART_MODE == SMCLK_9600

    UCA0CTLW0 |= UCSSEL__SMCLK;               // CLK = SMCLK
    // Baud Rate Setting
    // Use Table 21-5
    UCA0BRW = 52;
    UCA0MCTL |= UCOS16 | UCBRF_1 | UCBRS_0;



#else
    # error "Please specify baud rate to 115200 or 9600"
#endif

    UCA0CTLW0 &= ~UCSWRST;                    // Initialize eUSCI
    UCA0IE |= UCRXIE;                         // Enable USCI_A0 RX interrupt
}

//******************************************************************************
// Device Initialization *******************************************************
//******************************************************************************

void initGPIO()
{
    // Configure GPIO
    P3SEL &= ~(BIT3 | BIT4);                 // USCI_A0 UART operation on FF5529LP
    P3SEL |= (BIT3 | BIT4);
}

void initClockTo16MHz()
{

    UCSCTL3 = SELREF_2;                       // Set DCO FLL reference = REFO
    UCSCTL4 |= SELA_2;                        // Set ACLK = REFO
    UCSCTL0 = 0x0000;                         // Set lowest possible DCOx, MODx

    // Loop until XT1,XT2 & DCO stabilizes - In this case only DCO has to stabilize
    do
    {
    UCSCTL7 &= ~(XT2OFFG + XT1LFOFFG + DCOFFG);
                                            // Clear XT2,XT1,DCO fault flags
    SFRIFG1 &= ~OFIFG;                      // Clear fault flags
    }while (SFRIFG1&OFIFG);                   // Test oscillator fault flag

    __bis_SR_register(SCG0);                  // Disable the FLL control loop
    UCSCTL1 = DCORSEL_5;                      // Select DCO range 16MHz operation
    UCSCTL2 |= 249;                           // Set DCO Multiplier for 8MHz
                                              // (N + 1) * FLLRef = Fdco
                                              // (249 + 1) * 32768 = 8MHz
    __bic_SR_register(SCG0);                  // Enable the FLL control loop

    // Worst-case settling time for the DCO when the DCO range bits have been
    // changed is n x 32 x 32 x f_MCLK / f_FLL_reference. See UCS chapter in 5xx
    // UG for optimization.
    // 32 x 32 x 8 MHz / 32,768 Hz = 250000 = MCLK cycles for DCO to settle
    __delay_cycles(250000);
}

void TransmitData(unsigned char* string, unsigned int length)
{
    pTXBuffer = string;
    bytesToSend = strlen((const char*) string);
    UCA0IE |= UCTXIE;                         // Enable USCI_A0 TX interrupt

}

int main(void)
{
    WDTCTL = WDTPW | WDTHOLD;                               // Stop watchdog timer
    initGPIO();
    initClockTo16MHz();
    initUART();
    int i;
    int data[9];
    /*
    * Use timer to demonstrate automatic tranmission.
    * Configure timer to periodically transmit a string.
    */
    TA0CCTL0 |= CCIE;                                      // TACCR0 interrupt enabled
    TA0CCR0 = 50000;
    TA0CTL |= TASSEL__SMCLK | MC__CONTINUOUS | ID__8;     // SMCLK, continuous mode
    for(i=0; i<11;i++){
    while(!(UCA0IFG&UCRXIFG));          //waits for character
           UCA0IFG&=~ UCRXIFG;
           data[i]=UCA0RXBUF;

    }                               //sets first element of user name equal to first character received from transmit buffer
    __bis_SR_register(GIE);

    while (1)
    {
        __bis_SR_register(LPM0_bits | GIE);               // Enter LPM0
        __no_operation();                                   // For debugger
    }
}

#pragma vector=USCI_A0_VECTOR
__interrupt void USCI_A0_ISR(void)
{
    switch(__even_in_range(UCA0IV,4))
    {
        case 0: break;
        case 2:
              UCA0IFG &=~ UCRXIFG;              // Clear interrupt
              *pRXBuffer = UCA0RXBUF;             // Clear buffer
              break;
        case 4:
            UCA0TXBUF = *pTXBuffer++;
            if(--bytesToSend == 0)
                UCA0IE &= ~UCTXIE;            // Disable USCI_A0 TX interrupt after string has been transmitted.
            break;
        default:
            break;

    }
    __bic_SR_register_on_exit(LPM0_bits); // Exit LPM0 on reti
}

// Timer A0 interrupt service routine

#pragma vector = TIMER0_A0_VECTOR
__interrupt void Timer_A (void)
{
    static int count = 10;
    if(--count == 0)
    {
        count = 10;
        TransmitData(&TXDataBuffer[0], sizeof(TXDataBuffer));
        _no_operation();

    }
}
