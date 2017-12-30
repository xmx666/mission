#include <IRremote.h>
#include <Adafruit_NeoPixel.h>
#include <SoftwareSerial.h>
SoftwareSerial mySerial(4, 5); 
#define my_Serial Serial/*引用”Adafruit_NeoPixel.h”文件。引用的意思有点象“复制-粘贴”。include文件提供了一种很方便的方式共享了很多程序共有的信息。*/
#define PIN 6
#define PIXEL_COUNT  6/*PIXEL_PIN:彩灯的控制引脚，用户可以自己更改控制引脚，同时也要更改硬件连接。/*定义了控制LED的引脚，6表示Microduino的D6引脚，可通过Hub转接出来，用户可以更改 */
Adafruit_NeoPixel strip = Adafruit_NeoPixel(1, PIN, NEO_GRB + NEO_KHZ800);
 //该函数第一个参数控制串联灯的个数，第二个是控制用哪个pin脚输出，第三个显示颜色和变化闪烁频率

#define Light_PIN A0  //光照传感器接AO引脚
#define Light_value 400
#define val_max 255
#define val_min 0   

int RECV_PIN = 10;   //红外线接收器OUTPUT端接在pin 10
IRrecv irrecv(RECV_PIN);  //定义IRrecv对象来接收红外线信号
decode_results results;   //解码结果放在decode_results构造的对象results里

const int buttonPin = 2;     
// 读取按键状态的变量
int buttonState = 0;//初始化状态值  
int times=0;
int red=0;
int green=0;
int blue=0;
int ThreeLoop=0;
int up=0;
int down=0;

String currentInfo;//定义一个string对象

char buffer[100];//定义一个字符串数组

boolean buffer_sta = false;//定义一个boolean常量

int buffer_num = 0;//定义一个整型常量

int sta[4];//定义一个有4个整型常量的数组

boolean color_en;
long safe_ms = millis();


int sensorValue;
void colorWipe(uint32_t c);
void rainbowCycle( int r, int g, int b, uint8_t wait);
void colorSet(uint32_t c);
 void ble();
 void rainbow(uint8_t wait);

void setup()                                //创建无返回值函数
 {
  Serial.begin(115200);
  my_Serial.begin(9600);
  strip.begin();                             //准备对灯珠进行数据发送
  strip.show();                         //初始化所有的灯珠为关的状态
  pinMode(buttonPin, INPUT);//配置引脚的模式为输入模式
  Serial.println("Initialisation complete.");//调试信息
  irrecv.enableIRIn(); // 启动红外解码
}
void loop()                                  //无返回值loop函数
{
  ble();//“ble()”函数是用来接收并解析手机发送的数据，解析数据函数。收到数据便进入蓝牙模式
 while(!color_en)//若未接收到数据，进入光感模式
  { 
  buttonState = digitalRead(buttonPin);//读取按键的状态
  Serial.println(buttonState);  //将状态输出到串口   
  sensorValue = analogRead(Light_PIN);             //光检测
  Serial.println(sensorValue); //彩色led灯根据光强调节颜色和亮度
  Serial.println(results.value,HEX);//输出红外线解码结果（十六进制）
  
  if (sensorValue <= Light_value)//光强小于400
  {
     if (irrecv.decode(&results))//解码成功，收到一组红外线信号。用整形变量记录，避免接受irrecv.resume()后，红外线解码结果消失导致出错。
     {
        
        if(results.value==33456255)//遥控器A键，变红光
       
        {        
           red++;
           delay(100);
           irrecv.resume();//接收下一个值
        }
        if(results.value==33439935)//遥控器B键，变绿光
        {
          green++;
          delay(100);
          irrecv.resume();
        }
        if(results.value==33472575)//遥控机C键，变蓝光
        {
          blue++;
          delay(100);
          irrecv.resume();
          
        }
        if(results.value==33444015)//遥控器循环键，变渐变光
        {
          ThreeLoop++;
          delay(100);
          irrecv.resume();
        }
        if(results.value==33464415)//遥控器up键，增大光强
        {
          up++;
          delay(100);
          irrecv.resume();
        }
        if(results.value==33478695)//遥控器down键
          down++;
          delay(100);
          irrecv.resume();
        }
         if(results.value==33427695)//遥控器OK键，回到初始状态
        {
          
           red=0;
           green=0;
           blue=0;
          ThreeLoop=0;
           up=0;
           down=0;
           delay(100);
          irrecv.resume();
        }
        if(results.value==33441975)//遥控器OFF/ON键，与下面触摸开关一起控制关灯或开灯，times为偶数亮，为奇数灭。
        {
           times++; Serial.println(times);delay(1000); irrecv.resume();     
        }
        else
        {
          delay(100);
          irrecv.resume();
          
        }  
     }
    if(buttonState==0)//触摸开关
    {
      times++; Serial.println(times);delay(100);       
    }
    if(times%2==0)
    {
      
        if(red%2!=0)//红光，越暗灯越亮
        {
          colorWipe(strip.Color(map(sensorValue, 0, 400, 255, 0), 0, 0));
          
        }
        
        else if(green%2!=0)//绿光，越暗灯越亮  
           colorWipe(strip.Color(0, map(sensorValue, 0, 400, 255, 0), 0));
           
        
        else if(blue%2!=0)//蓝光，越暗灯越亮     
           colorWipe(strip.Color(0, 0, map(sensorValue, 0, 400, 255,0)));
        
        else if(ThreeLoop%2!=0)//渐变光
        {
          for (int i = 0; i < 3; i++)
          rainbow(30);
        }
        else if(up!=0||down!=0)//控制光强
        {
          if((up-down)>0&&(up-down)<=5)
          {
            switch(up-down)
            {
             case 1:
                colorWipe(strip.Color(50, 50, 50));
                break;
             case 2:
                colorWipe(strip.Color(100, 100, 100));
                break;
              case 3:
                colorWipe(strip.Color(150, 150, 150));
                break;
              case 4:
                colorWipe(strip.Color(200, 200, 200));
                break;
              case 5:
                colorWipe(strip.Color(250, 250, 250));
                break;
                
                
            }
          }
          else if((up-down)<=0)
            colorWipe(strip.Color(0, 0, 0));
          else                                     
             colorWipe(strip.Color(255, 255, 255));
          
        }

        
         else        //  初始状态：白光，越暗灯越亮          
           colorWipe(strip.Color(map(sensorValue, 0, 400, 255, 0), map(sensorValue, 0, 400, 255, 0), map(sensorValue, 0, 400, 255, 0)));
        
      
      }      
     

    
     if(times%2!=0)    //灭灯  
     colorWipe(strip.Color(0,0,0));
  
  
  if (sensorValue > Light_value)// 光强超过400后，灭灯，回到原始状态。
  {
    times=0;
     red=0;
     green=0;
     blue=0;
     ThreeLoop=0;
     up=0;
    down=0;
    colorWipe(strip.Color(0,0,0));
  } 
     ble(); 
 }
 
}



