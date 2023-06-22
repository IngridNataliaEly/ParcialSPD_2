# Parcial 2 Sistema de incendio con Arduino
El objetivo de este proyecto es diseñar un sistema de incendio utilizando Arduino que pueda
detectar cambios de temperatura y activar un servo motor en caso de detectar un incendio.
Además, se mostrará la temperatura actual y la estación del año en un display LCD.
## Alumna 
Ingrid Natalia Ely

# PARCIAL :  Sistema de incendio 
![Sin título](![Sin título](https://github.com/IngridNataliaEly/ParcialSPD_2/assets/108601149/2d262c73-2e78-4fb6-a509-7a82e77b52e1)


## Descripción
Este proyecto utiliza Arduino para monitorear la temperatura y mostrar alertas en un LCD. Se utiliza un sensor de temperatura y un control remoto infrarrojo. Cuando la temperatura excede un umbral, se activa una alarma y se muestra en el LCD. El control remoto permite activar y desactivar la alarma. También se muestra la temperatura actual y la estación del año correspondiente en el LCD.

## Circuito Esquematico
![circuito esquematico](https://github.com/IngridNataliaEly/ParcialSPD_2/assets/108601149/a5f732f2-cf19-4a3e-92b6-6792f4964b58)


## Explicación de los componentes:

Utiliza Arduino, un sensor de temperatura, una pantalla LCD, un LED, un control remoto y un servo motor. Arduino es la plataforma de desarrollo utilizada para controlar todos los componentes. El sensor de temperatura mide la temperatura ambiente. La pantalla LCD muestra información como la temperatura y la estación del año. El LED indica el estado de la alarma. El control remoto permite activar o desactivar la alarma y controlar el servo motor. El servo motor se utiliza para realizar acciones según la activación de la alarma.

~~~
#include <IRremote.h>
#include <LiquidCrystal_I2C.h>
#include <Wire.h>
#include <Servo.h>

#define Tecla_1 0xEF10BF00
#define Tecla_2 0xEE11BF00
#define LED_ROJO 4
#define LED_VERDE 3

int IR = 11;
int TMP = 0; // El sensor está conectado al pin analógico A0
float temperatura = 0; // Leer datos del sensor
const float tempPrimaveraMin = 15.0;
const float tempPrimaveraMax = 25.0;
const float tempVeranoMin = 25.0;
const float tempVeranoMax = 35.0;
const float tempOtonoMin = 10.0;
const float tempOtonoMax = 20.0;
const float tempInviernoMin = -41.0;
const float tempInviernoMax = 10.0;
bool encendido = false;
bool volverLow = false;
bool servoActivado = false;

LiquidCrystal_I2C lcd(34, 16, 2); // Cambia la dirección I2C 0x27 según la dirección real de tu LCD
Servo motor_9;

void setup()
{
  pinMode(LED_ROJO, OUTPUT);
  pinMode(LED_VERDE, OUTPUT);
  motor_9.attach(9, 500, 25000);
  lcd.init(); // Inicializar la pantalla LCD
  lcd.backlight(); // Encender la retroiluminación de la pantalla LCD
  Serial.begin(9600);
  IrReceiver.begin(IR, DISABLE_LED_FEEDBACK);
}

unsigned long lastDeactivationTime = 0;
const unsigned long debounceDelay = 1000; // Tiempo de espera en milisegundos después de desactivar la alarma

void loop()
{
  control();

  if (leerTemperatura() >= 40 || encendido)
  {
    if (!volverLow && (millis() - lastDeactivationTime > debounceDelay)) // Agrega la condición de temporizador
    {
      alertaTemperatura(HIGH);
    }
    else
    {
      volverLow = false; // Reinicia el valor de volverLow a falso
      alertaTemperatura(LOW);
    }
  }
  else if (leerTemperatura() < 40 || !encendido)
  {
    alertaTemperatura(LOW);
    activarDesactivarServo(LOW);
  }

  delay(10);
}

float leerTemperatura()
{
  // Leer los valores del sensor TMP36
  temperatura = map(analogRead(TMP), 0, 1023, -50, 450);

  return temperatura;
}

void alertaTemperatura(int encendido)
{
  if (encendido == HIGH)
  {
    ledEncendidoApagado(1, 0); // Encender  el LED rojo. Apagar el LED verde
    lcd.clear();
    lcd.setCursor(0, 0); // Posiciona el cursor de escritura
    lcd.print("ALERTA!MAX:");
    lcd.print(temperatura);
    activarDesactivarServo(HIGH);
  }
  else
  {
    ledEncendidoApagado(0, 1); // Apagar el LED rojo. Encender el LED verde
     lcd.clear();
    lcd.setCursor(0, 0); // Posiciona el cursor de escritura
    lcd.print("Temperatura:");
    lcd.print(temperatura);
	
    activarDesactivarServo(LOW);
  }

  switch (mostrarEstacion())
  {
    case 1:
     
      lcd.setCursor(0, 1); // Posiciona el cursor de escritura
      lcd.print("Primavera");
      break;
    case 2:
      lcd.setCursor(0, 1); // Posiciona el cursor de escritura
      lcd.print("Verano");
      break;
    case 3:
      lcd.setCursor(0, 1); // Posiciona el cursor de escritura
      lcd.print("Otonio");
      break;
    case 4:
      lcd.setCursor(0, 1); // Posiciona el cursor de escritura
      lcd.print("Invierno");
      break;
    default:
      lcd.setCursor(0, 1); // Posiciona el cursor de escritura
      lcd.print("Esperando");
      break;
  }

  Serial.println(mostrarEstacion());
}

void control()
{
  if (IrReceiver.decode())
  {
    Serial.println(IrReceiver.decodedIRData.decodedRawData, HEX);

    if (IrReceiver.decodedIRData.decodedRawData == Tecla_1)
    {
      alertaTemperatura(HIGH);
      encendido = true;
    }

    if (IrReceiver.decodedIRData.decodedRawData == Tecla_2 && (millis() - lastDeactivationTime > debounceDelay))
    {
      volverLow = true;
      encendido = false;
      lastDeactivationTime = millis(); // Actualiza el tiempo de desactivación
    }

    IrReceiver.resume();
  }

  delay(10);
}

void ledEncendidoApagado(int ledRojo, int ledVerde)
{
  digitalWrite(LED_ROJO, ledRojo);
  digitalWrite(LED_VERDE, ledVerde);
}

void activarDesactivarServo(int estado)
{
  if (estado == HIGH && !servoActivado)
  {
    for (int i = 0; i <= 180; i++)
    {
      motor_9.write(i);
      delay(10);
    }
    servoActivado = true;
  }
  else if (estado == LOW && servoActivado)
  {
    motor_9.write(0);
    servoActivado = false;
  }
}

int mostrarEstacion()
{
  // Determinar la estación del año
  int estacion;
  if (temperatura >= tempPrimaveraMin && temperatura <= tempPrimaveraMax)
  {
    estacion = 1;
  }
  else if (temperatura >= tempVeranoMin && temperatura <= tempVeranoMax)
  {
    estacion = 2;
  }
  else if (temperatura >= tempOtonoMin && temperatura <= tempOtonoMax)
  {
    estacion = 3;
  }
  else if (temperatura >= tempInviernoMin && temperatura <= tempInviernoMax)
  {
    estacion = 4;
  }
  else
  {
    estacion = 5;
  }

  return estacion;
}
~~~
## Funcionamiento Integral
Arduino recibe la lectura de temperatura del sensor y la convierte en un valor numérico.

Si la temperatura supera un umbral predefinido o si la alarma está activada mediante el control remoto, se activa la alerta.

La alerta se indica encendiendo un LED rojo y mostrando un mensaje en la pantalla LCD que indica la temperatura máxima alcanzada.

Además, se activa un servo motor que puede realizar una acción específica, como abrir una puerta o mover un objeto, dependiendo de las necesidades del proyecto.

Si la temperatura vuelve a estar por debajo del umbral o si la alarma se desactiva mediante el control remoto, la alerta se apaga.

La pantalla LCD también muestra la estación del año correspondiente según la temperatura actual. Esto se determina comparando la temperatura con rangos predefinidos para cada estación.

El control remoto permite al usuario activar o desactivar la alarma y controlar el servo motor según sea necesario.

![alarma](https://github.com/IngridNataliaEly/ParcialSPD_2/assets/108601149/58269c0d-d670-40a1-b543-6584e2c7575e)

![alarma encendida](https://github.com/IngridNataliaEly/ParcialSPD_2/assets/108601149/a984ca6a-5317-4521-9b2f-ebbd8f7ef28f)

## Enlace 
[Parcial 2 SPD]([https://www.tinkercad.com/things/lnlCAm9TDYs](https://www.tinkercad.com/things/fx7wsHPQvJh?sharecode=rd6b8XcAruNTtT6XU7Ou2Cjx1vrN_RfxpAWrgZqncR8)https://www.tinkercad.com/things/fx7wsHPQvJh?sharecode=rd6b8XcAruNTtT6XU7Ou2Cjx1vrN_RfxpAWrgZqncR8)
