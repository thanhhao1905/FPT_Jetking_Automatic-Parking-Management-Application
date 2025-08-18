````C
#include <reg51.h>

#define LCD_PORT P0  
#define PWM_PERIOD 0xB7FE
#define MAX_SLOTS 10 

// I/O
sbit senin = P1^0;
sbit senout = P1^1;
sbit signin = P1^2;
sbit signout = P1^3;
sbit camdata = P1^4;
sbit servoPinEntry = P1^6;
sbit servoPinExit = P1^7;
sbit rs = P2^6;
sbit rw = P2^5;
sbit en = P2^7;
sbit ESP32_Signal = P2^0;
sbit ESP32_Signal1 = P2^1;

volatile unsigned int count = 5;
unsigned int digit[3];
volatile unsigned int ON_Period, OFF_Period;
volatile float period;
bit servo_select = 0; 
int angle;

// LCD
void lcdcmd(char);
void lcdinit();
void lcddata(char);
void lcdstring(char *);

// delay
void delay(unsigned int);
void delay_ms(unsigned int);

// UART
void UART_Init();
char UART_ReceiveChar_Timeout(unsigned int);
void processUARTCommand(char);

// Timer và Servo
void Timer_init();
void Set_DutyCycle(float);
float AngleToDutyCycle(int);

// Door
void entry_door_open();
void entry_door_close();
void exit_door_open();
void exit_door_close();


void display_count(unsigned int);


void UART_Init() {
    TMOD &= 0x0F;    
    TMOD |= 0x20;     
    TH1 = 0xFD;       
    SCON = 0x50;   
    TR1 = 1;          
    TI = 1;         
}


char UART_ReceiveChar_Timeout(unsigned int timeout_ms) {
    unsigned int i, j;
    for(i = 0; i < timeout_ms; i++) {
        if(RI) {
            RI = 0;
            return SBUF;
        }
        for(j = 0; j < 112; j++); // ~1ms delay
    }
    return 0;
}


void processUARTCommand(char cmd) {
    camdata = (cmd == '1') ? 1 : 0;
}


void Timer_init() {
    TMOD &= 0xF0;    
    TMOD |= 0x01;   
    TH0 = (PWM_PERIOD >> 8);
    TL0 = PWM_PERIOD & 0xFF;
    TR0 = 1;
}


void Timer0_ISR() interrupt 1 {
    if(servo_select == 0) {
        servoPinEntry = !servoPinEntry;
        if(servoPinEntry) {
            TH0 = (ON_Period >> 8);
            TL0 = ON_Period & 0xFF;
        } else {
            TH0 = (OFF_Period >> 8);
            TL0 = OFF_Period & 0xFF;
        }
    } else {
        servoPinExit = !servoPinExit;
        if(servoPinExit) {
            TH0 = (ON_Period >> 8);
            TL0 = ON_Period & 0xFF;
        } else {
            TH0 = (OFF_Period >> 8);
            TL0 = OFF_Period & 0xFF;
        }
    }
}


float AngleToDutyCycle(int angle) {
    return 5.0 + ((float)angle * 5.0 / 180.0);
}


void Set_DutyCycle(float duty_cycle) {
    period = 65535 - PWM_PERIOD;
    ON_Period = (period * duty_cycle) / 100.0;
    OFF_Period = period - ON_Period;
    ON_Period = 65535 - ON_Period;
    OFF_Period = 65535 - OFF_Period;
}


void display_count(unsigned int value) {
    int i;
    for(i = 0; i < 3; i++) {
        digit[i] = value % 10;
        value /= 10;
    }
    for(i = 2; i >= 0; i--) {
        lcddata(digit[i] + '0');
        delay_ms(5);
    }
}


void lcdcmd(char value) {
    LCD_PORT = value;
    rs = 0; rw = 0; en = 1;
    delay(500);
    en = 0;
}

void lcddata(char value) {
    LCD_PORT = value;
    rs = 1; rw = 0; en = 1;
    delay(500);
    en = 0;
}

void lcdstring(char *p) {
    while(*p) {
        lcddata(*p++);
        delay_ms(1);
    }
}

void lcdinit() {
    lcdcmd(0x38); delay(500);
    lcdcmd(0x01); delay(500);
    lcdcmd(0x0C); delay(500);
    lcdcmd(0x80); delay(500);
    lcdcmd(0x0E); delay(500);
}


