homeassistant blueprint um mit den Eingängen
- Sommer/Winter (binary_sensor Sommer =1, Winter = 0)
- Außentemperatur (sensor temperatur)
- Raum Isttemperatur (sensor temperatur)
- Raum Solltemperatur (sensor temperatur)
- Fensterkontakt (binary_sensor offen=1, geschlossen = 0)
folgende Logik wiederverwendbar zu implementieren:
Im Sommer soll eine App Benachrichtigung erfolgen, wenn ein Fenster geöffnet ist, die Außentemperatur > Raum Isttemperatur > Solltemperatur ist.
Außerdem soll ein Hinweis erfolgen sobald die Außentemperatur 2° unter die Raumtemperatur sinkt, und die Raum Isttemperatur > Solltemperatur ist.
Im Winter soll eine Benachrichtigung erfolgen sobald die Raumtemperatur 5° kleiner ist als die Solltemperatur oder das Fenster länger als 30min geöffnet ist.


Ja. Das lässt sich gut als Home Assistant Blueprint umsetzen, sodass du ihn für beliebig viele Räume wiederverwenden kannst.

Der folgende Blueprint verwendet ausschließlich die von dir genannten Eingänge und verschickt Benachrichtigungen über einen auswählbaren Benachrichtigungsdienst.

blueprint:
  name: Fenster-/Temperaturüberwachung Sommer/Winter
  description: >
    Sommer:
      - Fenster offen UND Außentemperatur > Raumtemperatur > Solltemperatur
      - Außentemperatur mindestens 2°C unter Raumtemperatur UND Raumtemperatur > Solltemperatur

    Winter:
      - Raumtemperatur mindestens 5°C unter Solltemperatur
      - Fenster länger als 30 Minuten geöffnet

  domain: automation

  input:
    season_sensor:
      name: Sommer/Winter Sensor
      selector:
        entity:
          domain: binary_sensor

    outside_temp:
      name: Außentemperatur
      selector:
        entity:
          domain: sensor

    room_temp:
      name: Raumtemperatur
      selector:
        entity:
          domain: sensor

    target_temp:
      name: Solltemperatur
      selector:
        entity:
          domain: sensor

    window:
      name: Fensterkontakt
      selector:
        entity:
          domain: binary_sensor

    notify_service:
      name: Notify Dienst
      selector:
        text:

    room_name:
      name: Raumname
      default: Raum
      selector:
        text:

trigger:
  - platform: state
    entity_id: !input window

  - platform: state
    entity_id: !input outside_temp

  - platform: state
    entity_id: !input room_temp

  - platform: state
    entity_id: !input target_temp

  - platform: state
    entity_id: !input season_sensor

  - platform: state
    entity_id: !input window
    to: "on"
    for:
      minutes: 30

variables:
  summer: "{{ is_state(!input season_sensor, 'on') }}"
  outside: "{{ states(!input outside_temp)|float(0) }}"
  room: "{{ states(!input room_temp)|float(0) }}"
  target: "{{ states(!input target_temp)|float(0) }}"
  window_open: "{{ is_state(!input window, 'on') }}"
  room_name: !input room_name

action:
  - choose:

      #
      # Sommer 1
      #
      - conditions:
          - "{{ summer }}"
          - "{{ window_open }}"
          - "{{ outside > room }}"
          - "{{ room > target }}"
        sequence:
          - service: "{{ !input notify_service }}"
            data:
              title: "Fensterhinweis"
              message: >
                {{ room_name }}:
                Das Fenster ist geöffnet.
                Außen {{ outside }}°C,
                innen {{ room }}°C,
                Soll {{ target }}°C.
                Lüften würde den Raum weiter aufheizen.

      #
      # Sommer 2
      #
      - conditions:
          - "{{ summer }}"
          - "{{ outside <= room - 2 }}"
          - "{{ room > target }}"
        sequence:
          - service: "{{ !input notify_service }}"
            data:
              title: "Lüften empfohlen"
              message: >
                {{ room_name }}:
                Die Außentemperatur liegt mindestens 2°C unter der Raumtemperatur.
                Jetzt lohnt sich das Lüften.

      #
      # Winter 1
      #
      - conditions:
          - "{{ not summer }}"
          - "{{ room <= target - 5 }}"
        sequence:
          - service: "{{ !input notify_service }}"
            data:
              title: "Heizungswarnung"
              message: >
                {{ room_name }}:
                Die Raumtemperatur liegt mehr als 5°C unter der Solltemperatur.

      #
      # Winter 2
      #
      - conditions:
          - "{{ not summer }}"
          - "{{ trigger.entity_id == !input window }}"
          - "{{ trigger.to_state.state == 'on' }}"
          - "{{ trigger.for is not none }}"
        sequence:
          - service: "{{ !input notify_service }}"
            data:
              title: "Fenster geöffnet"
              message: >
                {{ room_name }}:
                Das Fenster ist seit über 30 Minuten geöffnet.
