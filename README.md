# ESPHome AC/DC wattmeter 

## Popis:

Následující postup je uplatněn pro měření DC (FVE) a AC (ze sítě) výkonu dodaného pro Bojler s AC/DC spirálou.  
Mezi FVE a bojlerem je regulátor bez měniče. Do bojelru příchází stejnosměrné napětí.  
Měření DC je realizováno pomocí PZEM-017 a AC PZEM-004Tv3.  

Bojler - Dražice LX ACDC/M+K 160l  
manual: [link](https://www.dzd-fv.cz/images/pdf/Navod_LX_ACDC_M_MKW_9_12_2020_CZ_6735552.pdf)
eshop: [link](https://www.solar-eshop.cz/p/fotovoltaicky-ohrivac-lx-acdc-m-k-abc-160/)

FV panely: 6ks Schutten Poly 250Wp  
MPPT regulátor Power Max:  

ESP32E board:  
![obrazek](https://user-images.githubusercontent.com/58307338/161389309-b47f301a-1040-422f-8a99-f0ece91003fe.png)

## PZEM-017 DC wattmeter - odstranění RS485:  

PZEM-017 používá pro komunikaci rozhraní RS485 a standardně bychom museli signál převést z RS485 přes převodník zpět na UART.  
To je pro nás tedy zbytečné a můžeme odstranit RS485 přímo z PZEM a použít přímo jeho UART.  
Odstraníme tedy RS485 rozhraní přímo ze základní desky a díky tomu můžeme a musíme pracovat s napětím 3V namísto původních 5V.  

** 1. odstraňte IO U5 a rezistor R19 (na U5 stačí odpájet pin1 a R19 na mém PZEM017 nebyl vůbec osazen) **

![obrazek](https://user-images.githubusercontent.com/58307338/161390331-cfa6a7f1-9662-453b-9f1b-0a01661cdbd1.png)

2. propojte optočlen U1 (pin4) s R19 (pin A) a  U2 (pin2) s R19 (pin B)  

![obrazek](https://user-images.githubusercontent.com/58307338/161390605-00ac177c-0d2f-46fa-aef7-136c7c0e2ff4.png)


Odteď bude tedy na svorce A signál TX a na svorce B RX.  
Na ESP připojte GPIO1 (TX) na svorku B (RX) a GPIO3 (RX) na svorku A (TX)  
Myslete na to že se již PZEM017 nenapájí 5V ale jen 3V z ESP - jinak nebude wattmetr funkční!!  