void ble()
{
  while (my_Serial.available())//while循环语句
  {
    char c = my_Serial.read();//定义一个字符常量
    delay(2);

    if (c == 'C')
      buffer_sta = true;// 当蓝牙连接时，会持续给BT发送“\n”的消息，只要有接收到则认为蓝牙一直在连接。并且给“color_en”赋值为真（true）
    if (c == '\n')
    {
      color_en = true;
      safe_ms = millis();
    }
    if (buffer_sta)
    {
      buffer[buffer_num] = c;
      buffer_num++;
    }
    //  Serial.println(c);//调用对象Serial的函数println（）
    //Serial.println(color_en);
  }

  if (buffer_sta)
  {
    buffer_sta = false;

    sscanf((char *)strstr((char *)buffer, "C:"), "C:%d,%d,%d,%d", &sta[0], &sta[1], &sta[2], &sta[3]);

    for (int a = 0; a < buffer_num; a++)
      buffer[a] = NULL;
    buffer_num = 0;

    for (int i = 0; i < 4; i++)
    {
      Serial.print(sta[i]);
      Serial.print(",");
    }
    Serial.println(" ");

    if (-1 == sta[3]) {
      colorSet(strip.Color(sta[0], sta[1], sta[2]));
    }
   else if ((0 <= sta[3]) && (sta[3] < PIXEL_COUNT)) 
   {
      colorSet(strip.Color(sta[0], sta[1], sta[2]), sta[3]);
  }//将解析到的数据来控制灯的颜色变化，如果sta[3]的值为-1，所有连接的灯都是同一个颜色，为接收到的“sta[0], sta[1], sta[2]”组合而来的颜色，即手机上选的颜色。否则将可以单独控制6个灯的颜色。
  }

 if (millis() - safe_ms > 3000)//用现在millis（）函数返回的值减去之前定义的safe_ms的数值并于3000比较大小，蓝牙断开时，系统会在3S后让“color_en”的值为假（false）。开始执行自定义颜色变化，“!”表示非的意思。

  {
    safe_ms = millis();
    color_en = false;
  }
}

 

   

void rainbowCycle( int r, int g, int b, uint8_t wait) {
  for (int val = 0; val < 255; val++) 
  //val由0自增到254不断循环
  {
colorSet(strip.Color(map(val, val_min, val_max, 0, r), map(val, val_min, val_max, 0, g), map(val, val_min, val_max, 0, b)));
//红绿蓝LED灯依次从暗到亮
/*“map(val,x,y,m,n)”函数为映射函数，可将某个区间的值（x-y）变幻成（m-n），val则是你需要用来映射的数据*/
    delay(wait); //延时
  }
  for (int val = 255; val >= 0; val--)  //val从255自减到0不断循环
  {
colorSet(strip.Color(map(val, val_min, val_max, 0, r), map(val, val_min, val_max, 0, g), map(val, val_min, val_max, 0, b)));
//红绿蓝LED灯依次由亮到暗
    delay(wait); //延时
  
  }
}

void colorWipe(uint32_t c) {
  for (uint16_t i = 0; i < strip.numPixels(); i++)  //i从0自增到LED灯个数减1
 {
    strip.setPixelColor(i, c); //将第i个灯点亮
    strip.show(); //led灯显示
  }
 }
 void colorSet(uint32_t c) {
  for (uint16_t i = 0; i < strip.numPixels(); i++) {
    strip.setPixelColor(i, c);
  }
  strip.show();
}

void colorSet(uint32_t c, int i) {
  strip.setPixelColor(i, c);
  strip.show();
}
 uint32_t Wheel(byte WheelPos) {
  if (WheelPos < 85) {
    return strip.Color(WheelPos * 3, 255 - WheelPos * 3, 0);
  } else if (WheelPos < 170) {
    WheelPos -= 85;
    return strip.Color(255 - WheelPos * 3, 0, WheelPos * 3);
  } else {
    WheelPos -= 170;
    return strip.Color(0, WheelPos * 3, 255 - WheelPos * 3);
  }
}
 void rainbow(uint8_t wait) {
  uint16_t i, j;

  for (j = 0; j < 256; j++) {
    for (i = 0; i < strip.numPixels(); i++) {
      strip.setPixelColor(i, Wheel((i + j) & 255));
    }
    strip.show();
    delay(wait);
  }
 }
 
                                                                                 
