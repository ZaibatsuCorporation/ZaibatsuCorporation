//==================================================
/*//  Прошивка бортового компьютера Бориса.
  //
  //  Код программы занимает 30% флеш-памяти и 19% динамической памяти.
  //  Программа разработана на базе Arduino Nano 328.
  //
  //  Код программы принадлежит ZC.
*/
//==================================================
String firmwareversion = "v200420242243";
//==================================================
// Редактор символов (порядок для TM1637 a,g,f,e,d,c,b,h):
// https://naladchikkip.ru/spravochnik/onlajn-generator-koda-semisegmentnyh-indikatorov

/* БИБЛИОТЕКИ */
#include <GyverButton.h>
#include <EEPROM.h> // Память
#include <RBD_Timer.h> // Таймер
#include <GyverTM1637.h>
/* ПИНЫ */
#define SPEED_INPUT 2 // пин датчика холла с колеса (D2 или D3)
#define LED_OUT 13 // пин встроенного светодиода (D13)
#define CLK 5 // пин дисплея
#define DIO 4 // пин дисплея
GButton BTN1(6, HIGH_PULL, NORM_OPEN);
GButton BTN2(7, HIGH_PULL, NORM_OPEN);
GButton BTN3(8, HIGH_PULL, NORM_OPEN);
GButton BTN4(9, HIGH_PULL, NORM_OPEN);
/* Пользовательские настройки */
byte numberOfMagnets = 1; // Количество магнитов/считываний за один оборот
unsigned int dispSpeedWelcome = 60; // скорость анимации при запуске (мс)
unsigned int dispSpeedWheel = 60; // скорость анимации колёс по краям (мс)
byte brightnessDay = 7; // яркость дисплея днём (от 0 до 7)
byte brightnessNight = 1; // яркость дисплея ночью (от 0 до 7)
bool autoBrightness = 0; // динамическая яркость (0/1)
float w_length = 2.120; // длина окружности колеса (мм)
unsigned int timeToStop = 1500; // время с последнего измерения до смены состояния на "остановился" (мс)
unsigned int maxStepSpeed = 50; // допустимое правдоподобное изменение скорости (км/ч за один расчет), от 1 до 99
unsigned int minWaitToNextCalcSpeed = 50; // минимально допустимая пауза между измерениями (мс)
/* СИМВОЛЫ */
#define _up 0x01
#define _upRight 0x02
#define _downRight 0x04
#define _down 0x08
#define _downLeft 0x10
#define _upLeft 0x20
#define _upLeftDownLeft 0x30
#define _upDownMiddle 0x49
#define _upRightDownRight 0x06

RBD::Timer timerSpeedWheel;
bool flagShowWheel = 1;
byte countShowWheel = 1;
byte cycleShowWheel = 0;
int menuState = 0;
bool firstDemoWheel = 1;
bool bikeMoveState = 0;
float DIST = 0;
float DISTtemp = 0;
float floatSpeed = 0;
int intSpeed = 0;
int preSpeed = 0;
float bufferSpeed = 0;
long lastturn = 0;
int firstNumber = 0;
int secondNumber = 0;
bool flagShowSpeed = 0;
uint32_t Now, clocktimer;
GyverTM1637 disp(CLK, DIO);

