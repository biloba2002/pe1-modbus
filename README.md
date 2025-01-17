# pe1-modbus

NOTE: This project is not an official HA or HACS integration!

This project offers a guideline how a Fröling PE1 / Lambdatronic Pellet boiler can be integrated into Home Assistant (HA - https://www.home-assistant.io) using the builtin Modbus interface.

This repo also includes python scripts that use a modbus module to read registers from PE1 an then post them via MQTT to use inside Home Assistant but this is not necessary since HA has it's own Modbus interface where we can directly map PE1 registers to HA entities.

At the time of writing the only drawback is that it seems to be impossible to combine those Modbus entites in HA into a Homeassitant device. The MQTT interface supports this so if you want that check the code in /src/pe1modbus.

## Hardware Requirements
The Lambdatronic board offers 2 serial interfaces, which can be configured to Modbus TCP in the settings.
To conveiently integrate the device into the local LAN network, the serial connection needs an adapter to RJ45 (RS232 to Ethernet Converter). This one by Waveshare (Industrial RS232/RS485 to Ethernet Converter) was used and is confiremd to work:
- https://www.waveshare.com/rs232-485-to-eth-for-eu.htm

The RS232 side can be connected to the COM2 port on the Lambdtronic. Ensure you use the proper cable that has the RX TX pins crossed.

More infromation on the hardware setup can be found in this guide:
https://loxwiki.atlassian.net/wiki/spaces/LOX/pages/1704984631/Fr+ling+Pelletskessel+RS232+an+Loxone+Modbus+TCP?focusedCommentId=1705050279


## Modbus TCP
After the hardware setup, there are many ways to test the modbus communication. QModMaster is a great tool for exploring Registers via a GUI - https://github.com/zhanglongqi/qModMaster. You could also write some short script in Python using the module pyModbusTCP (https://pypi.org/project/pyModbusTCP/), see the /src/ section in this repo for an example.

Information on Register addresses etc. can be studied in the ModBus Lambdatronic 3200 Modbus Definition Document (google search should provide a link to download). 

## Home Assitant Modbus

Home Assitant offers a interface to include entities using the Modbus protocol - https://www.home-assistant.io/integrations/modbus/

The cleanest way to do this would be to create a `modbus.yaml` in your HA home directory (next to the configuration.yaml) and include it into the configuration like so:

```yaml
modbus: !include modbus.yaml
```

This repo includes an example `modbus.yaml` file including a few entities that can be easily extended to suit your needs. 

Inside the `modbus.yaml` we first need to specify the `hub` which for this setup is the address of the Waveshare Converter.

The PE1 boiler usues a seperate Ethernet connection which is intended for the Fröling Connect service. Make sure to specify the Waveshare converter address in the yaml.

```yaml
- name: pe1_hub
  type: tcp
  host: xxx.xxx.xxx.xxx # address of the Waveshare converter
  port: 502
  sensors:
    ...
```

In the sensors section, you can spceify the register addresses you want to map to HA entities. For further information on how to address input registers holding registers etc. study the HA Modbus documentation linked above.

The PE1 system offers two input registers that specify the system status and the boiler status which are impelemented via an enum. See the Lambdatronic Modbus Definition PDF for more information on this. If you want to amp those status integer values to the corresponding status strings you can do this in HA via templating.

Example: So lets say we have declared one entity for the system status in our `modbus.yaml` like so: 

```yaml
- name: Modbus PE1 System Status Enum
  unique_id: modbus_pe1_system_status_enum
  slave: 2
  input_type: input
  address: 4000
  scan_interval: 30
  device_class: enum
```

We can then create a template entity in e.g. another yaml file used for template entities in the HA config directory `template.yaml` which is again included in the `configuration.yaml` like so:

```yaml
template: !include template.yaml
```

Inside this file we can map the enum values of our entity `modbus_pe1_system_status_enum` to string values that we can use to display in the frontend in some human readable format:

```yaml
- name: "Modbus PE1 System Status"
  unique_id: "modbus_pe1_system_status"
  state: >
    {% set mapper =  {      
      '0' : 'Continuous load',
      '1' : 'Domestic hot water',
      '2' : 'Automatic',
      '3' : 'Firewood operation',
      '4' : 'Cleaning',
      '5' : 'Boiler off',
      '6' : 'Extra heating',
      '7' : 'Chimney sweep',
      '8' : 'Cleaning' } %}
    {% set state =  states.sensor.modbus_pe1_system_status_enum.state %}
    {{ mapper[state] if state in mapper else 'Unknown' }}
  icon: >
    {% if this.state == 'Automatic' %}
      mdi:refresh-auto
    {% elif this.state == 'Domestic hot water' %}
      mdi:water-pump
    {% elif this.state == 'Continous load' %}
      mdi:hours-24
    {% else %}
      mdi:alert-circle
    {% endif %}
```
Here you could also change the icon of the entity based on it's state like shown above.

The same can be done for the boiler/furnace status. See the full `modbus.yaml` and `template.yaml` included in this repo for more info.


## TODO

The Lambdatronic Modbus Definition document describes how to use modbus to remote control the boiler heating mode. I have not yet managed to get that working. According to the document the heating mode register adrresses should reside at 48047 - 48064 in the holding registers. 
So theoretically we should be able to write those registers to change the heating mode.
There are 18 registers for that because we could have a max of 18 possible heating-circuits. Those registers should hold values from 0 to 5 for the six different modes: 

* 0 ... Aus
* 1 ... Automatik
* 2 ... Extraheizen
* 3 ... Absenken
* 4 ... Dauerabsenken
* 5 ... Partybetrieb

However when experimenting with those registeres I am unable to neither read the correct heating mode value nor write to it. Any help here would be much appreciated.
