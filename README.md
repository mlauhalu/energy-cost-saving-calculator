# energy-cost-saving-calculator
Creates entities in Home Assistant to monitor energy cost saving for common tariff contract compared to spot price contract

**Needed entities in Home Assistant:**
- ```sensor.house_total_energy``` - Total energy consumption sensor in kWh. Can be retrieved through e.g. [Home Assistant Glow](https://github.com/klaasnicolaas/home-assistant-glow)
- ```sensor.nordpool_kwh_fi_eur_2_10_0``` - Current spot price (without VAT) in your energy market. Use instructions on [Nord Pool integration for Home Assistant](https://github.com/custom-components/nordpool)
- ```input_number.energy_cost``` - Sensor for current cost of your common tariff contract (without VAT) [cents/kWh]. Use [input number helper](https://www.home-assistant.io/integrations/input_number/) in order to be able to modify value quickly without having to change configuration.yaml if price updates.
- ```input_number.energy_vat_rate``` - Sensor for current Energy VAT-%. Use [input number helper](https://www.home-assistant.io/integrations/input_number/) and use % as unit of measurement.
- ```input_number.energy_margin``` - Sensor for spot energy margin [cents/kWh]. Again, use [input number helper](https://www.home-assistant.io/integrations/input_number/). This is in order to get a fair comparison between common tariff and spot price contract. Average values are around 0,4cents/kWh.
- ```sensor.hourly_energy``` - [Utility meter](https://www.home-assistant.io/integrations/utility_meter/) hourly energy sensor needed for calculation, as following:
  
  ```
  utility_meter:
    hourly_energy:
      source: sensor.house_total_energy
      cycle: hourly
  ```
<br>

**[Template sensors](https://www.home-assistant.io/integrations/template/):**
- ```sensor.spot_price_incl_vat_and_margin``` - Sensor for total spot price incl. VAT and margin.

```
template:
  - sensor:
        - name: "spot_price_incl_vat_and_margin"
          unit_of_measurement: "cents"
          state: >
                {{ (( states('sensor.nordpool_kwh_fi_eur_2_10_0') | float ) * (1 + (states('input_number.energy_vat_rate') | float / 100)) + (states('input_number.energy_margin') | float )) | round(3) }}
```
- ```sensor.current_energy_price_incl_vat``` - Sensor for total common tariff price incl. VAT.
```
template:
  - sensor:
        - name: "current_energy_price_incl_vat"
          unit_of_measurement: "cents"
          state: >
                {{ ( states('input_number.energy_cost') | float ) * (1 + (states('input_number.energy_vat_rate') | float / 100)) | round(3) }}
```
- ```sensor.energy_cost_hourly_saving_incl_vat_and_margin``` - Sensor to calculate saving when common tariff price is below spot price.
```
template:
  - sensor:
        - name: "energy_cost_hourly_saving_incl_vat_and_margin"
          unit_of_measurement: "€"
          state: >
                {% if ( states('sensor.current_energy_price_incl_vat') | float ) < ( states('sensor.spot_price_incl_vat_and_margin') | float ) %}
                {{ (( states('sensor.hourly_energy') | float ) * (( states('sensor.spot_price_incl_vat_and_margin') | float ) - ( states('sensor.current_energy_price_incl_vat') | float )) / 100 ) | round(2) }}
                {% else %}
                0
                {% endif %}
```
- ```sensor.energy_cost_hourly_loss_incl_vat_and_margin``` - Sensor to calculate loss when common tariff price is above spot price.
```
template:
  - sensor:
        - name: "energy_cost_hourly_loss_incl_vat_and_margin"
          unit_of_measurement: "€"
          state: >
                {% if ( states('sensor.current_energy_price_incl_vat') | float ) > ( states('sensor.spot_price_incl_vat_and_margin') | float ) %}
                {{ (( states('sensor.hourly_energy') | float ) * (( states('sensor.current_energy_price_incl_vat') | float ) - ( states('sensor.spot_price_incl_vat_and_margin') | float )) / 100 ) | round(2) }}
                {% else %}
                0
                {% endif %}
```
<br>

**[Utility meters](https://www.home-assistant.io/integrations/utility_meter/) to calculate savings/losses for different periods:**
```
utility_meter:
  energy_cost_daily_saving_incl_vat_and_margin:
    source: sensor.energy_cost_hourly_saving_incl_vat_and_margin
    cycle: daily
  energy_cost_monthly_saving_incl_vat_and_margin:
    source: sensor.energy_cost_hourly_saving_incl_vat_and_margin
    cycle: monthly
  energy_cost_quarterly_saving_incl_vat_and_margin:
    source: sensor.energy_cost_hourly_saving_incl_vat_and_margin
    cycle: quarterly
  energy_cost_yearly_saving_incl_vat_and_margin:
    source: sensor.energy_cost_hourly_saving_incl_vat_and_margin
    cycle: yearly

  energy_cost_daily_loss_incl_vat_and_margin:
    source: sensor.energy_cost_hourly_loss_incl_vat_and_margin
    cycle: daily
  energy_cost_monthly_loss_incl_vat_and_margin:
    source: sensor.energy_cost_hourly_loss_incl_vat_and_margin
    cycle: monthly
  energy_cost_quarterly_loss_incl_vat_and_margin:
    source: sensor.energy_cost_hourly_loss_incl_vat_and_margin
    cycle: quarterly
  energy_cost_yearly_loss_incl_vat_and_margin:
    source: sensor.energy_cost_hourly_loss_incl_vat_and_margin
    cycle: yearly
```
<br>

**[Template sensors](https://www.home-assistant.io/integrations/template/) to calculate profit (delta between saving and loss) for different periods:**
```
template:
  - sensor:
        - name: "energy_cost_today_profit_incl_vat_and_margin"
          unit_of_measurement: "€"
          state: >
                {{ (( states('sensor.energy_cost_daily_saving_incl_vat_and_margin') | float ) - ( states('sensor.energy_cost_daily_loss_incl_vat_and_margin') | float )) | round(2) }}

        - name: "energy_cost_yesterday_profit_incl_vat_and_margin"
          unit_of_measurement: "€"
          state: >
                {{ (( state_attr('sensor.energy_cost_daily_saving_incl_vat_and_margin', 'last_period') | float ) - ( state_attr('sensor.energy_cost_daily_loss_incl_vat_and_margin', 'last_period') | float )) | round(2) }}

        - name: "energy_cost_this_month_profit_incl_vat_and_margin"
          unit_of_measurement: "€"
          state: >
                {{ (( states('sensor.energy_cost_monthly_saving_incl_vat_and_margin') | float ) - ( states('sensor.energy_cost_monthly_loss_incl_vat_and_margin') | float )) | round(2) }}

        - name: "energy_cost_this_year_profit_incl_vat_and_margin"
          unit_of_measurement: "€"
          state: >
                {{ (( states('sensor.energy_cost_yearly_saving_incl_vat_and_margin') | float ) - ( states('sensor.energy_cost_yearly_loss_incl_vat_and_margin') | float )) | round(2) }}
```
<br>

**Charting to illustrate saving/losses on daily basis using [ApexCharts](https://github.com/RomRider/apexcharts-card):**

- Additional template sensors:
```
template:
  - sensor:
        - name: "energy_cost_diff_cents_incl_vat_and_margin"
          unit_of_measurement: "cents"
          state: >
                {{ (( states('sensor.spot_price_incl_vat_and_margin') | float ) - ( states('sensor.current_energy_price_incl_vat') | float )) | round(2) }}
        - name: "energy_cost_hourly_saving_incl_vat_and_margin_chart"
          unit_of_measurement: "€"
          state: >
                {{ (( states('sensor.hourly_energy') | float ) * (( states('sensor.spot_price_incl_vat_and_margin') | float ) - ( states('sensor.current_energy_price_incl_vat') | float )) / 100 ) | round(2) }}
```
- Create an ApexChart with the following configuration:
```
type: custom:apexcharts-card
graph_span: 24h
update_interval: 10min
update_delay: 5000ms
yaxis:
  - id: left
    decimals: 2
    apex_config:
      title:
        text: €
  - id: right
    decimals: 2
    opposite: true
    apex_config:
      title:
        text: cents
show:
  last_updated: true
header:
  show: true
  title: Hourly energy saving (incl VAT and margin)
span:
  end: hour
series:
  - entity: sensor.energy_cost_hourly_saving_incl_vat_and_margin_chart
    type: column
    yaxis_id: left
    float_precision: 2
    name: Energy cost saving [€]
    show:
      legend_value: false
      extremas: true
    group_by:
      func: last
      duration: 1h
  - entity: sensor.energy_cost_diff_cents_incl_vat_and_margin
    type: line
    yaxis_id: right
    float_precision: 2
    name: Price delta [cents]
    show:
      legend_value: false
      extremas: false
    group_by:
      func: last
```