void setup() {
  pinMode(SPEED_INPUT, INPUT);
  pinMode(LED_OUT, OUTPUT);
  digitalWrite(LED_OUT, LOW);
  if (SPEED_INPUT == 2) {
    attachInterrupt(0, calcSpeed, FALLING);
  }
  if (SPEED_INPUT == 3) {
    attachInterrupt(1, calcSpeed, FALLING);
  }
  BTN1.setTickMode(AUTO);
  BTN1.setDebounce(30);
  BTN1.setTimeout(400);
  BTN2.setTickMode(AUTO);
  BTN2.setDebounce(30);
  BTN2.setTimeout(400);
  BTN3.setTickMode(AUTO);
  BTN3.setDebounce(30);
  BTN3.setTimeout(400);
  BTN4.setTickMode(AUTO);
  BTN4.setDebounce(30);
  BTN4.setTimeout(400);
  EEPROM.get(0, DIST);
  EEPROM.get(4, DISTtemp);
  timerSpeedWheel.setTimeout(dispSpeedWheel);
  //DIST = 18;
  //DISTtemp = 2;
  Serial.begin(9600);
  Serial.setTimeout(50);
  Serial.println(F("///////////////////////////////////////////////////////////"));
  Serial.println(F("Прошивка бортового компьютера Бориса "));
  Serial.print(firmwareversion);
  Serial.println(F(", автор Печерников О.А."));
  Serial.println(F("///////////////////////////////////////////////////////////"));
  disp.clear();
  disp.brightness(brightnessDay);  // яркость, 0 - 7 (минимум - максимум)
  disp.clear();
  firstShow();
  if (DIST > 99999 || DIST < 0) {
    DIST = 0;
    saveData();
    Serial.println(F("Исправлена переменная DIST."));
    disp.displayByte(_E, _r, _r, _empty);
    delay(1500);
    disp.displayByte(_empty, _d, _1, _empty);
    delay(3000);
    disp.clear();
  }
  if (DISTtemp > 99999 || DISTtemp < 0) {
    DISTtemp = 0;
    saveData();
    Serial.println(F("Исправлена переменная DISTtemp."));
    disp.displayByte(_E, _r, _r, _empty);
    delay(1500);
    disp.displayByte(_empty, _d, _2, _empty);
    delay(3000);
    disp.clear();
  }
  Serial.print(F("Общий пробег: "));
  Serial.print(DIST);
  Serial.println(F("км"));
  Serial.print(F("Разовый пробег: "));
  Serial.print(DISTtemp);
  Serial.println(F("км"));
  digitalWrite(LED_OUT, HIGH);
  delay(500);
  digitalWrite(LED_OUT, LOW);
  Serial.println(F("ГОТОВ К РАБОТЕ."));
  disp.display(1, 0);
  disp.display(2, 0);
}
void loop() {
  disp.brightness(brightnessDay);
  /*
      if (BTN1.isClick()) {

      }
      if (BTN2.isClick()) {
        flag = 1;
      }
      if (menuState == 4 && flag) {
        flag = 0;
        menuState = 1;
        disp.displayByte(_o, _d, _o, _empty);
      }
      if (menuState == 1 && flag) {
        flag = 0;
        menuState = 2;
        disp.displayByte(_o, _d, _o, _t);
      }
      if (menuState == 2 && flag) {
        flag = 0;
        menuState = 3;
        disp.displayByte(_, _d, _o, _t);
      }
    }
    if (BTN2.isClick()) {
    disp.displayByte(_o, _d, _o, _empty);
    delay(2000);
    int i = DIST;
    disp.displayInt(i);
    delay(3000);
    disp.displayByte(_empty, _0, _0, _empty);
    }
  */
  if (bikeMoveState == 0 && intSpeed != 0) {
    bikeMoveState = 1;
    Serial.println("Поехал.");
    flagShowWheel = 1;
  }
  if (flagShowSpeed == 1) {
    flagShowSpeed = 0;
    disp.display(1, firstNumber);
    disp.display(2, secondNumber);
    digitalWrite(LED_OUT, LOW);
    //Serial.print(firstNumber);
    //Serial.println(secondNumber);
  }
  if ((millis() - lastturn) > timeToStop && bikeMoveState == 1) {
    intSpeed = 0;
    floatSpeed = 0;
    bikeMoveState = 0;
    firstNumber = 0;
    secondNumber = 0;
    flagShowSpeed = 1;
    Serial.println("Остановился.");
    saveData();
  }
  wheelShow();
}

void calcSpeed () {
  if (millis() - lastturn > minWaitToNextCalcSpeed) {
    preSpeed = w_length / ((float)(millis() - lastturn) / 1000) * 3.6 / numberOfMagnets;
    lastturn = millis();
    if (preSpeed > 100) {
      preSpeed = 99;
    }
    if ((preSpeed + maxStepSpeed) > bufferSpeed && preSpeed < (bufferSpeed + maxStepSpeed)) { // Если скорость резко не уменьшилась
      digitalWrite(LED_OUT, HIGH);
      floatSpeed = preSpeed;
      intSpeed = floatSpeed;
      DIST += w_length / 1000;
      DISTtemp += w_length / 1000;
      int tempSpeed = intSpeed;
      firstNumber = 0;
      secondNumber = 0;
      if (intSpeed >= 10 && intSpeed < 100) {
        for (tempSpeed = intSpeed; tempSpeed >= 10; tempSpeed -= 10) {
          firstNumber++;
          //Serial.println(tempSpeed);
        }
      }
      for (tempSpeed = tempSpeed; tempSpeed >= 1; tempSpeed--) {
        secondNumber++;
        //Serial.println(tempSpeed);
      }
      flagShowSpeed = 1;
    }
  }
}

