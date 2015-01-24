//COde for UHF RFID Passive Reader

#include<reg51.h>
#include<string.h>
//0000 to 7FFF

sbit RS = P3^2;
sbit EN = P3^3;

sbit SDA = P1^0;
sbit SCL = P1^1;

sbit OK_LED = P1^2;
sbit NOT_OK_LED = P1^3;

code unsigned char RFID_1[] = "76002C4D3324";
code unsigned char RFID_2[] = "76002C4D3126";
code unsigned char RFID_3[] = "76002C4D3027";
code unsigned char RFID_4[] = "76002C4D2532";
code unsigned char RFID_5[] = "420061231E1E";
code unsigned char RFID_6[] = "76002C4D3324";
code unsigned char RFID_7[] = "76002C4D3126";
code unsigned char RFID_8[] = "76002C4D3027";
code unsigned char RFID_9[] = "76002C4D2532";
code unsigned char RFID_10[] = "420061231E1E";

code unsigned char name_1[] = "Package_1 OK";
code unsigned char name_2[] = "Package_2 OK";
code unsigned char name_3[] = "Package_3 OK";
code unsigned char name_4[] = "Package_4 OK";
code unsigned char name_5[] = "Package_5 OK";
code unsigned char name_6[] = "Package_6 OK";
code unsigned char name_7[] = "Package_7 OK";
code unsigned char name_8[] = "Package_8 OK";
code unsigned char name_9[] = "Package_9 OK";
code unsigned char name_10[] = "Package_10 OK";

unsigned char rs[15];

unsigned int no_of_records;

void delay()
{
int i;
for(i=0;i<500;i++);
}

void long_delay()
{
unsigned int i;
for(i=0;i<65000;i++);
}

void idelay()
{
unsigned int i;
for(i=0;i<10000;i++);
}

void lcd_command(char lc)
{
P2 = lc;
RS = 0;
EN = 1;
delay();
EN = 0;
}

void lcd_data(char ld)
{
P2 = ld;
RS = 1;
EN = 1;
delay();
EN = 0;
}

void lcd_init()
{
lcd_command(0x38);
lcd_command(0x0E);
lcd_command(0x01);
}

void serial_init()
{
SCON = 0x50;
TMOD = 0x20;
TH1 = 0xFD;
TR1 = 1;
}

void transmit(unsigned char tx)
{
SBUF = tx;
while(TI==0);
TI = 0;
}

void send_string(unsigned char *str)
{
int i;
for(i=0;str[i]!='\0';i++)
transmit(str[i]);
}

unsigned char receive()
{
char rx;
while(RI==0);
RI = 0;
rx = SBUF;
return(rx);
}

void lcd_string(char add,char *str)
{
int i;
lcd_command(add);
for(i=0;str[i]!='\0';i++)
lcd_data(str[i]);
}

void start()
{
SDA = 1;
SCL = 1;
SDA = 0;
}

void stop()
{
SDA = 0;
SCL = 1;
SDA = 1;
}

void write(unsigned char w)
{
int i;
SCL = 0;
for(i=0;i<8;i++)
{
if((w & 0x80)==0)
SDA = 0;
else
SDA = 1;
SCL = 1;
SCL = 0;
w = w << 1;
}
SCL = 1;
SCL = 0;
}

unsigned char read()
{
int i;
unsigned char r = 0x00;
SDA = 1;

for(i=0;i<8;i++)
{
SCL = 1;
r = r << 1;
if(SDA == 1)
r = r | 0x01;
SCL = 0;
}
return(r);
}

void ack()
{
SDA = 0;
SCL = 1;
SCL = 0;
}

void nack()
{
SDA = 1;
SCL = 1;
SCL = 0;
}

void rtc_read()
{
unsigned char ss,mm,hh,day,mn,date,yr;
start();
write(0xD0);
write(0x00);
stop();
start();
write(0xD1);
ss = read();
ack();
mm = read();
ack();
hh = read();
ack();
day = read();
ack();
date = read();
ack();
mn = read();
ack();
yr = read();
nack();
stop();

rs[0] = hh/0x10 + 48;
rs[1] = hh%0x10 + 48;
rs[2] = ':';
rs[3] = mm/0x10 + 48;
rs[4] = mm%0x10 + 48;
rs[5] = ',';
rs[6] = date/0x10 + 48;
rs[7] = date%0x10 + 48;
rs[8] = '/';
rs[9] = mn/0x10 + 48;
rs[10] = mn%0x10 + 48;
rs[11] = '/';
rs[12] = yr/0x10 + 48;
rs[13] = yr%0x10 + 48;
rs[14] = '\0';
}

