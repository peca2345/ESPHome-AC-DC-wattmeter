# ESPHome AC/DC wattmeter (FVE)

## Popis:

- následující postup je uplatněn pro měření DC (FVE) a AC (ze sítě) výkonu dodaného pro Bojler s AC/DC spirálou
- mezi FVE a bojlerem je regulátor bez měniče. Do bojelru příchází stejnosměrné napětí 
- měření DC je realizováno pomocí PZEM-017 a AC PZEM-004Tv3

## Zařízení:
- Bojler - Dražice LX ACDC/M+K 160l [manual](https://www.dzd-fv.cz/images/pdf/Navod_LX_ACDC_M_MKW_9_12_2020_CZ_6735552.pdf) [eshop](https://www.solar-eshop.cz/p/fotovoltaicky-ohrivac-lx-acdc-m-k-abc-160/)  
- FV panely: 6ks Schutten Poly 250Wp  
- MPPT regulátor Power Max   
- ESP32 1ch relay board   
- DC wattmetr PZEM-017v1 DC 0-300V / 0-300A  
- AC wattmetr PZEM-004Tv3 AC 80-260V / 0-100A 

## Zapojení:

![schema](https://user-images.githubusercontent.com/58307338/161399144-c7f32090-afcf-4360-a1a0-9e27996f3412.png)

## ESP32E 1ch relay board:

- z neznámého důvodu je relé připojeno na UART2_RX - GPIO16.  
- díky tomu při používání UARTu pro PZEM dochází ke spínání relé.  
- řešením je přerušení cesty mezi GPIO16 a relé. V případě potřeby použití relé můžete cestu napojit na jiný GPIO.  

![img](https://user-images.githubusercontent.com/58307338/161399294-d6a7c2da-280d-4cb1-b927-bd970b2a0270.png)

## PZEM-017 DC wattmeter:  

- PZEM-017 používá pro komunikaci rozhraní RS485 a standardně bychom museli signál převést z RS485 přes převodník zpět na UART.  
- to je pro nás tedy zbytečné a můžeme odstranit RS485 přímo z PZEM a použít přímo jeho UART.  
- odstraníme tedy RS485 rozhraní přímo ze základní desky a díky tomu můžeme a musíme pracovat s napětím 3V namísto původních 5V.  

**1. odstraňte IO U5 a rezistor R19 (na U5 stačí odpájet pin1 a R19 na mém PZEM017 nebyl vůbec osazen)**

![removeU5](https://user-images.githubusercontent.com/58307338/161399111-db25c8f7-ec0b-4754-973d-85968283fbed.png)

**2. propojte optočlen U1 (pin4) s R19 (pin A) a  U2 (pin2) s R19 (pin B)**

<img align="right" src="https://user-images.githubusercontent.com/58307338/161399115-7f604a6b-4fb5-4c64-a20c-d4d6b986b9b6.png">

- odteď bude tedy na svorce A signál TX a na svorce B RX
- na ESP připojte GPIO1 (TX) na svorku B (RX) a GPIO3 (RX) na svorku A (TX)  
- myslete na to že se již PZEM017 nenapájí 5V ale jen 3V z ESP - jinak nebude wattmetr funkční!!  

- po správném zapojení nahrajte postupně program pro PZEM017 a zvlášť pro PZEM004 abych si ověřili funkčnost zapojení  
- pracuji s ESP32 který má jeden volný UART (gpio16,17) navíc a proto mohu mít zapnutý i logger 
- pokud používáte ESP8266 tak jediný UART používá logger a pokud ho chcete uvolnit pro PZEM tak ho musíte vypnout!
- v esphome dále pracuji s použitím druhého UARTU pro PZEM a k tomu zapnutý logger

Vypnutí loggeru:  

```
logger:
   level: NONE
   baud_rate: 0
```


**ESPHome yaml PZEM017:**  

- aby PZEM v HA zobrazoval hodnoty tak musí být přivedeno DC napětí na svorky pro měření!  
- konkrétně jde o dvě krajní svorkovnice na straně microUSB konektoru. (první plus, druhá mínus)  

```
logger:
   level: debug
   baud_rate: 0
  
uart:
  rx_pin: GPIO16 # GPIO3
  tx_pin: GPIO17 # GPIO1
  baud_rate: 9600
  stop_bits: 2

sensor:
  - platform: pzemdc
    current:
      name: "PZEM-017 Current"
    voltage:
      name: "PZEM-017 Voltage"
    power:
      name: "PZEM-017 Power"
    update_interval: 1s
```

**ESPHome yaml PZEM004Tv3:** 

- pokud vše funguje tak přejdeme k testu PZEM004T  
- vymažeme konfiguraci pro PZEM017 a nahrajeme novou jen pro PZEM004T   


```
logger:
   level: debug
   baud_rate: 0
  
uart:
  rx_pin: GPIO16 # GPIO3
  tx_pin: GPIO17 # GPIO1
  baud_rate: 9600
  stop_bits: 2

sensor:
  - platform: pzemdc
    current:
      name: "PZEM-003 Current"
    voltage:
      name: "PZEM-003 Voltage"
    power:
      name: "PZEM-003 Power"
    update_interval: 1s
    
  - platform: pzemac
    current:
      name: "PZEM-004T V3 Current"
    voltage:
      name: "PZEM-004T V3 Voltage"
    energy:
      name: "PZEM-004T V3 Energy"
    power:
      name: "PZEM-004T V3 Power"
    update_interval: 1s
```

Pokud tedy vše funguje potřebujeme u PZEM004T změnit adresu jelikož mají oba PZEM defaultní stejnou 0x01 adresu.  

**PZEM004Tv3 - změna adresy**

- vymažemé původní kód a nahradíme sekvencí pro automatické přepsání adresy po bootu
- jakmile nahrajeme do ESP tak ho restartujeme a sekvenci vymažeme a vložíme konfig pro PZEM017 i PZEM004T

```
esphome:
  name: esp32-fve-AC-DC
  on_boot:
    ## configure controller settings at setup
    ## make sure priority is lower than setup_priority of modbus_controller
    priority: -100
    then:
      - lambda: |-
          auto new_address = 0x03;

          if(new_address < 0x01 || new_address > 0xF7) // sanity check
          {
            ESP_LOGE("ModbusLambda", "Address needs to be between 0x01 and 0xF7");
            return;
          }

          esphome::modbus_controller::ModbusController *controller = id(pzem);
          auto set_addr_cmd = esphome::modbus_controller::ModbusCommandItem::create_write_single_command(
            controller, 0x0002, new_address);

          delay(200) ;
          controller->queue_command(set_addr_cmd);
          ESP_LOGI("ModbusLambda", "PZEM Addr set"); 
          
          
logger:
   level: debug
   baud_rate: 0
  
uart:
  rx_pin: GPIO16 # GPIO3
  tx_pin: GPIO17 # GPIO1
  baud_rate: 9600
  stop_bits: 2
  
modbus:
  send_wait_time: 200ms
  id: mod_bus_pzem

modbus_controller:
  - id: pzem
    ## the current device addr
    address: 0x01
    modbus_id: mod_bus_pzem
    command_throttle: 0ms
    setup_priority: -10
    update_interval: 30s
```

**ESPHome - PZEM017 + PZEM004Tv3:**

- nahrajeme configy pro oba wattmetry
- u PZEM004T jsme změnili adresu na 0x03 tak to musíme zohlednit v kódu nastavením adresy

```
logger:
   level: debug
   baud_rate: 0
  
uart:
  rx_pin: GPIO16 # GPIO3
  tx_pin: GPIO17 # GPIO1
  baud_rate: 9600
  stop_bits: 2
  
sensor:
  - platform: pzemdc # PZEM017
    current:
      name: "PZEM-003 Current"
    voltage:
      name: "PZEM-003 Voltage"
    power:
      name: "PZEM-003 Power"
    update_interval: 1s
    address: 0x01 # defaultní adresa
  - platform: pzemac # PZEM004Tv3
    current:
      name: "PZEM-004T V3 Current"
    voltage:
      name: "PZEM-004T V3 Voltage"
    energy:
      name: "PZEM-004T V3 Energy"
    power:
      name: "PZEM-004T V3 Power"
    update_interval: 1s
    address: 0x03 # nová adresa 
```


# ZDROJE:

[link](https://github.com/Gio-dot/PZEM-016-OLED-2-OUT-ESPHome/blob/master/wemos_d1_pzem016_display.yaml)
[link](https://github.com/arendst/Tasmota/issues/3694)
[link](https://hassiohelp.eu/2019/03/27/pzem-016/)
[link](https://github.com/Gio-dot/PZEM-016-OLED-2-OUT-ESPHome)
[link](https://esphome.io/components/sensor/pzemdc.html)