void saveData() {
  EEPROM.put(0, DIST);
  EEPROM.put(4, DISTtemp);
}

void firstShow() {
  disp.brightness(7);
  disp.point(POINT_ON);
  delay(dispSpeedWelcome);
  disp.brightness(6);
  disp.point(POINT_OFF);
  disp.displayByte(1, _upRightDownRight);
  disp.displayByte(2, _upLeftDownLeft);
  delay(dispSpeedWelcome);
  disp.brightness(5);
  disp.displayByte(1, _upDownMiddle);
  disp.displayByte(2, _upDownMiddle);
  delay(dispSpeedWelcome);
  disp.brightness(4);
  disp.displayByte(1, _upLeftDownLeft);
  disp.displayByte(2, _upRightDownRight);
  delay(dispSpeedWelcome);
  disp.brightness(3);
  disp.displayByte(1, _empty);
  disp.displayByte(2, _empty);
  disp.displayByte(0, _upRightDownRight);
  disp.displayByte(3, _upLeftDownLeft);
  delay(dispSpeedWelcome);
  disp.brightness(2);
  disp.displayByte(0, _upDownMiddle);
  disp.displayByte(3, _upDownMiddle);
  delay(dispSpeedWelcome);
  disp.brightness(1);
  disp.displayByte(0, _upLeftDownLeft);
  disp.displayByte(3, _upRightDownRight);
  delay(dispSpeedWelcome);
  disp.brightness(0);
  disp.displayByte(0, _empty);
  disp.displayByte(3, _empty);
}

void wheelShow() {
  if (flagShowWheel == 1) {
    if (firstDemoWheel == 1) {
      firstDemoWheel = 0;
    }
    else {
      disp.display(1, firstNumber);
      disp.display(2, secondNumber);
    }
    disp.brightness(brightnessDay);
    if (flagShowWheel == 1 && timerSpeedWheel.onExpired()) {
      bool flag = 1;
      if (countShowWheel == 1 && flag) {
        flag = 0;
        countShowWheel = 2;
        timerSpeedWheel.restart();
        disp.displayByte(0, _up); disp.displayByte(3, _up);
      }
      if (countShowWheel == 2 && flag) {
        flag = 0;
        countShowWheel = 3;
        timerSpeedWheel.restart();
        disp.displayByte(0, _upRight); disp.displayByte(3, _upLeft);
      }
      if (countShowWheel == 3 && flag) {
        flag = 0;
        countShowWheel = 4;
        timerSpeedWheel.restart();
        disp.displayByte(0, _downRight); disp.displayByte(3, _downLeft);
      }
      if (countShowWheel == 4 && flag) {
        flag = 0;
        countShowWheel = 5;
        timerSpeedWheel.restart();
        disp.displayByte(0, _down); disp.displayByte(3, _down);
      }
      if (countShowWheel == 5 && flag) {
        flag = 0;
        countShowWheel = 6;
        timerSpeedWheel.restart();
        disp.displayByte(0, _downLeft); disp.displayByte(3, _downRight);
      }
      if (countShowWheel == 6 && flag) {
        flag = 0;
        countShowWheel = 1;
        timerSpeedWheel.restart();
        disp.displayByte(0, _upLeft); disp.displayByte(3, _upRight);
        cycleShowWheel++;
        if (cycleShowWheel == 3) {
          cycleShowWheel = 0;
          countShowWheel = 7;
        }
      }
      if (countShowWheel == 7 && flag) {
        flag = 0;
        flagShowWheel = 0;
        countShowWheel = 1;
        timerSpeedWheel.restart();
        disp.displayByte(0, _empty);
        disp.displayByte(3, _empty);
      }
    }
  }
}
