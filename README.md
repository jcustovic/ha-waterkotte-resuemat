
Integration to read values from Waterkotte Resümat CD4 (Version 8126) and have them available in Home Assistant (HA).

I am using this on Waterkotte AI1 5006.4 and it is connected via USB-Serial-Port Converter.

The logic is implemented via Node-Red so you will need to install the following modules on your HA installation:
* node-red-contrib-home-assistant-websocket
* node-red-contrib-buffer-parser
* node-red-node-serialport
* node-red-show-value


The Node-Red flow reads all the values every 2 min (but you can adjust) and you can control the following:
* Heating on/off (02.00 Handabschaltung)
* Cooling on/off (03.00 Abschaltung)
* Hot watter on/off (04.00 Abschaltung)
* Heating temperature (02.02 Hzg Ruecklauf Sollwert am Einsatzp.)
* Cooling temperature (03.02 Ruecklauftemp. Sollwert am Einsatzzeitp.)
* Hot watter temperature (04.04 Warmwasser Soll)
* Synchronize time and date

How it looks like:

Node-Red:

<img width="2576" height="1158" alt="image" src="https://github.com/user-attachments/assets/e60b6fa0-83c5-4555-8a8c-e245453ca4e4" />

Home Assistant:

<img width="454" height="1150" alt="image" src="https://github.com/user-attachments/assets/dfe4b228-b4cf-4e25-abcf-7a442aedcf9e" />
<img width="451" height="384" alt="image" src="https://github.com/user-attachments/assets/a5154e80-98a4-440b-be5b-3f688fd51c08" />

--

NOTE: You are using this at your own risk.