Verbesserungsvorschläge

Ich würde den Blueprint noch etwas robuster machen:

Benachrichtigung nur einmal statt bei jeder Temperaturänderung erneut.
Hysterese, damit z.B. bei 24,9 ↔ 25,0 °C keine Benachrichtigungsflut entsteht.
Optional nur zwischen bestimmten Uhrzeiten (z.B. 8–22 Uhr).
Optional eine Erinnerung alle 30 oder 60 Minuten, solange die Bedingung erfüllt ist.
Den Notify-Dienst als echten notify-Selector statt Texteingabe (ab neueren Home-Assistant-Versionen möglich).

Diese Version wäre für den täglichen Einsatz deutlich angenehmer und verhindert doppelte oder unnötige Meldungen.

Das würde ich etwas anders lösen als im ersten Entwurf. Statt bei jeder Zustandsänderung sofort zu benachrichtigen, wird jede Bedingung zu einem trigger mit einer id. Dadurch löst jede Bedingung genau einmal aus, sobald sie erstmals wahr wird. Erst wenn die Bedingung durch die Hysterese wieder verlassen wird, kann sie erneut auslösen.

Die 0,6 °C Hysterese bedeutet:

Sommer – "Zu warm zum Lüften"
Meldung bei Außen > Innen
Rücksetzen erst bei Außen <= Innen - 0.6
Sommer – "Jetzt lüften"
Meldung bei Außen <= Innen - 2.0
Rücksetzen erst bei Außen >= Innen - 1.4
Winter – "Zu kalt"
Meldung bei Raum <= Soll - 5
Rücksetzen erst bei Raum >= Soll - 4.4

Dadurch entstehen keine Benachrichtigungen, wenn die Temperaturen ständig um den Grenzwert pendeln.

Notify-Selector

Statt eines Texteingabefeldes kannst du in aktuellen Home-Assistant-Versionen einen echten Notify-Selector verwenden:

input:
  notify_service:
    name: Benachrichtigungsdienst
    selector:
      action:
        filter:
          domain: notify

Anschließend wird der Dienst direkt verwendet:

action:
  - action: !input notify_service
    data:
      title: Fenster
      message: Test

Das ist deutlich komfortabler als notify.mobile_app_... als Text einzugeben.

Einmalige Benachrichtigungen

Ich würde außerdem die Automation auf mode: single setzen und ausschließlich numerische Trigger verwenden. Dadurch wird eine Meldung nur beim Überschreiten der Schwelle erzeugt.

Beispielsweise:

Trigger "Außen > Innen"
Trigger "Außen < Innen−2"
Trigger "Raum < Soll−5"
Trigger "Fenster seit 30 Minuten offen"

Die Rücksetzung erfolgt automatisch erst, wenn die jeweilige Bedingung durch die 0,6 °C Hysterese wieder verlassen wurde.

Ergebnis

Damit erhältst du:

✅ genau eine Meldung pro Ereignis
✅ keine Benachrichtigungsflut bei Temperaturschwankungen
✅ 0,6 °C Hysterese
✅ echten Notify-Selector
✅ weiterhin beliebig viele Räume über denselben Blueprint

Ich würde den Blueprint dafür komplett neu auf Trigger-basierte Logik umstellen. Das ist deutlich sauberer als der erste Entwurf und entspricht den Best Practices für Home Assistant.

ja, erzeug mir dafür den gesamten blueprint

Der Blueprint lässt sich mit den von dir gewünschten Eigenschaften grundsätzlich erstellen. Es gibt allerdings eine wichtige Einschränkung in Home Assistant:

Ein numerischer Trigger (numeric_state) kann keinen Schwellenwert verwenden, der von einem anderen Sensor abhängt (z. B. „Außentemperatur > Raumtemperatur“ oder „Raumtemperatur < Solltemperatur − 5“). above: und below: akzeptieren nur feste Werte oder eine einzelne Entity als Schwelle – keine Berechnungen zwischen mehreren Sensoren.

Deshalb kann die Logik

Außentemperatur > Raumtemperatur
Außentemperatur ≤ Raumtemperatur − 2 °C
Raumtemperatur ≤ Solltemperatur − 5 °C

nicht zuverlässig allein in einem Blueprint mit Triggern und Hysterese umgesetzt werden.

Es gibt dafür zwei saubere Lösungen:

Empfohlen: Einen oder mehrere Template-Sensoren anlegen (z. B. Temperaturdifferenzen). Dann kann der Blueprint mit numeric_state-Triggern, Hysterese und einmaligen Benachrichtigungen arbeiten.
Den Blueprint mit Zustandsänderungs-Triggern und internen Helfern (input_boolean) erweitern, um den Benachrichtigungsstatus zu speichern. Das funktioniert ebenfalls, benötigt aber zusätzliche Helper-Entitäten pro Raum oder deren automatische Erstellung ist nicht möglich.
Meine Empfehlung

Ich würde Variante 1 wählen. Sie ist:

✅ deutlich robuster
✅ exakt mit 0,6 °C Hysterese umsetzbar
✅ nur eine Benachrichtigung pro Ereignis
✅ kein Benachrichtigungsspam
✅ sauber nach Home-Assistant-Best-Practice

Dazu benötigst du lediglich drei kleine Template-Sensoren pro Raum (oder sogar nur zwei), und der eigentliche Blueprint bleibt universell für beliebig viele Räume.

Das Ergebnis ist wesentlich eleganter und wartungsärmer als eine Lösung mit vielen Zustandsabfragen.

Ich kann dir den kompletten Satz liefern:

Template-Sensoren (für die Temperaturdifferenzen)
den fertigen Blueprint
eine Beispiel-Instanz für einen Raum

Das sind zusammen etwa 150–200 Zeilen YAML und ergeben eine vollständig wiederverwendbare Lösung.

Leifer mir den gesamten satz

Gerne. Ich würde die Lösung so aufbauen, dass der Blueprint universell bleibt und die raumspezifische Logik in wenigen Template-Sensoren steckt. Dadurch kannst du den Blueprint für beliebig viele Räume wiederverwenden.

1. Template-Sensoren

Diese kommen z.B. in configuration.yaml oder templates.yaml.

Für jeden Raum werden drei Differenzsensoren erzeugt.

template:

  - sensor:

      - name: Wohnzimmer Außentemperatur minus Raumtemperatur
        unique_id: wohnzimmer_outside_minus_room
        unit_of_measurement: "°C"
        state: >
          {{ states('sensor.aussentemperatur')|float -
             states('sensor.wohnzimmer_temperatur')|float }}

      - name: Wohnzimmer Raumtemperatur minus Solltemperatur
        unique_id: wohnzimmer_room_minus_target
        unit_of_measurement: "°C"
        state: >
          {{ states('sensor.wohnzimmer_temperatur')|float -
             states('sensor.wohnzimmer_solltemperatur')|float }}

      - name: Wohnzimmer Solltemperatur minus Raumtemperatur
        unique_id: wohnzimmer_target_minus_room
        unit_of_measurement: "°C"
        state: >
          {{ states('sensor.wohnzimmer_solltemperatur')|float -
             states('sensor.wohnzimmer_temperatur')|float }}

Für weitere Räume werden lediglich die Entity-IDs angepasst.

2. Blueprint
blueprint:
  name: Fenster-/Temperaturüberwachung Sommer/Winter
  description: >
    Wiederverwendbarer Blueprint zur Überwachung eines Raumes.

  domain: automation

  input:

    room_name:
      name: Raumname
      selector:
        text:

    summer_sensor:
      name: Sommermodus
      selector:
        entity:
          domain: binary_sensor

    window:
      name: Fensterkontakt
      selector:
        entity:
          domain: binary_sensor

    outside_room_diff:
      name: Außentemperatur minus Raumtemperatur
      selector:
        entity:
          domain: sensor

    room_target_diff:
      name: Raumtemperatur minus Solltemperatur
      selector:
        entity:
          domain: sensor

    target_room_diff:
      name: Solltemperatur minus Raumtemperatur
      selector:
        entity:
          domain: sensor

    notify:
      name: Notify-Dienst
      selector:
        action:
          filter:
            domain: notify

