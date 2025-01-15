# Senzor-kvality-a-teploty-ovzdu-ia
Tento depozitár bude obsahovať projekt (RŠ -učiteľstvo INF) ku skúške z predmetu "Internet vecí".  

// kód 

#include <Wire.h>//kniznica i2c
//kniznica pre displej
#include"lcd_st7567s.h"
lcd_st7567s Lcd;
//kniznice pre senzory 
#include <Adafruit_Sensor.h>
#include <Adafruit_BME680.h>
#include <Adafruit_AHT10.h>

//defincie pre senzory
#define SEALEVELPRESSURE_HPA (1013.25)
Adafruit_BME680 bme(&Wire); // I2C
Adafruit_AHT10 aht;

//zadefinovanie funkcii pre triedenie stavov
int state135();
int state7();
int state2();
int state9();
int sbme680R();

//zadefinovanie pomocnych premenych
double MQ135=0,MQ9=0,MQ2=0,MQ7=0,BME680;
int s135,s9,s2,s7,sbme680,zaciatok=0,stav=0;
unsigned long endTime=0;
char buf[10];


void setup() {
  Serial.begin(115200);//inicializacia seriovej linky
  //inicializacia a vypis "uvodnej strany"
  Lcd.Init();
  Lcd.Clear(false);
  Lcd.Cursor(0, 0);
  Lcd.Display("Senzor kvality a");
  Lcd.Cursor(0, 1);
  Lcd.Display("Teploty ovzdusia");
  Lcd.Cursor(0, 2);
  Lcd.Display("  Projekt IVE");
  Lcd.Cursor(0, 3);
  Lcd.Display("  Matus Rusnak");
  Lcd.Cursor(0, 7);
  Lcd.Display("Made in Slovakia");
  //inicializacia senzorov na i2c
  if (!bme.begin()) {
    Serial.println("doslo ku chybe z bme!");
    while (1);
  }
  if (! aht.begin()) {
    Serial.println("doslo ku chybe aht");
    while(1) delay(10);
  }
  delay(5000);//aby bolo vydno uvodnu obrazovku
  //inicializacia tlacidiel 
  pinMode(14, INPUT_PULLUP);
  pinMode(13, INPUT_PULLUP);
  pinMode(12, INPUT_PULLUP);
  pinMode(11, INPUT_PULLUP);
  pinMode(10, INPUT_PULLUP);
  //inicializacia rozlisenia adc prevodnika
  analogReadResolution(12);
  //zadefinovanie bme sravania 
  bme.setTemperatureOversampling(BME680_OS_8X);
  bme.setHumidityOversampling(BME680_OS_2X);
  bme.setPressureOversampling(BME680_OS_4X);
  bme.setIIRFilterSize(BME680_FILTER_SIZE_3);
  bme.setGasHeater(320, 150); 
  //inicializacia pwm pre ventilator
  ledcAttach(26, 12000, 8);
} 

