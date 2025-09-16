
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

How it looks like:

NOTE: You are using this at your own risk.
