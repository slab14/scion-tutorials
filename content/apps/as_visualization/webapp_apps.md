---
title: Visualization for Apps
parent: AS Visualization
grand_parent: Applications
nav_order: 20
---

# Webapp SCIONLab Apps Visualization

## Webapp Features

This Go web server wraps several SCION test client apps and provides an interface for any text and/or image output received. [SCIONLab Apps](http://github.com/netsec-ethz/scion-apps) are on Github.

Two functional server tests are included to test the networks without needing specific sensor or camera hardware, `imagetest` and `statstest`.

Supported client applications include `camerapp`, `sensorapp`, and `bwtester`. For best results, ensure the desired server-side apps are running and connected to the SCION network first. Instructions to setup the servers are [here](https://github.com/netsec-ethz/scion-apps). The web interface launched above can be used to run the client-side apps.

### bwtester

Simply adjust the dials to the desired level, while the icon lock will allow you to keep one value constant.

![Webapp Bandwidth Test](../images/webapp_bwtester.png?raw=true "Webapp Bandwidth Test")

### camerapp

The retrieved image will appear scaled down and can be clicked on to open a larger size.

![Webapp Image Test](../images/webapp_camerapp.png?raw=true "Webapp Image Test")

### sensorapp

![Webapp Stats Test](../images/webapp_sensorapp.png?raw=true "Webapp Stats Test")

## Related Links

* [Fetching sensor readings or time stamps](../apps/fetch_sensor_readings.md)
* [Fetching a camera image over the SCION network](../apps/access_camera.md)
* [Running the bandwidthtester application](../apps/bwtester.md)