void rtc_init()
{
start();
write(0xD0);
write(0x00);
write(0x00);
write(0x00);
write(0x13);
write(0x05);
write(0x12);
write(0x04);
write(0x12);
stop();

}

void write_records(unsigned char *str);
void read_records();

void main()
{
unsigned char rec_data[13],i,t;

OK_LED = 1;

lcd_init();
serial_init();
rtc_init();
idelay();
start();
write(0xA0);
write(0x7F);
write(0xFF);
stop();
start();
write(0xA1);
no_of_records = read();
nack();
stop();

// no_of_records = 0;

while(1)
{
start:
lcd_command(0x01);
lcd_string(0x81,"RFID DISPATCH");
lcd_string(0xC0,"CHECKING DEVICE");

i = 0;
while(1)
{
if(RI==1)
{
RI = 0;
t = receive();
if(t == '+')
{
read_records();
goto start;
}
else
{
rec_data[i] = t;
for(i=1;i<12;i++)
rec_data[i] = receive();
rec_data[i] = '\0';
break;
}
}
}

i = strcmp(RFID_1,rec_data); //match => i = 0

lcd_command(0x01);

if(i==0)
{
OK_LED = 0;
lcd_string(0x80,name_1);
rtc_read();
lcd_string(0xC0,rs);
long_delay();
write_records(name_1);
long_delay();
OK_LED = 1;
goto start;
}

//
i = strcmp(RFID_2,rec_data); //match => i = 0

if(i==0)
{
OK_LED = 0;
lcd_string(0x80,name_2);
rtc_read();
lcd_string(0xC0,rs);
long_delay();
write_records(name_2);
long_delay();
OK_LED = 1;
goto start;
}

//
i = strcmp(RFID_3,rec_data); //match => i = 0

if(i==0)
{
OK_LED = 0;
lcd_string(0x80,name_3);
rtc_read();
lcd_string(0xC0,rs);
long_delay();
write_records(name_3);
long_delay();
OK_LED = 1;
goto start;
}

i = strcmp(RFID_4,rec_data); //match => i = 0

if(i==0)
{
OK_LED = 0;
lcd_string(0x80,name_4);
rtc_read();
lcd_string(0xC0,rs);
long_delay();
write_records(name_4);
long_delay();
OK_LED = 1;
goto start;
}

i = strcmp(RFID_5,rec_data); //match => i = 0

if(i==0)
{
OK_LED = 0;
lcd_string(0x80,name_5);
rtc_read();
lcd_string(0xC0,rs);
long_delay();
write_records(name_5);
long_delay();
OK_LED = 1;
goto start;
}

i = strcmp(RFID_6,rec_data); //match => i = 0

if(i==0)
{
OK_LED = 0;
lcd_string(0x80,name_6);
rtc_read();
lcd_string(0xC0,rs);
long_delay();
write_records(name_6);
long_delay();
OK_LED = 1;
goto start;
}

i = strcmp(RFID_7,rec_data); //match => i = 0

if(i==0)
{
OK_LED = 0;
lcd_string(0x80,name_7);
rtc_read();
lcd_string(0xC0,rs);
long_delay();
write_records(name_7);
long_delay();
OK_LED = 1;
goto start;
}

i = strcmp(RFID_8,rec_data); //match => i = 0

if(i==0)
{
OK_LED = 0;
lcd_string(0x80,name_8);
rtc_read();
lcd_string(0xC0,rs);
long_delay();
write_records(name_8);
long_delay();
OK_LED = 1;
goto start;
}

i = strcmp(RFID_9,rec_data); //match => i = 0

if(i==0)
{
OK_LED = 0;
lcd_string(0x80,name_9);
rtc_read();
lcd_string(0xC0,rs);
long_delay();
write_records(name_9);
long_delay();
OK_LED = 1;
goto start;
}

i = strcmp(RFID_10,rec_data); //match => i = 0

if(i==0)
{
OK_LED = 0;
lcd_string(0x80,name_10);
rtc_read();
lcd_string(0xC0,rs);
long_delay();
write_records(name_10);
long_delay();
OK_LED = 1;
goto start;
}
/* i = strcmp(RFID_5,rec_data); //match => i = 0

if(i==0)
{
OK_LED = 0;
lcd_string(0x80,name_5);
no_of_records = 0;
start();
write(0xA0);
write(0x7F);
write(0xFF);
write(0x00);
stop();
lcd_string(0xC0,"MEMORY CLEARED");
long_delay();
OK_LED = 1;
goto start;
} */
lcd_string(0x80,"ERROR");
NOT_OK_LED = 0;
lcd_string(0xC0,name_3);
long_delay();
NOT_OK_LED = 1;
long_delay();
}
}