void delay(unsigned int x) {
    while(x--);
}


void delay_ms(unsigned int ms) {
    unsigned int i, j;
    for(i = 0; i < ms; i++)
        for(j = 0; j < 120; j++);
}


void entry_door_open() {
    lcdcmd(0xC0); lcdstring("OPENING DOOR");
    EA = 1; ET0 = 1; Timer_init();
    for(angle = 0; angle <= 160; angle += 10) {
        Set_DutyCycle(AngleToDutyCycle(angle));
        delay_ms(20);
    }
    ET0 = 0;
}


void entry_door_close() {
    lcdcmd(0xC0); lcdstring("CLOSING DOOR");
    EA = 1; ET0 = 1; Timer_init();
    for(angle = 160; angle >= 0; angle -= 10) {
        Set_DutyCycle(AngleToDutyCycle(angle));
        delay_ms(20);
    }
    ET0 = 0;
}


void exit_door_open() {
    lcdcmd(0xC0); lcdstring("EXIT OPENING...");
    EA = 1; ET0 = 1; Timer_init();
    for(angle = 160; angle >= 0; angle -= 10) {
        Set_DutyCycle(AngleToDutyCycle(angle));
        delay_ms(20);
    }
    ET0 = 0;
}


void exit_door_close() {
    lcdcmd(0xC0); lcdstring("EXIT CLOSING");
    EA = 1; ET0 = 1; Timer_init();
    for(angle = 0; angle <= 160; angle += 10) {
        Set_DutyCycle(AngleToDutyCycle(angle));
        delay_ms(20);
    }
    ET0 = 0;
}

void main() {
    lcdinit();
    UART_Init();
    senin = senout = 0;
    signin = signout = 1;
    camdata = 0;
    ESP32_Signal = ESP32_Signal1 = 0;

    lcdcmd(0x01);
    lcdstring("HELLO SIR");
    delay_ms(500);

    lcdcmd(0x01);
    lcdstring("WELCOME TO CAR");
    lcdcmd(0xC0);
    lcdstring("PARKING SYSTEM");
    delay_ms(500);

    lcdcmd(0x01);
    lcdstring("TOTAL CARS:");
    lcdcmd(0x8D);
    display_count(count);

    while(1) {
        lcdcmd(0xC0);
        lcdstring("CHECK ID        ");


        while(senin == 0 && senout == 0 && ESP32_Signal == 0 && ESP32_Signal1 == 0);

        if(ESP32_Signal || ESP32_Signal1) {
            lcdcmd(0xC0);
            lcdstring("WRONG ID        ");
            delay_ms(2000);
            lcdcmd(0xC0);
            lcdstring("TURN RIGHT ==>> ");
            delay_ms(2000);
            while(senin == 0 && senout == 0 && (ESP32_Signal || ESP32_Signal1));
            continue;
        }


        if(senin && signin) {
            if(count >= MAX_SLOTS) {
                lcdcmd(0x01);
                lcdstring("FULL CAR");
                delay_ms(3000);
                continue;
            }
            count++;
            lcdcmd(0x8D);
            display_count(count);
            servo_select = 0;
            entry_door_open();
            while(senin);       
            while(signin == 1);
            while(signin == 0);
            delay_ms(2000);
            entry_door_close();
        }

        else if(senout && signout) {
            char received = UART_ReceiveChar_Timeout(2000);
            if(received) processUARTCommand(received);
            else camdata = 0;

            if(camdata) {
                if(count > 0) count--;
                lcdcmd(0x01);
                lcdstring("EXIT ID CONFIRMED");
                delay_ms(1000);
                lcdcmd(0x01);
								lcdstring("TOTAL CARS:");
								lcdcmd(0x8D);
								display_count(count);
                delay_ms(2000);
                servo_select = 1;
                exit_door_open();
                while(senout);
                while(signout == 1);
                while(signout == 0);
								delay_ms(2000);
								lcdcmd(0x01);
                lcdstring("TOTAL CARS:");
								lcdcmd(0x8D);
								display_count(count);
                exit_door_close();
                camdata = 0;
            } else {
                lcdcmd(0x01);
                lcdstring("WRONG NUMBER");
                delay_ms(2000);
                lcdcmd(0x01);
                lcdstring("TOTAL CARS:");
                lcdcmd(0x8D);
                display_count(count);
            }
        }
    }
}