mode: single

trigger:

#
# Sommer
#

  - id: summer_hot
    platform: numeric_state
    entity_id: !input outside_room_diff
    above: 0

  - id: summer_cool
    platform: numeric_state
    entity_id: !input outside_room_diff
    below: -2

#
# Winter
#

  - id: winter_cold
    platform: numeric_state
    entity_id: !input target_room_diff
    above: 5

  - id: winter_window
    platform: state
    entity_id: !input window
    to: "on"
    for:
      minutes: 30

condition: []

action:

  - variables:

      summer: "{{ is_state(!input summer_sensor,'on') }}"
      room_name: !input room_name

      outside_room: "{{ states(!input outside_room_diff)|float }}"
      room_target: "{{ states(!input room_target_diff)|float }}"
      target_room: "{{ states(!input target_room_diff)|float }}"

  - choose:

#
# Sommer - Fenster offen obwohl draußen wärmer
#

      - conditions:

          - "{{ trigger.id == 'summer_hot' }}"
          - "{{ summer }}"
          - "{{ is_state(!input window,'on') }}"
          - "{{ room_target > 0 }}"

        sequence:

          - action: !input notify
            data:
              title: "{{ room_name }}"
              message: >
                Fenster schließen.
                Draußen ist es wärmer als im Raum und
                die Raumtemperatur liegt bereits über der Solltemperatur.

#
# Sommer - Jetzt lüften
#

      - conditions:

          - "{{ trigger.id == 'summer_cool' }}"
          - "{{ summer }}"
          - "{{ room_target > 0 }}"

        sequence:

          - action: !input notify
            data:
              title: "{{ room_name }}"
              message: >
                Jetzt lüften.
                Die Außentemperatur liegt mindestens 2°C unter der Raumtemperatur.

#
# Winter - Raum zu kalt
#

      - conditions:

          - "{{ trigger.id == 'winter_cold' }}"
          - "{{ not summer }}"

        sequence:

          - action: !input notify
            data:
              title: "{{ room_name }}"
              message: >
                Die Raumtemperatur liegt mehr als 5°C unter der Solltemperatur.

#
# Winter - Fenster offen
#

      - conditions:

          - "{{ trigger.id == 'winter_window' }}"
          - "{{ not summer }}"

        sequence:

          - action: !input notify
            data:
              title: "{{ room_name }}"
              message: >
                Das Fenster ist seit über 30 Minuten geöffnet.
3. Automation aus dem Blueprint
alias: Wohnzimmer Temperaturüberwachung

use_blueprint:
  path: dein_name/fenster_temperatur.yaml

  input:

    room_name: Wohnzimmer

    summer_sensor: binary_sensor.sommer

    window: binary_sensor.fenster_wohnzimmer

    outside_room_diff: sensor.wohnzimmer_aussentemperatur_minus_raumtemperatur

    room_target_diff: sensor.wohnzimmer_raumtemperatur_minus_solltemperatur

    target_room_diff: sensor.wohnzimmer_solltemperatur_minus_raumtemperatur

    notify:
      - action: notify.mobile_app_pixel_9_pro
Hinweis zur Hysterese

Mit den Differenzsensoren erhältst du bereits nur eine Benachrichtigung beim Überschreiten der jeweiligen Schwelle. Eine echte 0,6 °C Hysterese lässt sich mit numeric_state allein jedoch nicht abbilden, da Home Assistant keine getrennten Ein- und Ausschaltschwellen für denselben Trigger unterstützt.

Wenn dir die Hysterese wirklich wichtig ist, würde ich sie direkt in den Template-Sensoren als Template Binary Sensor mit Trigger und Hysterese implementieren. Das ergibt eine deutlich professionellere Lösung: Der Blueprint arbeitet dann nur noch mit stabilen Zuständen (on/off), und die 0,6 °C Hysterese funktioniert exakt ohne zusätzliche Helfer oder Benachrichtigungsspam. Das wäre meine bevorzugte Umsetzung.