/////////////////////////////////////////////////////////
void loop() {
  //citanie senzorov i2c
  endTime = bme.beginReading();
  sensors_event_t humidity, temp;
  aht.getEvent(&humidity, &temp);
  if (! bme.performReading()) {
    Serial.println("nepodarilo sa precitat hodnoty senzora bme");
    return;
  }

  //vypis obrazovky "0"
  if(!digitalRead(10)||zaciatok==0){
    zaciatok=1;
    stav=0;
    Lcd.Clear(false);
    Lcd.Cursor(0, 0);               
    Lcd.Display("Inforamcie 0-9");
    Lcd.Cursor(0, 1);               
    Lcd.Display("teplota");
    Lcd.Cursor(0, 2);               
    Lcd.Display("vlhkost");
    Lcd.Cursor(0, 3);               
    Lcd.Display("stav BME680");
    Lcd.Cursor(0, 4);               
    Lcd.Display("stav MQ2");
    Lcd.Cursor(0, 5);               
    Lcd.Display("stav MQ7");
    Lcd.Cursor(0, 6);               
    Lcd.Display("stav MQ9");
    Lcd.Cursor(0, 7);               
    Lcd.Display("stav MQ135");
    
  }
  if(!digitalRead(11)){//vypis obrazovky "1"
    stav=1;
    Lcd.Clear(false);
    Lcd.Cursor(0, 0);                    
    Lcd.Display("Surove hodnoty");
    Lcd.Cursor(0, 1);               
    Lcd.Display("teplota");
    Lcd.Cursor(0, 2);               
    Lcd.Display("vlhkost");
    Lcd.Cursor(0, 3);               
    Lcd.Display("R BME680");
    Lcd.Cursor(0, 4);               
    Lcd.Display("U MQ2");
    Lcd.Cursor(0, 5);               
    Lcd.Display("U MQ7");
    Lcd.Cursor(0, 6);               
    Lcd.Display("U MQ9");
    Lcd.Cursor(0, 7);               
    Lcd.Display("U MQ135");
  }
  //precitanie hodnot senzorov typu MQ a zaroven triedenie
  MQ135=analogRead(5)*(3.3/4096.0);
  MQ9=analogRead(2)*(3.3/4096.0);
  MQ2=analogRead(3)*(3.3/4096.0);
  MQ7=analogRead(4)*(3.3/4096.0);
  s135=state135(MQ135);
  s9=state9(MQ9);
  s2=state2(MQ2);
  s7=state7(MQ7);
  sbme680=sbme680R(bme.gas_resistance / 1000.0);
//regulacia otacok
  if(sbme680+s135+s9+s2+s7>5) ledcWrite(26,255);
  else ledcWrite(26, 0);
  //vypis hodot na obrazovku "0" 
  if(stav==0){
    Lcd.Cursor(12, 1);
    sprintf(buf, "%.2f", (temp.temperature+bme.temperature)/2.0);               
    Lcd.Display(buf);
    Lcd.Cursor(12, 2);
    sprintf(buf, "%.2f", humidity.relative_humidity);               
    Lcd.Display(buf);
    Lcd.Cursor(12, 3);
    sprintf(buf, "%1d", sbme680);               
    Lcd.Display(buf);
    Lcd.Cursor(12, 4);
    sprintf(buf, "%1d", s2);               
    Lcd.Display(buf);
    Lcd.Cursor(12, 5);
    sprintf(buf, "%1d", s7);               
    Lcd.Display(buf);
    Lcd.Cursor(12, 6);
    sprintf(buf, "%1d", s9);               
    Lcd.Display(buf);
    Lcd.Cursor(12, 7);
    sprintf(buf, "%1d", s135);               
    Lcd.Display(buf);
  }
  else{//vypis hodnot na obrazovku "1"
    Lcd.Cursor(12, 1);
    sprintf(buf, "%.2f", (temp.temperature+bme.temperature)/2.0);               
    Lcd.Display(buf);
    Lcd.Cursor(12, 2);
    sprintf(buf, "%.2f", humidity.relative_humidity);               
    Lcd.Display(buf);
    Lcd.Cursor(12, 3);
    sprintf(buf, "%.2f", bme.gas_resistance / 1000.0);               
    Lcd.Display(buf);
    Lcd.Cursor(12, 4);
    sprintf(buf, "%.2f", MQ2);               
    Lcd.Display(buf);
    Lcd.Cursor(12, 5);
    sprintf(buf, "%.2f", MQ7);               
    Lcd.Display(buf);
    Lcd.Cursor(12, 6);
    sprintf(buf, "%.2f", MQ9);               
    Lcd.Display(buf);
    Lcd.Cursor(12, 7);
    sprintf(buf, "%.2f", MQ135);               
    Lcd.Display(buf);
  }
  //vypis seriovej linky
  Serial.print("teplota AHT10= "); Serial.print(temp.temperature); Serial.println(" *C"); Serial.print("teplota BME680= ");Serial.print(bme.temperature);Serial.println(F(" *C"));
  Serial.print("tlak BME680= ");Serial.print(bme.pressure / 100.0);Serial.println(" hPa");
  Serial.print("vlhkost AHT10= "); Serial.print(humidity.relative_humidity); Serial.println(" %"); Serial.print("vlhkost BME680= ");Serial.print(bme.humidity);Serial.println(F(" %"));
  Serial.print("plyn BME680= ");Serial.print(bme.gas_resistance / 1000.0);Serial.print(" KOhm; stav= "); Serial.println(sbme680);
  Serial.print("napetia MQ2;MQ7;MQ9;MQ135:  "); Serial.print(MQ2); Serial.print("V ;"); Serial.print(MQ7); Serial.print("V ;"); Serial.print(MQ9); Serial.print("V ;"); Serial.print(MQ135); Serial.print("V; stavy:"); Serial.print(s2); Serial.print(s7); Serial.print(s9); Serial.println(s135);
  Serial.println();
  delay(100);
}

int state135(float a){
  if(a<0.2) return 0;
  else if(a<0.3) return 1;
  else if(a<0.4) return 2;
  else if(a<0.6) return 2;
  else if(a<0.8) return 4;
  else if(a<1.0) return 5;
  else if(a<1.3) return 6;
  else if(a<1.7) return 7;
  else if(a<2.2) return 8;
  return 9;
}
int state9(float a){
  if(a<0.4) return 0;
  else if(a<0.5) return 1;
  else if(a<0.6) return 2;
  else if(a<0.7) return 2;
  else if(a<0.9) return 4;
  else if(a<1.1) return 5;
  else if(a<1.5) return 6;
  else if(a<1.9) return 7;
  else if(a<2.4) return 8;
  return 9;
}
int state7(float a){
  if(a<0.2) return 0;
  else if(a<0.3) return 1;
  else if(a<0.4) return 2;
  else if(a<0.6) return 2;
  else if(a<0.8) return 4;
  else if(a<1.0) return 5;
  else if(a<1.3) return 6;
  else if(a<1.7) return 7;
  else if(a<2.2) return 8;
  return 9;
}
int state2(float a){
  if(a<0.2) return 0;
  else if(a<0.3) return 1;
  else if(a<0.4) return 2;
  else if(a<0.6) return 2;
  else if(a<0.8) return 4;
  else if(a<1.0) return 5;
  else if(a<1.3) return 6;
  else if(a<1.7) return 7;
  else if(a<2.2) return 8;
  return 9;
}
int sbme680R(float a){
  if(a>250.0) return 0;
  else if(a>200.0) return 1;
  else if(a>200.0) return 2;
  else if(a>175.0) return 3;
  else if(a>150.0) return 5;
  else if(a>125.0) return 5;
  else if(a>100.0) return 6;
  else if(a>75.0) return 7;
  else if(a>50.0) return 8;
  else return 9;
}
