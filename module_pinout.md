| Pin name | Alt name | Pin function |
| --- | --- | --- |
| ADC | ADC | ADC with range of 0~1.875 V. |
| P7 | PIN7 | *Connected to UART1_RI (ring indicator)* |
| VIO | VDD_IO | Voltage of microcontroller IO |
| RXD | UART_RXD | Connected to AT command Rx (request) |
| TXD | UART_TXD | Connected to AT command Tx (response) |
| GPS | GPS_EN | GPIO3, which can probably be configured to output voltage if GPS turned on ??? |
| GND | GND | - |
| GND | GND | - |
| VIN | VIN | Input voltage: 4.7~15 V |
| --- | --- | --- |
| BAT | VBAT | Battery input voltage: 3.4~4.4 V. Will be charged if VIN is connected. |
| BAT | VBAT | *See above* |
| GND | GND | - |
| PWR | PWRKEY | System power control. Should not be grounded for long time (>12s). Pull low for >1s/1.2s to power on/off module. Pin does not need to be pulled up due to interal pull-up diode. Absolute max voltage 2.1V. |
| EN | EN | Connected to regulator (MP140) EN input. Pulled high by 100k resistor to VIN. Can be pulled low to disable regulator/ |
| GTX | GPS_TX | *Connected to GPS_TXD* |
| GRX | GPS_RX | *Connected to GPS_RXD* |
| IO | IO | Outputs high level when power on, low level when power off. Not sure if open-collector/emitter or not. |
| P5 | PIN5 | *Connected to UART1_DCD (carrier detects)* |

UART_RXD, UART_TXD, GPS_RX, and GPS_TX are level shifted from VDD_IO to VDD_EXT.
VDD_IO is broken out on the header, and VDD_EXT is provided by the SIM7080G module (1.8V).
VDD_IO should be held at the UART voltage of the microcontroller.

## AT Commands

- Require `AT` (case insensitive) prefix for all commands.
- Commands are terminated by `<CR>` (`\r`).
- Responses are usually given as `<CR><LF><response><CR><LF>`.

uPy example:

```python
from machine import UART
uart = UART(0, 9600)
uart.write(b'AT+GMI\r')
uart.read()
# expected response: b'AT+GMI\r\r\nSIMCOM_Ltd\r\n\r\nOK\r\n'
```

```python
uart.write(b'AT+CMEE=2\r')
uart.read()
uart.write(b'AT+CPIN?\r')
uart.read()
# AT+CPIN?\r\r\n+CME ERROR: SIM not inserted\r\n
```

```python
def do_stuff():
    while True:
        cmd = input("Enter command\n>>> ")
        if cmd:
            print(uart.write(cmd + '\r'))
        else:
            response = uart.read()
            if response:
                print(response.decode())
```

## Setup Instructions

```
AT # expected response: OK - try multiple times until started
ATE0 # only do if you want to disable echo
AT+CPIN? # check sim card - expected response: READY
AT+CGATT=1 # attach to GPRS - response: OK
AT+CSQ # check signal strength - expected response: non-99,anything
AT+CGDCONT=1,"IP","hologram" # set the APN to hologram
AT+COPS? # check network connection - expected response: 0,0,"Vodafone NZ",7
AT+CGNAPN # returns the APN ("hologram"), could be used to check that user configuration is correct
AT+CNCFG=0,1,"hologram",<username>,<password>,<auth_type> # could provide the PDP context optional parameters if needed, could default to AT+CGNAPN if not provided. (0, 1 are pdp context and IP type (0[both],1[ipv4],2[ipv6]) respectively). - expected response: OK
AT+CNACT=0,1 # enable context 0 - expected response: OK\r\n\r\n+APP PDP: 0,ACTIVE
AT+CNACT? # fetch contexts - expected response: +CNACT: 0,1,"ip.ad.dr.ess" [x4], first number = context, second num = enabled/disabled
# Test ping
AT+SNPDPID=0 # set ping context
AT+SNPING4="8.8.8.8",1,1,10000 # ping: expected response: not ERROR; +SNPING4: 1,8.8.8.8,time(int)
# Update time
AT+CNTP=nz.pool.ntp.org # expected response: OK
AT+CNTP # OK then wait for response: +CNTP: 1,"2022/07/17,05:27:36"
# MQTT
AT+SMCONF="URL","broker.hivemq.com",1883 # set broker and port
AT+SMCONF="CLIENTID","clientId-e51RWhZbTA" # set client ID
AT+SMCONF="TOPIC","among/us/jolon" # set topic
AT+SMCONF="MESSAGE","hello world from modem"
AT+SMCONN # connect
AT+SMSTATE? # should return: +SMSTATE:1
AT+SMPUB="my/topic",<msg_len>,0,0 # gives prompt starting with `>`
AT+SMSUB="my/topic",0 # will receive unsolicited messages: +SMSUB: "among/us/jolon","hello"
AT+SMUNSUB="my/topic" # unsubs from topic
AT+SMDISC # disconnect # receive: OK

# POWER
AT+CPOWD=1
```



```
AT+SMCONF="CLIENTID","id"
OK
AT+SMCONF="KEEPTIME",60
OK
AT+SMCONF="URL","test.mosquitto.org","1883"
OK
AT+SMCONF="CLEANSS",1
OK
AT+SMCONF="QOS",1
OK
AT+SMCONF="TOPIC","will topic"
OK
AT+SMCONF="MESSAGE","will message"
OK
AT+SMCONF="RETAIN",1
OK
```