# energy-cost-saving-calculator
Creates entities in Home Assistant to monitor energy cost saving for common tariff contract compared to spot price contract

**Needed entities in Home Assistant:**
- Total energy consumption sensor in kWh. Can be retrieved through e.g. [Home Assistant Glow](https://github.com/klaasnicolaas/home-assistant-glow)
- Current spot price (without VAT) in your energy market. Use instructions on [Nord Pool integration for Home Assistant](https://github.com/custom-components/nordpool)
- Sensor for current price of your common tariff contract [cents/kWh]. Use [input number helper](https://www.home-assistant.io/integrations/input_number/) in order to be able to modify value quickly without having to change configuration.yaml if price updates.
- Sensor for current Energy VAT-%. Use [input number helper](https://www.home-assistant.io/integrations/input_number/) and use % as unit of measurement.
- Sensor for spot energy margin [cents/kWh]. Again, use [input number helper](https://www.home-assistant.io/integrations/input_number/). This is in order to get a fair comparison between common tariff and spot price contract. Average values are around 0,4cents/kWh.
- [Utility meter](https://www.home-assistant.io/integrations/utility_meter/) hourly energy sensor needed for calculation, as following:
  
    ```
  utility_meter:
  hourly_energy:
    source: sensor.house_total_energy
    cycle: hourly
  ```
